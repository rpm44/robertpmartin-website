---
title: "Auditing Mailbox Permissions Across an 80,000-Mailbox Tenant"
date: Mon, 21 Sep 2026 00:00:00 +0000
slug: 2026/9/21/auditing-mailbox-permissions-at-80000-mailbox-scale
status: publish
tags:
  - powershell
  - microsoft-365
  - exchange-online
---

> **Note:** this post lays out the design decisions behind `Get-MailboxUserPermissions.ps1`. I'm publishing the technique now and will follow up with the actual runtime numbers from a full production pass once I've pulled them.

"Who has access to which mailboxes" is a simple question that stops being simple around the tens-of-thousands-of-mailboxes mark. I needed a full-tenant answer (every mailbox, every grantee, every access right) for a tenant with roughly 80,000 mailboxes. Getting that report to actually finish was less about the reporting logic and more about which cmdlets and execution model I used to run it.

## Why the obvious approach doesn't scale

The straightforward version of this script loops every mailbox and calls a permissions cmdlet against each one, one at a time. That works fine on a few hundred mailboxes. At 80,000, two things compound against you:

- The classic Exchange Online management cmdlets run over Remote PowerShell (RPS) sessions, and RPS has real per-command overhead and connection-level throttling built in. It was never designed to be hammered sequentially at that volume.
- Sequential execution means the total runtime is the per-mailbox cost times the mailbox count, with no way to overlap the waiting time of one call with the useful work of another.

Run the naive version against a real 80K-mailbox tenant and you're looking at a job that either takes an impractically long time or gets throttled hard enough that it effectively never finishes cleanly.

## EXO V3: REST cmdlets change the constraint

The EXO V3 module's cmdlets (`Get-EXOMailbox`, `Get-EXOMailboxPermission`, `Get-EXORecipient`, and the rest of the `EXO*` family) talk to Exchange Online over REST instead of Remote PowerShell. That removes the RPS session as the bottleneck and changes the throttling model to one built for higher-volume programmatic access rather than interactive admin sessions. It's the same reason the EXO V3 cmdlets show up throughout the rest of the tooling I maintain for this tenant. Anywhere a script needs to touch a large fraction of the mailbox population, REST-backed cmdlets are the starting point, not the classic ones.

## PowerShell 7 parallelism: overlapping the waiting, not just the running

Even with REST cmdlets, a permissions lookup is still mostly I/O wait: round trips to the service. `ForEach-Object -Parallel` in PowerShell 7 (backed by runspace pools under the hood) lets multiple mailboxes' worth of those round trips be in flight at once instead of one at a time, so the wall-clock time tracks the slowest batch rather than the sum of every individual call.

The tuning knob that matters here is the throttle limit: the number of concurrent runspaces. Too low and you're leaving the REST endpoint's real capacity on the table; too high and you start seeing connection-level throttling that costs more in retries than it saves in concurrency (a pattern I've hit in more than one of these tenant-wide scripts: sustained high concurrency against any M365 REST endpoint eventually finds its throttling ceiling, and the fix is almost always backoff-and-retry logic rather than backing off the concurrency itself).

## What this buys, concretely

Combining EXO V3's REST transport with PS7's parallel execution model is what makes a full 80,000-mailbox permissions sweep a script you can actually run on demand, rather than an overnight job you have to babysit. The report itself (FullAccess, SendAs, and SendOnBehalf grants per mailbox, rolled up with group-trustee membership expanded to real headcounts) is the easy part once the collection layer can actually finish in a reasonable window.
