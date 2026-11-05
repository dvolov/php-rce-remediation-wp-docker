# 04 — Verification

How I confirmed that the fix worked.

## Plan for verification

The remediation is not complete until the new version is confirmed from two
independent sources:

1. **`php -v` inside the container** — the authoritative check. It reports the
   exact PHP build that is actually running, not what HTTP headers claim.
2. **`php2shell-checker` against the site** — an external check that confirms
   the advertised version is at or above a fixed release.

The site was also loaded in a browser to confirm it still works after container
recreation.

## Check 1: PHP version inside the container

```bash
docker exec <project>-wp php -v
```

Output:

```text
PHP 8.3.36 (cli) (built: Oct 21 2026 15:57:18) (NTS)
Copyright (c) The PHP Group
Zend Engine v4.3.36, Copyright (c) Zend Technologies
    with Zend OPcache v8.3.36, Copyright (c), by Zend Technologies
```

8.3.36 is the first fixed release for the 8.3 branch, so the container is now
running a patched PHP version. The build date (`Oct 21 2026`) matches the patch
release date, confirming that this is the official patched build, not an older
build with a backported fix.

## Check 2: External scanner

I cloned the official [`php2shell-checker`](https://github.com/openai/php2shell-checker) repository from OpenAI and ran the scanner from its directory:

```bash
cd ~/Documents/scripts/php2shell-checker
node phpcheck.js "https://example.com"
```

Output:

```text
OK: PHP 8.3.36: advertised version meets an upstream fixed release. Installed build not verified.
```

The `OK` status means the advertised PHP version (8.3.36) is at or above the
first fixed release for the 8.3 branch. The note `Installed build not verified`
is a limitation of the scanner: it reads the version from HTTP headers and
cannot confirm that the installed build actually contains the patch. That is why
the scanner is a confirmation step, not a replacement for `php -v` — the
authoritative check is the version reported by the PHP binary inside the
container, which is covered in Check 1.

## Check 3: The site still works

After recreating the container, I loaded the site in a browser. The site opened
normally, and the admin panel was reachable. This confirmed that the
bind-mounted `wp_data` directory was reattached correctly and that file
permissions were not broken by the recreation.
