# StealMyBitcoin — investigation notes

Sources: stealmybitcoin.com, github.com/KillaBoi/StealMyBitcoin-PoC, and
security-press coverage (gbhackers, lilting.ch, pasqualepillitteri.it, coin-turk).
Analysis kept at the conceptual "how it works" level — no weaponized payload or
build-specific offsets reproduced here.

## What it is

Not a fraud/scam. It is a capture-the-flag (CTF) bug bounty. A researcher placed a
real Bitcoin wallet's private key on a live server and offered the coins to anyone
who could achieve remote code execution and read the key. The "steal" is the
invitation, not a trick played on the visitor.

- Solved in ~19 hours.
- Prize pot: 0.02511679 BTC (~$2,021), won 25 Oct 2026, 15:47 UTC.
- Winner: `KillaBoi`, who published a standard-library-only Python 3.11+ PoC.

## Target setup

- PHP 8.4.26, deliberately vulnerable to CVE-2026-109618 / CVE-2026-109619
  ("PHP2Shell", CVSS v4.0 9.2).
- Bug class: PHP session upload-progress / multipart parsing memory handling.
- Flag: the PHP user could read the key only by launching a `print-flag` binary,
  so winning required running a command on the box.
- Lab reproduction: Alpine PHP 8.4.26 CLI server behind nginx, in Docker.

## Exploit chain (conceptual)

1. Memory exhaustion — a small multipart upload with many nested fields forces PHP
   to allocate far more memory than the request's wire size implies. It hits the
   128 MiB limit mid-parse and leaves stale state behind in a worker that is reused.
2. Heap leak — follow-up requests read that leftover memory back out through a
   bloated `PHPSESSID` cookie. Pointer-like values are harvested; a heap base is
   accepted only when several candidates agree.
3. Code leak / ASLR bypass — a second exhaustion run leaks a pointer to a known
   internal PHP function (`zval_ptr_dtor`). Its offset from the binary base is fixed
   for a given build, revealing where the code is loaded despite ASLR.
4. Control-flow hijack -> RCE — a crafted structure is placed where PHP's cleanup
   routine will find it, causing PHP to call its internal handler for `system` with
   an attacker-chosen command. Output returns in the HTTP response (proof: fresh
   nonce, `id` output, hash of the `print-flag` binary).

## Weak point vs. hard part

- Weak point (the entry): the memory-disclosure primitive. The upload-progress bug
  reliably leaks heap memory — the crack everything else is pried from.
- Hard part (the real challenge):
  - Defeating ASLR — you must leak and compute memory layout before placing anything.
  - Build-specific fragility — offsets are tuned to one exact Alpine PHP 8.4.26 build
    and a specific worker count; change the build and it breaks.
  - Reliability — author reports only ~7-in-10 success even in their own lab, because
    memory corruption fights nondeterministic heap state.

## The PHP-version "trick"

The lull people fall for: a recent PHP version looks safe, because vulnerabilities
usually hit old releases. But 8.4.26 specifically predated the fixes shipped on
21 Oct 2026. It was not an old, obviously-vulnerable range — it was the narrow
window of a brand-new version right before the patch. The version string looks
current and "surely patched," while the server sits just below the fix line.

## The twist

The PoC author says the final live run returned a private key with a valid checksum
that did NOT derive the advertised wallet address — attributed to a mistake by the
challenge creator, reportedly acknowledged. So "won," with an asterisk.

## Caveat

Based on the site, the repo README, and press coverage. Nothing was run locally, and
the live-challenge/recovered-key claims cannot be verified from these pages alone.
