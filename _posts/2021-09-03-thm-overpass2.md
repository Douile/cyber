---
title: "Overpass 2: hacked Writeup"
tags: writeup tryhackme
---

# [tryhackme/overpass2hacked](https://tryhackme.com/room/overpass2hacked)

## Forensics

### Locating the reverse shell
Locating the attacker uploader reverse shell is as simple as filtering the pcap file for 
HTTP uploads, the exact filter I used was `http.request.method == "POST"`: this allows
you to view all files that were uploaded via HTTP over the duration of the capture.

After reading the uploaded reverse shell you learn that they are using port `4242` for
the reverse connection, as such if you filter your pcap for tcp streams on port `4242`
(`tcp.port == 4242`) you are able to follow the TCP stream and view the entire reverse
shell session.

### Investigating the source code
Most questions in this section are easily located in the
[main.go](https://github.com/NinjaJc01/ssh-backdoor/blob/master/main.go) file. The
hash used by the attacker is found in the reverse shell TCP stream when they call the
backdoor.

#### Cracking the hash
In order to crack the hash we need to find the format used, luckily we have the source
code. The handily named `hashPassword()` function contains a [line](https://github.com/NinjaJc01/ssh-backdoor/blob/39556dba26a5cc32a4c77e76b53c28e1f281da4c/main.go#L61)
that shows us the hash is the sha512 of the password with the salt added to the end.
Luckily we also have the static salt from the source code before and the question tells
us the wordlist to use.

**Setting up the hash file**

There is a specific format this file needs to be in for john to be able to read it.
```
# username:hash$salt
user:6d05358f090eea56a238af02e47d44ee5489d234810ef6240280857ec69712a3e5e370b8a41899d0196ade16c0d54327c5654019292cbfe0b5e98ad1fec71bed$1c362db832f3f864c8c2fe05f2002a05
```

We can then get to cracking using john's dynamic format
```shell
# Don't forget the '' otherwise your shell will expand the $
$ john ./crackme.john --format=dynamic='sha512($s.$p)' --wordlist=/usr/share/wordlists/rockyou.txt
```

Now we have the backdoor password.

## Re-Entry
The easiest way to access this machine is through the backdoor as we already have the
creds. Don't forget to use port 2222 though (found through the source code).
```shell
$ ssh user@$target -p 2222
```

The user flag is now within reach `/home/james/user.txt`

### Escalating privledge
Although it seemed like the attackers had sudo access in the reverse shell and we have
the password they used for it for some reason the password they used no longer works.
However there is for some reason a SUID binary in `/home/james` called `.suid_bash`.
Checking [gtfobins](https://gtfobins.github.io/gtfobins/bash/#suid) we find we can
escalate via this binary.

```shell
$ pwd
/home/james
$ ./.suid_bash -p
# cat /root/root.txt
```

:)
