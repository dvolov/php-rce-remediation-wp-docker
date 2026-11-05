# 05 — Lessons Learned

What worked, and what I would do next.

## What worked

**Checking `php -v` inside the container first.**
Before running any external scanner, I checked the exact PHP version from the
container itself. That gave me an authoritative answer — not "potentially
vulnerable" or "unknown", but `8.3.27`, which is below the first fixed release
for the 8.3 branch. A scanner reading HTTP headers cannot see the exact patch
level; `php -v` can.

**Taking a backup even though bind mounts made it optional.**
`--force-recreate` does not touch bind-mounted directories, so the data was
safe by design. But the backup cost a few minutes and protected against a
scenario where the new image would fail to start and I would need to roll back.
For a production site, that is a reasonable trade.

**Understanding `--force-recreate` before using it.**
`docker compose up -d` alone would not have recreated the container, because
the image tag (`wordpress:latest`) did not change. The flag forces recreation
from the freshly pulled image, and because the data lives in bind mounts on
the host, recreation does not touch it.

**Confirming the result from two independent sources.**
`php -v` inside the container (authoritative) and `php2shell-checker` against
the site (external) agreed: PHP 8.3.36, patched. Using two sources rather than
one made the conclusion stronger.

**Checking the SAPI inside the container.**
`apachectl -M` showed `php_module (shared)`, which confirms the container runs
Apache `mod_php` — the highest-risk configuration for CVE-2026-109618. Knowing
this made the risk assessment concrete rather than generic: the vulnerable code
path was the one that could execute before any application code handled the
request.

## What I would do next

**Use WP-CLI for WordPress core updates.**
The admin panel asked for FTP credentials because the web process does not have
write access to the files it would need to replace. That is a separate issue from the PHP vulnerability and outside the scope of this report, but WP-CLI inside the container
([`wp core update`](https://developer.wordpress.org/cli/commands/core/update/), run as `www-data`) avoids the FTP prompt entirely and can be scripted.

**Keep a repeatable verification routine.**
The three checks used here — `php -v` inside the container, `php2shell-checker`
against the site, and loading the site in a browser — form a short, repeatable
routine. Running the same checks after any PHP or image update makes the result
auditable without relying on memory.

**Automate image update tracking.**
`wordpress:latest` is convenient, but "latest" is only latest when the image is
actually pulled. A scheduled `docker compose pull` with a notification, or a
tool like Watchtower, closes the gap between "a patch exists" and "the container
is running the patched version".
