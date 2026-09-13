# Silentium Hack The Box

A penetration testing lab documenting the discovery and exploitation of vulnerabilities in a self-hosted Git service and an AI workflow automation platform.

## Overview

| Information      | Details                                                                                    |
| ---------------- | ------------------------------------------------------------------------------------------ |
| Platform         | Hack The Box                                                                               |
| Machine          | Silentium                                                                                  |
| Difficulty       | Easy                                                                            |
| Operating System | Linux                                                                                      |
| Focus Areas      | Web Enumeration, API Security, Authentication, Remote Code Execution, Privilege Escalation |

## Objectives

* Enumerate exposed services and discover hidden web applications.
* Investigate authentication and password-reset vulnerabilities.
* Explore API functionality and application configuration.
* Obtain remote command execution in the lab environment.
* Investigate local services and privilege escalation opportunities.

## 1. Initial Enumeration

I started by scanning the target to identify exposed ports and services.

```bash
nmap -A -p22,80 <TARGET_IP>
```
![nmap](/NMAP.png)
### Findings
| Port | Service | Observation               |
| ---- | ------- | ------------------------- |
| 22   | SSH     | OpenSSH running on Ubuntu |
| 80   | HTTP    | Nginx web server          |

The HTTP service redirected requests to the `silentium.htb` hostname.

I configured the hostname locally:

```bash
echo "<TARGET_IP> silentium.htb" | sudo tee -a /etc/hosts
```

### Virtual Host Enumeration

I used Gobuster to discover additional virtual hosts.

```bash
gobuster vhost \
  -u http://silentium.htb \
  -w /usr/share/wordlists/dirb/common.txt \
  --append-domain
```
![subdomain](subdomain.png)

This revealed a staging subdomain:

```text
staging.silentium.htb
```

The staging application became the primary focus of my investigation.

![sub](staging.silentium.htb.png)

### Impact

The exposed reset token could be used to reset an account's password without the original account credentials.

This provided an authentication foothold in the application.

### User Discovery

While exploring the staging website, I found the username ben on the homepage.

Based on the discovered username and the target's domain, I identified the following email address:
```text
ben@silentium.htb
```
I used this email address when investigating the password-reset functionality.

This discovery provided a potential account to investigate and helped me proceed with the password-reset vulnerability analysis.

## 2. Password Reset Token Disclosure

The staging application exposed a password-reset functionality.

While investigating its API, I identified an unauthenticated password-reset token disclosure affecting the Flowise application.

The vulnerability was associated with:

**CVE-2025-58434 — Flowise unauthenticated password-reset token disclosure leading to account takeover.**

The GitHub Advisory Database identified affected Flowise versions as `3.0.5` and below, with `3.0.6` listed as the patched version. <a href="https://github.com/advisories/GHSA-wgpv-6j63-x5ph"> here </a>

![exploit](CVE-2025-58434.png)

![account](account.png)

## 3. Flowise API and Authentication

After obtaining access to the application, I explored its API endpoints and authentication mechanisms.

I examined:

* Password-reset requests and responses.
* API authentication headers.
* API key management.
* Application configuration.
* Available workflow and tool functionality.

I also investigated how Flowise handled requests to its internal services.

![api](API.png)
![subdomain](Shell.png)

## 4. Remote Code Execution

During further investigation, I discovered an MCP tool configuration that allowed server-side JavaScript execution.

The vulnerable configuration could be abused to execute operating-system commands from the application.

I used this behavior to establish a reverse shell in the authorized lab environment.

### Result

```text
uid=0(root) gid=0(root) groups=...
```

## 5. Pivoting to the Host

After obtaining a shell inside the Docker container, I began investigating how to access the underlying host machine.

Credential Discovery

I enumerated the container's environment variables using:

env

This revealed sensitive application configuration, including credentials for the local user ben.

Since SSH was exposed on port 22, I used the discovered credentials to authenticate as ben on the host machine.
```text
SSH Access
ssh ben@silentium.htb
```
After successfully authenticating, I gained access to the host as the ben user.

User Flag

I retrieved the user flag from Ben's home directory:
```text
cat /home/ben/user.txt
```
User flag captured!


## 6. Gogs Investigation

Privilege Escalation: Gogs Arbitrary File Write (CVE-2025-8110) <a href="https://github.com/zAbuQasem/gogs-CVE-2025-8110.git"> here </a>

After gaining SSH access as ben, I continued enumerating the host to identify potential privilege escalation opportunities.

Gogs Process Enumeration

I inspected the running processes using:

ps aux | grep gogs

This revealed a locally running Gogs instance. Further investigation showed that the Gogs web process was running as root.

The service was accessible internally on port 3000 (and was also referenced on port 3001 during my investigation).

SSH Local Port Forwarding

Since the Gogs web interface was only accessible from the host, I used SSH local port forwarding to expose the internal service to my attacking machine.

ssh -L 8080:127.0.0.1:3000 ben@silentium.htb

This forwarded my local port 8080 to port 3000 on the remote host.

I could then access the Gogs web interface through:

http://127.0.0.1:8080
Vulnerability: CVE-2025-8110

During my investigation, I identified CVE-2025-8110, an arbitrary file write vulnerability affecting Gogs.

The vulnerability is caused by improper handling of symbolic links when files are updated through the Gogs API.

An attacker can create a repository containing a symbolic link that points outside the repository and then update the linked file through the API. This can cause Gogs to write to an arbitrary file using the permissions of the Gogs process.

Because the Gogs web process was running as root, successful exploitation could result in arbitrary file writes with root-level privileges.

Exploitation Overview

The vulnerability investigation involved the following concepts:

Accessing the internal Gogs web interface through SSH port forwarding.
Investigating the repository and API functionality.
Understanding how symbolic links can reference files outside a repository.
Examining how file updates through the API interact with symbolic links.
Evaluating the impact of the Gogs process running with root privileges. 

## 7. Key Takeaways

This machine provided practical experience with several areas of penetration testing:

* Virtual host enumeration and hostname resolution.
* Understanding HTTP requests, headers, and API responses.
* Identifying authentication and password-reset weaknesses.
* Investigating JSON-based API functionality.
* Understanding the security implications of exposed API keys and configuration values.
* Exploiting server-side command execution in a controlled environment.
* Enumerating Linux services and application configurations.
* Investigating known CVEs and adapting proof-of-concept research to a lab environment.

## Tools Used

* Nmap
* Gobuster
* cURL
* Netcat
* SSH
* Linux command-line utilities
* Burp Suite

## Conclusion

Silentium was a valuable hands-on exercise in web application security and Linux enumeration.

The investigation demonstrated how weaknesses in authentication, application configuration, and server-side execution can combine to compromise an application.

It also reinforced the importance of understanding HTTP and API behavior rather than relying solely on automated tools or existing exploit scripts.

## Disclaimer

This documentation is for educational purposes and reflects work performed in an authorized Hack The Box lab environment.

All sensitive information, including credentials, API keys, tokens, and target addresses, has been omitted.

Do not use these techniques against systems without explicit authorization.
