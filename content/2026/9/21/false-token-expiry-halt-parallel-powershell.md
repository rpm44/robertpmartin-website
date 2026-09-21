---
title: "A False 'Token Expired' Halt in 50-Thread PowerShell, and the Statistics That Actually Fixed It"
date: Mon, 21 Sep 2026 00:00:00 +0000
slug: 2026/9/21/false-token-expiry-halt-parallel-powershell
status: publish
tags:
  - powershell
  - microsoft-365
  - microsoft-teams
---

I run a monthly audit that checks Teams Voice Isolation enrollment against policy eligibility for our ~27,500 Teams-licensed users. It calls an internal REST endpoint under a short-lived bearer token, spread across 50 concurrent PowerShell runspace threads to make the per-user calls finish in a reasonable time. This post is about a bug that made the script quit early, convinced its own authentication had died when it hadn't, and the statistical fix that actually held up.

## The symptom: halting on a token that hadn't expired

The script had safety logic to stop early if the bearer token expired mid-run, rather than burn through thousands of calls failing silently. The trigger was simple: too many `401 Unauthorized` responses in a short window, treated as proof the token was dead.

The problem was that it kept firing well before the token should have expired. I decoded the token's own `exp` claim across 12 consecutive restarts using the same token and watched it count down smoothly and continuously: 68.4 minutes remaining, then 63.1, then further down, never resetting, never jumping. The token was fine. The halt logic was reacting to something else.

## First fix attempt: also wrong, for an interesting reason

My first fix was a rolling window: halt only if N 401s land within a fixed time span (5 in 10 seconds, say), rather than any 401 at all. That still produced false halts.

The reason took a minute to see: at 50-thread concurrency, ordinary per-user failures (a bad license state, a stale group membership, whatever the normal ~2% baseline error rate covers) cluster in time purely because the threads themselves are synchronized. Right after a restart, a big batch of requests resolves in a tight window. Five 401s landing within 10 seconds of each other isn't rare noise at that concurrency level; it's what noise looks like when 50 threads finish work at roughly the same moment.

## The fix that held: consecutive streaks, not time windows

The signal that actually separates real token death from ordinary noise isn't *how many* failures happen in a window. It's whether failures are ever interleaved with successes. At a ~2% baseline error rate, real per-user failures are sprinkled among far more successes. A dead token fails *every* subsequent call, with no successes at all breaking up the streak.

So the halt condition became: N consecutive 401s with zero successes interleaved (default 15). At a 2% error rate, the odds of 15 failures in a row with no successes purely from noise are on the order of 10⁻²⁶. A streak that clean doesn't happen by chance. It means the token is dead.

## The concurrency gotcha hiding inside the fix

Implementing a shared streak counter across 50 runspace threads has its own trap. My first pass stored the counter as a value in a shared hashtable and updated it through a `[ref]` to that hashtable entry. It didn't reliably write back. Under concurrent access, `[ref]` against a hashtable *value* doesn't behave like a real reference the way it does against a variable.

The fix was to store the counter as a single-element array (`$streak = ,0`) instead of a bare integer, and update `$streak[0]` directly. An array element is a genuine reference-type slot, so it's safely addressable with `Interlocked`-style increments across concurrent threads in a way a hashtable value isn't. It's a small distinction, but it's the difference between a counter that actually works under load and one that silently drifts.

## The fix uncovered a second, real problem

Once the false halts stopped, the job could finally sustain full 50-thread concurrency continuously instead of getting interrupted and restarted every few minutes. That surfaced something the restarts had been accidentally masking: the error rate jumped from ~2.2% to ~7.6% under sustained load, dominated by connection-level failures with no HTTP status code at all: timeouts and resets, not 401s.

That pattern is consistent with the endpoint doing connection-level throttling under sustained pressure without ever returning a proper `429 Too Many Requests`. I added the same backoff behavior for these connection-level errors that already existed for rate limiting, plus a `-RetryFailed` switch so a resumed run re-queues failed rows instead of treating any prior attempt, success or failure, as permanently done.

## Where the numbers landed

After both fixes, a full run against the ~27,500-user population came back with 391 users eligible for Voice Isolation, 203 enrolled, and a residual error rate of 0.13%, all confirmed per-user rejections, zero connection-level errors left. That's down from the 7.6% the sustained-load run had surfaced, and it's the number I'd trust to hand to someone else.

The broader lesson: a safety heuristic that fires on noise is worse than no heuristic at all, because it looks like it's protecting you right up until you check what it's actually reacting to. Decoding the token's own expiry claim and comparing it against the failure pattern, rather than trusting the failure pattern in isolation, was what turned a plausible-looking fix into a correct one.
