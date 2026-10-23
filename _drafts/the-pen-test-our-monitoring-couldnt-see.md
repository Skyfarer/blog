---
layout: post
title: The pen test our monitoring couldn't see
categories: tech
---

Earlier this month our company's security team ran a penetration test against a fleet of about twenty of our Rails apps. We didn't know it was a test. For several days we treated it as a real, active attack, and we investigated, patched, and cleaned up accordingly.

The test found a real bug: a SQL injection in shared sign-up code, where a string was interpolated into a `where` clause. But the more useful result came from the response. Almost every question we asked afterwards ("how long has this been going on?", "who is it?", "what did they get?") ran into a gap in what we had logged, watched, or kept. A pen test is supposed to show you where your defences fail. This one did, including a few places we hadn't thought to look.

This post is about those gaps. The bug is the least interesting part.

## How we noticed (by accident)

We did not detect the test. We noticed an outage.

One of our apps hung for about twenty minutes. The cause had nothing to do with security: a single page that rendered 192 image thumbnails tied up all five of the app's Puma threads, each one waiting on object storage. A manual restart fixed it. The same signature had appeared in the proxy's error log on four earlier dates, and nobody had connected them.

While reading logs for that outage, we saw that another app had hung the previous day, and this time the cause was different. Its threads were pinned by `sleep(3)` calls inside SQL queries that the sign-up form had passed straight to the database. That is a time-based blind injection, and it was being run by hand.

Two unrelated problems, one afternoon, and neither was caught by an alert. The tester's `sleep()` payloads caused a real outage of their own, which is worth remembering when you scope a test against production.

## Gap 1: the proxy that nobody was watching

We run two nginx reverse proxies. Public DNS points at only one. The second was a spare, built for a load-balanced pair that was never finished, and no production traffic was meant to reach it.

About 70% of the tester's sign-up requests went through it.

