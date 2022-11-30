---
title: "Golang Libaray Sync.Map Source Code Analysis"
date: 2022-11-30T19:47:37+08:00
draft: true
author: "whiledoing"
tags: [golang]
categories: [programming]
---

源码分析golang的线程安全map实现: [sync.Map](https://pkg.go.dev/sync#Map)

<!--more-->

## intro

go中官方的线程安全map实现，其设计的思路是用空间换时间。保存了两个map：

- read：无锁的map，只会增加数据，不会减少数据（不需要考虑re-hash）
- dirty：加锁的map，可以写入，删除数据，可以进行re-hash

所以，`sync.Map`的主要实现思路就是：尽量把高频的操作放在read map中执行，低频的操作放在dirty map中执行。

## impl

对应的数据结构定义如下：

```go
type Map struct {
    mu Mutex

    // read contains the portion of the map's contents that are safe for
    // concurrent access (with or without mu held).
    //
    // The read field itself is always safe to load, but must only be stored with
    // mu held.
    //
    // Entries stored in read may be updated concurrently without mu, but updating
    // a previously-expunged entry requires that the entry be copied to the dirty
    // map and unexpunged with mu held.
    read atomic.Value // readOnly

    // dirty contains the portion of the map's contents that require mu to be
    // held. To ensure that the dirty map can be promoted to the read map quickly,
    // it also includes all of the non-expunged entries in the read map.
    //
    // Expunged entries are not stored in the dirty map. An expunged entry in the
    // clean map must be unexpunged and added to the dirty map before a new value
    // can be stored to it.
    //
    // If the dirty map is nil, the next write to the map will initialize it by
    // making a shallow copy of the clean map, omitting stale entries.
    dirty map[interface{}]*entry

    // misses counts the number of loads since the read map was last updated that
    // needed to lock mu to determine whether the key was present.
    //
    // Once enough misses have occurred to cover the cost of copying the dirty
    // map, the dirty map will be promoted to the read map (in the unamended
    // state) and the next store to the map will make a new dirty copy.
    misses int
}

// readOnly is an immutable struct stored atomically in the Map.read field.
type readOnly struct {
    m       map[interface{}]*entry
    amended bool // true if the dirty map contains some key not in m.
}

type entry struct {
    p unsafe.Pointer // *interface{}
}
```

- `read`是一个原子类型，因为该结构会被原子的变更（比如，将dirty内容替换到read中，需要重新生成一个read结构）
- `dirty`就是一个原生的map类型，需要加锁保护
- `entry`保存实际的数据，内部保存的是一个指针，dirty和read都记录相同内容的entry(修改其中内部的指针，就会修改对应的数值)

store一个数据的流程：

- 如果read中有数据，通过**CAS**修改
- 如果read中有数据，但是为`expunged`状态，说明read中该数据之前被删除过，但是没有同步到dirty中，需要在dirty中创建一个entry（这样子，dirty升级为read的时候，两边的entry内容一致）
- 如果read中没有数据，dirty中有数据，更新dirty中的entry中指针
- 如果都没有，且dirty为空，就原子创建一个dirty，且将read中的非空内容，拷贝到dirty中（空的内容会标记为expunged，如果以后都没有访问，dirty中就没有该记录，如果后续访问了，会在dirty中再新建一个entry）
- 如果都没有，且dirty有数据，就在dirty中建立一个entry条目

```go
// Store sets the value for a key.
func (m *Map) Store(key, value interface{}) {
    read, _ := m.read.Load().(readOnly)
    // 基于CAS修改
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }

    // 否则需要加锁
    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok {
        // read中有，但是为expunged状态，所以之前删除过，但是没有同步到dirty中，需要在dirty中记录一个entry
        if e.unexpungeLocked() {
            // The entry was previously expunged, which implies that there is a
            // non-nil dirty map and this entry is not in it.
            m.dirty[key] = e
        }
        e.storeLocked(&value)
    } else if e, ok := m.dirty[key]; ok {
        // 直接更新entry中的数值
        e.storeLocked(&value)
    } else {
        // diry中没有数据，创建dirty
        if !read.amended {
            // We're adding the first new key to the dirty map.
            // Make sure it is allocated and mark the read-only map as incomplete.
            m.dirtyLocked()
            // 原子的更新当前read，记录amended为true，表示已经存在dirty了
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        m.dirty[key] = newEntry(value)
    }
    m.mu.Unlock()
}
```

对应的，Load数据的时候，如果发现多次miss（不在read）中，说明有一些高频的数据在dirty中，需要将dirty升级为read。

升级非常简单，就是把dirty的内容直接放到read中（都是map，赋值即可）

```go
// Load returns the value stored in the map for a key, or nil if no
// value is present.
// The ok result indicates whether value was found in the map.
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]

    // read.amended 表示是否在dirty中存在数据，dirty升级到read之后，amended默认就是false
    // 一旦dirty中写入数据，amended就被设置为true
    if !ok && read.amended {
        m.mu.Lock()
        // Avoid reporting a spurious miss if m.dirty got promoted while we were
        // blocked on m.mu. (If further loads of the same key will not miss, it's
        // not worth copying the dirty map for this key.)
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
            e, ok = m.dirty[key]
            // Regardless of whether the entry was present, record a miss: this key
            // will take the slow path until the dirty map is promoted to the read
            // map.
            m.missLocked()
        }
        m.mu.Unlock()
    }
    if !ok {
        return nil, false
    }
    return e.load()
}

func (m *Map) missLocked() {
    m.misses++
    if m.misses < len(m.dirty) {
        return
    }
    m.read.Store(readOnly{m: m.dirty})
    m.dirty = nil
    m.misses = 0
}
```

## wrap up

`sync.Map`非常适合**读多写少**的场景，如果写的多，会导致read经常拷贝到dirty中，而如果多次读，很少加载的话，该结构非常的快，因为所有的read操作都不需要加锁。

其核心思路是：

- 空间换时间
- 分级考虑，read不加锁，提供只读，单调增的服务。write加锁，可以删除数据。
- 频次考虑，访问少的数据放在dirty中，如果dirty miss多了，就升级到read中。
- read中删除数据，只是标记为空的entry。
