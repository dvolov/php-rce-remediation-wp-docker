# PHP RCE Remediation — WordPress in Docker

A short incident-prevention report documenting how I detected and remediated
CVE-2026-109618 / CVE-2026-109619 on a WordPress site running in Docker.

The site was running PHP 8.3.27 inside the official `wordpress` image, which is
vulnerable to two critical PHP core vulnerabilities (CVSS 9.2, unauthenticated
RCE). Official patches were released for PHP 8.2.35, 8.3.36, 8.4.27 and 8.5.12.
After pulling the updated image, the site runs PHP 8.3.36 and the vulnerability
scanner reports `OK`.

## Context

- **When:** October 21, 2026, at APISEC|CON 2026.
- **What:** During one of the sessions at the conference, two recent PHP core
  vulnerabilities were mentioned — CVE-2026-109618 and CVE-2026-109619. I later
  read the detailed write-up in The Hacker News ([link](https://thehackernews.com/2026/10/php-issues-emergency-patches-for-two.html)), which confirmed the affected versions and the patched releases.
- **Why it mattered to me:** My own WordPress site runs in a Docker container,
  and I had not verified which PHP version the container was actually using.

## What I did

1. Checked the PHP version inside the running container.
2. Confirmed that 8.3.27 is below the patched version (8.3.36).
3. Backed up the database and the WordPress files as a precaution.
4. Pulled the updated `wordpress` image and recreated the container.
5. Verified the new PHP version and ran `php2shell-checker` to confirm the site
   is no longer reported as vulnerable.

Detailed steps, commands and reasoning are in the files below.

## Result

| Before                                          | After                              |
| ----------------------------------------------- | ---------------------------------- |
| PHP 8.3.27 (`php -v`)                           | PHP 8.3.36 (`php -v`)              |
| Vulnerable to CVE-2026-109618 / CVE-2026-109619 | Patched, `php2shell-checker`: `OK` |

The site was not compromised. The remediation took under an hour, including
backup and verification.

## Structure

- [`01-detection.md`](01-detection.md) — how I found out the site was vulnerable.
- [`02-risk-assessment.md`](02-risk-assessment.md) — what the vulnerabilities
  allow, and what was at risk.
- [`03-remediation.md`](03-remediation.md) — the remediation plan and the exact
  commands I ran.
- [`04-verification.md`](04-verification.md) — how I confirmed the fix worked.
- [`05-lessons-learned.md`](05-lessons-learned.md) — what worked, and what I would do next.

## A note on privacy

The real domain and project name are replaced with placeholders
(`example.com`, `<project>`). `example.com` is reserved for documentation
(RFC 2606). The commands, versions and outputs are from the actual
remediation — only the identifiers are masked.

## Environment

- WordPress running in Docker (official `wordpress:latest` image) in container
  `<project>-wp`.
- MariaDB running in Docker (official `mariadb:latest` image) in container
  `<project>-db`.
- Bind mounts for both WordPress files and the database:
  - `/opt/docker/<project>-wp/wp_data` → `/var/www/html`
  - `/opt/docker/<project>-db/db_data` → `/var/lib/mysql`
- Docker Compose v5.6.1 (`docker compose`, v2 syntax).
- Scanner: [`php2shell-checker`](https://github.com/openai/php2shell-checker)
