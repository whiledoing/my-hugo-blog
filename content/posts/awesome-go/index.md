---
title: "Awesome Golang"
date: 2022-11-29T14:54:29+08:00
author: "whiledoing"
tags: ["golang"]
categories: ["software"]

resources:
- name: featured-image
  src: logo.png
- name: featured-image-preview
  src: logo.png
---

记录使用学习[awesome-go](https://go.libhunt.com/)相关内容.

<!--more-->

## command lines

### corbra

[cobra](https://cobra.dev/) is a framework for modern cli apps.

cobra支持sub-commands, 有利于实现复杂的cli工具.

实现上, sub-commands构成一个树状结构, 在运行时, 会递归找到实际执行的子命令.

```go
// https://github.com/spf13/cobra/blob/6b0bd3076cfafd1c108264ed1e4aa0c0fe3f8537/command.go#L699
// Find the target command given the args and command tree
// Meant to be run on the highest node. Only searches down.
func (c *Command) Find(args []string) (*Command, []string, error) {
  var innerfind func(*Command, []string) (*Command, []string)

  innerfind = func(c *Command, innerArgs []string) (*Command, []string) {
    argsWOflags := stripFlags(innerArgs, c)
    if len(argsWOflags) == 0 {
      return c, innerArgs
    }
    nextSubCmd := argsWOflags[0]

    cmd := c.findNext(nextSubCmd)
    if cmd != nil {
      // 关键是这里, 递归同步向下查找, 并且去掉当前的前缀命令(nextSubCmd)
      return innerfind(cmd, c.argsMinusFirstX(innerArgs, nextSubCmd))
    }
    return c, innerArgs
  }

  commandFound, a := innerfind(c, args)
  if commandFound.Args == nil {
    return commandFound, a, legacyArgs(commandFound, stripFlags(a, commandFound))
  }
  return commandFound, a, nil
}
```

其中关键的算法, 是去掉命令行中的flags参数, 有别于flag库做的解析, `stripFlags`主要是去掉flag参数,
而重点提取第一个实际执行命令.

```go
// https://github.com/spf13/cobra/blob/6b0bd3076cfafd1c108264ed1e4aa0c0fe3f8537/command.go#L618
func stripFlags(args []string, c *Command) []string {
	if len(args) == 0 {
		return args
	}
	c.mergePersistentFlags()

	commands := []string{}
	flags := c.Flags()

Loop:
	for len(args) > 0 {
		s := args[0]
		args = args[1:]
		switch {
		case s == "--":
			// "--" terminates the flags
			break Loop
		case strings.HasPrefix(s, "--") && !strings.Contains(s, "=") && !hasNoOptDefVal(s[2:], flags):
			// If '--flag arg' then
			// delete arg from args.
			fallthrough // (do the same as below)
		case strings.HasPrefix(s, "-") && !strings.Contains(s, "=") && len(s) == 2 && !shortHasNoOptDefVal(s[1:], flags):
			// If '-f arg' then
			// delete 'arg' from args or break the loop if len(args) <= 1.
			if len(args) <= 1 {
				break Loop
			} else {
				args = args[1:]
				continue
			}
		case s != "" && !strings.HasPrefix(s, "-"):
			commands = append(commands, s)
		}
	}

	return commands
}
```

代码中, `fallthrough`用来直接漏到下一个case执行, 且不管case语句的bool测试结果. 在上述代码中, 目的是去掉类似`--flag arg`中的`arg`数据.

---

cobra中的help是自动生成的, 核心基于go-template生成**复杂结构化**信息, 这种方式值得学习:

```go
// https://github.com/spf13/cobra/blob/6b0bd3076cfafd1c108264ed1e4aa0c0fe3f8537/command.go#L531
// UsageTemplate returns usage template for the command.
func (c *Command) UsageTemplate() string {
	if c.usageTemplate != "" {
		return c.usageTemplate
	}

	if c.HasParent() {
		return c.parent.UsageTemplate()
	}
	return `Usage:{{if .Runnable}}
  {{.UseLine}}{{end}}{{if .HasAvailableSubCommands}}
  {{.CommandPath}} [command]{{end}}{{if gt (len .Aliases) 0}}

Aliases:
  {{.NameAndAliases}}{{end}}{{if .HasExample}}

Examples:
{{.Example}}{{end}}{{if .HasAvailableSubCommands}}{{$cmds := .Commands}}{{if eq (len .Groups) 0}}

Available Commands:{{range $cmds}}{{if (or .IsAvailableCommand (eq .Name "help"))}}
  {{rpad .Name .NamePadding }} {{.Short}}{{end}}{{end}}{{else}}{{range $group := .Groups}}

{{.Title}}{{range $cmds}}{{if (and (eq .GroupID $group.ID) (or .IsAvailableCommand (eq .Name "help")))}}
  {{rpad .Name .NamePadding }} {{.Short}}{{end}}{{end}}{{end}}{{if not .AllChildCommandsHaveGroup}}

Additional Commands:{{range $cmds}}{{if (and (eq .GroupID "") (or .IsAvailableCommand (eq .Name "help")))}}
  {{rpad .Name .NamePadding }} {{.Short}}{{end}}{{end}}{{end}}{{end}}{{end}}{{if .HasAvailableLocalFlags}}

Flags:
{{.LocalFlags.FlagUsages | trimTrailingWhitespaces}}{{end}}{{if .HasAvailableInheritedFlags}}

Global Flags:
{{.InheritedFlags.FlagUsages | trimTrailingWhitespaces}}{{end}}{{if .HasHelpSubCommands}}

Additional help topics:{{range .Commands}}{{if .IsAdditionalHelpTopicCommand}}
  {{rpad .CommandPath .CommandPathPadding}} {{.Short}}{{end}}{{end}}{{end}}{{if .HasAvailableSubCommands}}

Use "{{.CommandPath}} [command] --help" for more information about a command.{{end}}
`
}
```

---

cobra中核心运行函数有个2个签名: 一个是无error的`Run`, 一个是有error的`RunE`. 而运行时, 根据设置情况运行(而不是必须只能提供一个类似全集性质的签名):

```go
func exec() {
  ...

	if c.RunE != nil {
		if err := c.RunE(c, argWoFlags); err != nil {
			return err
		}
	} else {
		c.Run(c, argWoFlags)
	}
}
```

---

一个推荐的命令行描述结构化说明(值得学习):

- `[-F file | -D dir]`定义了可选的参数
- `{-F file | -D dir}`定义了必须的参数

```go
type Command struct {
  // https://github.com/spf13/cobra/blob/6b0bd3076cfafd1c108264ed1e4aa0c0fe3f8537/command.go#L49
  // Use is the one-line usage message.
  // Recommended syntax is as follow:
  //   [ ] identifies an optional argument. Arguments that are not enclosed in brackets are required.
  //   ... indicates that you can specify multiple values for the previous argument.
  //   |   indicates mutually exclusive information. You can use the argument to the left of the separator or the
  //       argument to the right of the separator. You cannot use both arguments in a single use of the command.
  //   { } delimits a set of mutually exclusive arguments when one of the arguments is required. If the arguments are
  //       optional, they are enclosed in brackets ([ ]).
  // Example: add [-F file | -D dir]... [-f format] profile
  Use string

  ...
}
```