nginx routes on the `Host` header and SNI, not on DNS, so anyone who finds the IP can supply hostnames themselves. The testers found it (we don't know how; a neighbouring-address scan and an internet-wide scan index both fit) and then supplied the hostnames. The list looked like it came from Certificate Transparency logs: it included retired services, and a name with no public DNS record at all but a Let's Encrypt certificate. Every certificate you issue publishes its hostnames. That history can't be removed.

The spare proxy did run fail2ban. It had banned nothing since September 25. Here is why:

1. The jail watched `access.log`, which had been zero bytes since that date.
2. logrotate's post-rotate step runs `nginx -s reload`, and the reload was failing.
3. The reload failed because `nginx -t` failed, because a vhost added in August referenced a certificate that didn't exist.
4. nginx therefore kept writing to the old, renamed file, and fail2ban watched an empty one.

A jail with a correct filter and a clean config looked healthy from every angle except the one that mattered. We compared the filter against the test traffic afterwards: a working jail would have matched 5,894 of 9,773 requests from the tester's main address that day and banned it early.

**Lesson:** a security control needs a heartbeat of its own. "The service is running" is not the same as "the service is seeing traffic." An alert on *the log file being empty* or *no bans in N days* would have caught this a fortnight sooner.

## Gap 2: filters that judge by status code

Our healthy proxy had a healthy fail2ban with three jails. None of them could see this activity, for two reasons.

- The injection payload is in the POST body, which nginx doesn't log.
- Both a successful injection and a rejected sign-up return HTTP 200. Our jails keyed on 400/404-class errors.

Of 1,743 injection attempts through the spare proxy, the jail would have matched five.

The tester's addresses did eventually get banned, but by tripping unrelated 401s while probing admin paths, and each ban lapsed after 24 hours. That is accidental detection, not a control.

What worked was changing what we counted. We looked at the history of `POST /users` per address in ten-minute windows: 158 addresses made one or two, 33 made three or four, and 68 made five or more (the worst peaked at 358). A jail that bans five sign-ups in ten minutes, regardless of response code, cuts that off and leaves normal users alone. It has caveats. There is no allowlist, and several real users behind one corporate egress address could be banned together.

**Lesson:** if an attack looks like normal traffic to your tooling, count the *behaviour* instead of the *outcome*.

## Gap 3: we couldn't answer "how long?"

The Rails apps log to journald with no retention policy. The nightly backup archived each app user's home directory, which didn't contain the logs. The backends log only the proxy's address. Real client addresses exist only in the proxies' own logs.

The earliest payload we could find was October 8. The earliest log we had was August 8. Anything before that is simply unknowable: we can't confirm it or rule it out.

We also found a quieter version of the same problem. One app's `~/logs` was a symlink, and `tar` doesn't follow symlinks, so that app's logs had never been in any backup, on any night.

The fix was dull and fast: add the app user to the `systemd-journal` group, export the journal in the nightly backup, write a full-history export, and copy everything to a second location. Once forensics is happening you want evidence in three places, and we had it in one.

**Lesson:** decide your retention before the incident. The question you most need answered will be about a date you wish you had logs for.

## Gap 4: our own search and our own patch

Two smaller misses, both from checking the thing we expected rather than the thing that was there.

**The search.** Our first pass counted payloads that begin with a quote character. That found the classic injections and missed a second technique entirely: submitting `%@example.com` as an email. It has no quote, passes every validation, matches every entry in the directory table via `LIKE`, and creates an account. In the end, 55 test accounts were sitting across seven apps, 42 with a privileged role. None had ever signed in, and email confirmation blocked login, but a SQL injection that can read the `users` table can read confirmation tokens too.

**The first fix.** An early validation fix included `return false if email =~ /[%*'";\-_:]/`. In a Rails `validate` method that does nothing: the return value is ignored, and only added errors count. The suspicious addresses sailed through. We caught it by testing from the outside, and the final fix adds the error properly, escapes `LIKE` wildcards with `sanitize_sql_like`, and parameterizes the query.

A later fleet-wide re-test found one app that our patch hadn't touched, because it didn't contain the file we'd been searching for. It had its own equivalent hole in a different query. "Our search found nothing there" had been written down as "not vulnerable."

**Lesson:** test the exploit, not the diff. Verify every app with the actual probe, including the ones you think are out of scope.

## Gap 5: nobody told the people who would respond

This one is about process, not tooling. The test was legitimate, but the people running the systems didn't know about it. From where we sat it looked like a hostile, hands-on intruder: spoofed "internal" user agents, route enumeration, ID walking, then thousands of injection attempts. We shut down a proxy, preserved forensic copies of logs, built a cleanup tool, and spent days on an investigation that a one-line heads-up could have scoped.

That wasn't wasted. The response drill produced most of the findings above, and it showed how we behave under pressure. But a test with no coordination point has costs: wasted effort, the risk of ops destroying evidence or blocking the testers mid-run, and an outage nobody had agreed to risk.

**Lesson:** agree in advance on who knows, what's in scope, and whether production may be taken down. A blind test can be a deliberate choice, but make it a choice. Even then, give someone the authority to stop the test quickly.

## What we changed

- Shut down the unwatched proxy. If it comes back, it comes back with working cert handling and the other proxy's jails.
- Added the sign-up rate-limit jail, which counts requests whatever the response.
- Moved log retention into the backup job and copied history off-host, with checksums.
- Parameterized the vulnerable queries and added strict email format validation.
- Deleted the test accounts with a tool that dry-runs by default and refuses to delete anything that has ever signed in.
- Started on the longer list: a catch-all HTTPS server block so unknown hostnames don't land on a real app, request timeouts, database statement limits, and logging the real client address on the backends.

## The short version

None of these gaps was exotic. A proxy no one remembered, a jail that watched an empty file, filters tuned to status codes, logs with no home, a search tuned to the attack we expected, and a test nobody told us about. Each looked fine on its own. The testers found the seams between them, which is what we hired them to do.

If you take one thing away, make it this: for every control you rely on, ask what you would see if it had silently stopped working, and make sure something is looking.
