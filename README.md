# ESP-KVM skills

Skills that let an AI agent operate a computer through an
[ESP-KVM](https://espkvm.io) - open-source IP-KVM firmware for the ESP32-P4.

A KVM already lets a *person* see a machine that has no working operating
system. This device can also hand back what a BIOS or a boot loader is showing
**as exact characters** rather than as a picture, so a model can read the
firmware menu, decide, and press a key.

Everything here talks to the device over its normal HTTPS API. Nothing is
installed on the KVM, and nothing here reaches the internet.

## Install

```
/plugin marketplace add espkvm/skills
/plugin install espkvm@espkvm
```

That gives you `/espkvm:operate`. Claude also loads it on its own when a task is
clearly about driving a machine through a KVM.

To try it without installing, copy `plugins/espkvm/skills/operate/` into
`~/.claude/skills/` (then it is `/operate`) or into a project's
`.claude/skills/`.

## What is in it

| Skill | |
|---|---|
| `operate` | sign in, read the screen as text, keyboard and pointer, virtual media, runbooks, power, settings - and the traps that cost an hour if nobody tells you |

## Before it will do anything

The keyboard, pointer and screenshot endpoints are **off by default** on the
device. Turn on **Agent REST API** in Settings -> Security. It is off because it
hands a program the same control over the target that the console has.

Reading the screen as text, runbooks, virtual media and power work without it.

## Not this repository

- The firmware itself: [espkvm/espkvm](https://github.com/espkvm/espkvm)
- An MCP server wrapping the same calls as tools, for hosts that speak MCP:
  [espkvm/mcp](https://github.com/espkvm/mcp)

The skill and the MCP server are two answers to the same question. The skill
suits an agent that can run `curl` and wants the whole API; the MCP server suits
a host that wants a fixed set of tools with a permission switch on input and
power.

## Licence

Apache-2.0, like the firmware.
