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

This demonstrated remote command execution with root-level privileges in the observed execution context.

## 5. Post-Exploitation Enumeration

After obtaining shell access, I examined the environment to understand the application's configuration and available services.

The investigation included:

* Environment variables and application credentials.
* Running processes and listening services.
* Localhost-only services.
* Application directories and configuration files.
* SSH access and available user accounts.

I identified additional application components running locally, including a Gogs Git service.

## 6. Gogs Investigation

I discovered a locally accessible Gogs instance running on port `3001`.

I investigated the service and identified a potential connection to:

**CVE-2025-8110 — Gogs remote code execution through Git configuration injection.** <a href="https://github.com/zAbuQasem/gogs-CVE-2025-8110.git"> here </a>

I examined the vulnerability's proof of concept and investigated the conditions required for exploitation.

This was a separate application-level investigation within the same lab environment.

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
