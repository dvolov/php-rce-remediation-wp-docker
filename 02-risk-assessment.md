# 02 — Risk Assessment

What the vulnerabilities allow, and what was at risk on my site.

## What the vulnerabilities allow

CVE-2026-109618 and CVE-2026-109619 (collectively referred to as PHP2Shell)
are flaws in the PHP runtime itself, not in any application running on top of
it. That distinction matters: a site is exposed even if its own code is secure,
because the vulnerable code executes before the application handles the request.

## Three impact scenarios

**1. Unauthenticated remote code execution (RCE).**
An attacker without an account can execute arbitrary code on the server with
the privileges of the web server process. The highest-risk configuration is
Apache `mod_php`, where a single unauthenticated request can trigger the attack
before any application code runs. PHP-FPM is also affected, but the confirmed
FPM path depends on an operation inside the running application.

RCE as `www-data` is not the end state. From that point, an attacker would
enumerate the host for privilege escalation vectors — sudo misconfigurations,
SUID binaries, writable cron jobs, Docker group membership. In a containerized
setup the scope of what an attacker can reach is smaller, but the container
still exposes application secrets and network access.

**2. Memory disclosure without code execution.**
Even without achieving RCE, an attacker can read memory from the PHP worker
process. That memory may contain passwords, authentication tokens, API keys and
certificates belonging to other requests handled by the same worker.

**3. Denial of service.**
A malformed request that triggers the vulnerable code path can crash the PHP
worker, making the site temporarily unavailable. This does not require a
successful exploit — just the right request.

## What was at risk on my site

My setup: WordPress in Docker, PHP 8.3.27, database in a separate container.
PHP runs through `apache2handler` (Apache `mod_php`, confirmed by `php_module`
in `apachectl -M`), which is the highest-risk configuration for
CVE-2026-109618 — the vulnerable code can execute before any application code
handles the request.

- **WordPress files** (`/var/www/html` on the host, bind-mounted into the
  container). This includes `wp-config.php`, but in my setup that file does not
  store the database password directly: it reads `WORDPRESS_DB_PASSWORD` from
  the container's environment at runtime. The password itself lives in the
  `.env` file on the host, which Docker Compose passes to the container as
  environment variables.
- **The database** — user accounts, password hashes, posts, and any data stored
  by plugins.
- **PHP worker memory** — session tokens for logged-in users, and any secrets
  passed through the application during a request.

The `.env` file on the host is therefore part of the attack surface: if an
attacker achieves RCE, they can read the container's environment (and thus the
database password) even though `wp-config.php` itself does not contain it. The
file is owned by my sudo user with `600` permissions, which limits exposure to
other host users, but does not change the in-container attack path.

The site is not a high-profile target, but the flaws do not require a targeted
attack. Automated scanning for vulnerable PHP versions was already underway
when the patch was released, so "not interesting to attackers" was not a
reliable assumption.

## Why the risk was real, not theoretical

Three factors raised the risk above "theoretical vulnerability":

1. **No patch for my branch at the time of detection.** 8.3.27 predates the
   first fixed release (8.3.36). Until I updated, the only options were
   mitigations, not a fix.
2. **Active exploitation attempts.** According to The Hacker News, Cloudflare
   observed early exploitation attempts before the patch shipped, and watchTowr
   reported active attempts shortly after. This was not a "patch at your leisure" situation.
3. **The vulnerability is in the engine, not the app.** Updating WordPress,
   plugins or themes would not have helped. Only updating the PHP runtime
   (in my case, the Docker image) would have closed the gap.
