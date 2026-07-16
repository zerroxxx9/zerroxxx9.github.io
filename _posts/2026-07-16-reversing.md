---
layout: post
title: "HackTheBox - Cyberpsychosis (Easy)"
date: 2026-07-16 22:20:00 +0200
description: "A documentation of my early steps learning to reverse engineer"
categories: [hackthebox]
tags: [hackthebox, reversing]
---

The challenge was simple on paper: disarm a Linux kernel module rootkit and recover hidden data. In practice, this was my first Hack The Box challenge, and until now I had mostly played with crackmes. Getting an IP address and port as part of a reversing challenge already threw me off for a moment.

The provided file was `diamorphine.ko`. A `.ko` file is a Linux kernel object: it can be loaded with `insmod`, removed with `rmmod`, and listed with `lsmod` if the module is not hiding itself.

`file` identified it as:

```text
ELF 64-bit LSB relocatable, x86-64, with debug_info, not stripped
```

That last part mattered. Because the module was not stripped, Ghidra still showed useful function names instead of leaving me with anonymous blobs. `strings` produced too much noise to be helpful, so I moved into Ghidra and started from the symbol tree.

The interesting functions were:

- diamorphine_init
- diamorphine_cleanup
- find_task
- get_syscall_table_bf
- give_root
- hacked_getdents
- hacked_getdents64
- hacked_kill
- is_invisible
- module_hide
- module_show

Every kernel module has an initialization and cleanup path, which in this case were `diamorphine_init` and `diamorphine_cleanup`. The rest of the names were already a pretty good map of the rootkit:

- `module_hide` and `module_show` remove or restore the module from the kernel's module list, which is why `lsmod` may not show it.
- `give_root` does exactly what the name suggests.
- `get_syscall_table_bf` locates the syscall table by brute force.
- `is_invisible` checks whether a process has been marked as hidden.

From there, the two most important hooks were `hacked_kill` and `hacked_getdents64`.

## `hacked_kill`: the hidden command channel

The rootkit hijacks the kill(pid, sig) syscall and repurposes specific signal numbers as covert commands. Any signal not matched below is passed through to the real kill().

| Signal | Hex | Effect |
| --- | --- | --- |
| `46` | `0x2e` | Toggle rootkit visibility by hiding or showing it in `lsmod`. |
| `64` | `0x40` | Grant root to the calling process through `prepare_creds` and `commit_creds`. |
| `31` | `0x1f` | Toggle the invisible bit on the process matching the provided PID. |

This means `kill` is not just being used as `kill` anymore. It becomes a command interface into the rootkit.

## `hacked_getdents64`: file and directory hiding

`getdents64` is used by tools like `ls` and `find` to read directory entries. The rootkit calls the real syscall first, then edits the returned results before userland sees them.

It filters entries in two places:

- For `/proc`, it parses directory names as PIDs. If the matching `task_struct` has the invisible bit set, that process is removed from the listing.
- For normal directories, it compares filenames against the constant `psychosis`, decoded from the little-endian comparison in the code. Anything whose name starts with that prefix is hidden.

The important takeaway: files or directories beginning with `psychosis` are invisible to normal directory listings, but they still exist on disk. If you know the exact path, direct access through tools like `cat` or `stat` can still work because the hiding happens when directory entries are listed.

## Disarming the rootkit

Once the behavior was clear, disarming it came down to using its own command channel against it.

```bash
# 1. Confirm whether the module is hidden.
lsmod                           # diamorphine is missing because the module hides itself

# 2. Unhide the module with magic signal 46.
kill -46 0
lsmod                           # diamorphine appears

# 3. Escalate to root using signal 64.
kill -64 0
id

# 4. Unload the module, which reverts the syscall table hooks.
rmmod diamorphine
lsmod                           # diamorphine is gone again
```

## Recovering the hidden data

With the module unloaded and the hooks removed, normal directory listings work again:

```bash
find / -iname "psychosis*" 2>/dev/null
```

Before disarming the module, `find` and `ls` are both filtered because they rely on `getdents64` under the hood. After removing the module, the hidden `psychosis*` path becomes visible and the data can be recovered normally.
