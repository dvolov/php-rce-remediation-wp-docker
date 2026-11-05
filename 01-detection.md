# 01 — Detection

How I found out that my WordPress site was running a vulnerable PHP version.

## Trigger 1: APISEC|CON 2026

In October 2026 I attended APISEC|CON 2026. During one of the sessions, two
recently disclosed PHP core vulnerabilities were mentioned:

- **CVE-2026-109618**
- **CVE-2026-109619**

Both are unauthenticated remote code execution (RCE) issues in the PHP engine
itself, rated CVSS 9.2. The mention was brief — not a dedicated talk, just a
point in a session about something else. I did not act on it immediately.

## Trigger 2: The Hacker News article

The Hacker News published its write-up on the same day, October 21, 2026
([link](https://thehackernews.com/2026/10/php-issues-emergency-patches-for-two.html)).
That article turned a vague mention into concrete facts:

1. The vulnerabilities affect PHP versions starting from 5.4.
2. Official patches exist only for **8.2.35, 8.3.36, 8.4.27 and 8.5.12**.
   Older branches (7.x, 8.0, 8.1) will not receive official fixes.
3. The flaws are in the PHP runtime, not in application code — updating
   WordPress itself does not help; the PHP version underneath has to be updated.

That last point was the one that made me act. My site runs WordPress in Docker,
and the PHP version comes from the `wordpress` image, not from WordPress
updates.

## First check: what PHP version is the container running?

The next day I checked the PHP version inside the running container:

```bash
docker exec <project>-wp php -v
```

Output:

```text
PHP 8.3.27 (cli) (built: Nov 4 2025 04:11:03) (NTS)
Copyright (c) The PHP Group
Zend Engine v4.3.27, Copyright (c) Zend Technologies
    with Zend OPcache v8.3.27, Copyright (c), by Zend Technologies
```

8.3.27 is below 8.3.36, which is the patched version for the 8.3 branch. The build date (`Nov 4 2025`) also predates the patch release, confirming that this was a pre-patch build — not a version with a backported fix. That meant the site was running a PHP version that had not received the fix for CVE-2026-109618 / CVE-2026-109619.

## Why I did not run a scanner before the update

I could have run `php2shell-checker` against the site before updating, but at
that point I already had a more reliable signal: the exact PHP version from
`php -v`. A scanner reading HTTP headers can only report "potentially
vulnerable" or "unknown" — it cannot see the exact patch level. The `php -v`
output inside the container is authoritative.

The scanner became useful **after** the update, as an independent confirmation
that the new version is not reported as vulnerable. That is covered in
[`04-verification.md`](04-verification.md).
