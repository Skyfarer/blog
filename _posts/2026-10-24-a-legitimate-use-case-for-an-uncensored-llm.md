---
layout: post
title: A legitimate use case for an uncensored LLM
categories: tech
---

Earlier this month one of my webapps went down and while investigating why, I found a SQL injection attack underway. Turns out it was an authorized pen test by a hired pen tester, and my client forgot to tell me about it. The interesting thing was it was not the kind that allowed an attacker to quickly exfiltrate enormous amounts of data.  It was a weakly written boolean function. The attack was able to inject some SQL and see whether it was valid or not. So they launched an attack to determine the database schema by making guesses, one character at a time. It was slow, and by the time I discovered it all they had was the running MariaDB version and a few obvious table names and column names. No data had been exfiltrated.

The code fix was rather quick and easy. But how did a repetitive attack like this not get blocked by fail2ban? Well, it also exposed a gap in my fail2ban rules which I quickly patched.

## The problem with asking a normal LLM to test this

I had Claude Opus review the code, and the fail2ban rules, and it gave its stamp of approval that everything was good to go. But how do I know for sure? This attack smelled of AI to me because it found a really specific path to exploit. It wasn't anything like the daily HTTP attacks you see on anything with a public IP address like PHP and directory traversal. This was as if someone already had some knowledge of our setup, which is likely since it was a hired pen test. But the thought crossed my mind that an LLM could have found this through trial and error, and it could just hammer along 24/7 and never get bored with it.  The cybersecurity defense posture is really changing, AI will find our blind spots much quicker than humans will.

I decided I would feel more confident in our fixes if I did the same thing. Have an LLM run the attack from the outside and confirm for me that fail2ban is working. Then run it again without fail2ban, is the code really fixed? But Claude flat out refused to do this. It was fine with testing from the inside of the network, but it would not run a test on the public IP addresses from my home IP.


## Renting an attacker

I spun up a GPU on vast.ai — and ran Qwen3.8-27B Heretic, a 4-bit quant of the abliterated build from Hugging Face. "Abliterated" means the refusal directions have been surgically removed from the model's weights, so instead of moralizing at you it just answers. I served it with llama.cpp's `llama-server`, which exposes an OpenAI-compatible API. On my homelab, I started a Proxmox VM and installed Pi agent and pointed it to the vast.ai instance. This way it was sandboxed from my normal dev work, agent skills, my home LAN, etc. Agents have a habit of poking around, which is nice when it's helpful for context, but not in this case.

Uncensored doesn't mean smarter or more dangerous in some mystical sense — it means no built-in hesitation. For authorized testing against infrastructure I own, that's the only property I actually wanted.

## Having it attack from the outside

I gave Qwen the same vantage point the real pen testers had: just the public sign-up form, no source code, no hints about what had changed. I had it go after the same techniques from the original test:

- the classic quote-based injection in the email field, extended into the full boolean-blind routine the bug actually calls for: inject a condition, read success or failure off the response, and step through every character and position until something readable comes out the other end,
- a time-based blind injection, the kind that hides a `sleep()` call inside the query and uses response delay as the oracle instead of true/false, (by the way, this is how I discovered the attack, the sleep calls occupied all the Puma threads and caused the app to go down, this caught my attention)
- and the quieter trick that doesn't use a quote at all — submitting something like `%@example.com` so it rides through on a SQL `LIKE` wildcard instead of breaking the query syntax.

Qwen didn't just hand me one clever string, it wrote the whole grind: a loop that submitted a guess for one character, checked whether the sign-up succeeded or failed, narrowed the guess, and moved to the next position — the same tedious, one-bit-at-a-time process a human attacker would have set an LLM on. I replayed each generated request against the live, patched endpoint. Every attempt failed the way it was supposed to: the parameterized query didn't care what was in the string, so there was no boolean to read anything off of, the wildcard got escaped before it ever reached `LIKE`, and the validation method that used to silently do nothing now actually added an error and rejected the input. The fix held against a model with no reason to pull punches and no instructions to go easy on me.

That's a better signal than a green test suite. The tests only check the cases I thought to write. An uncensored model attacking from outside doesn't know or care what I thought to write.

The real fun came when I had it try one of my webapps that I hadn't patched yet. It was scary how easily it started guessing the database schema. Now of course fail2ban had to be temporarily disabled, it would stop this attack in its tracks if active. But I had the peace of mind that if fail2ban is accidentally left offline, or misconfigured, the next layer of defense is solid. For now at least.

## What's next

I'll keep this tooling handy for external pen testing of networks I own. But this brings up an alarming thought. We are now at a time where anyone can easily run an uncensored model and abuse it. This radically changes our defense posture. But thankfully the LLMs are available to use on the blue team as well, and can be leveraged to build and test our defenses.
