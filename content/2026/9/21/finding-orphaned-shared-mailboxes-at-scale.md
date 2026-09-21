---
title: "Finding Orphaned Shared Mailboxes at Scale: What My Precision Numbers Actually Taught Me"
date: Mon, 21 Sep 2026 00:00:00 +0000
slug: 2026/9/21/finding-orphaned-shared-mailboxes-at-scale
status: publish
tags:
  - powershell
  - microsoft-365
  - exchange-online
---

Every large Exchange Online tenant collects mailboxes nobody remembers creating. In our case, one flavor of these is what I'll call "underscore mailboxes": shared mailboxes created years ago as mass-mailer accounts, named to mimic a real user's dotted address but with underscores instead of dots (`tyler_gamble@domain.com` standing in for a real person, `tyler.gamble@domain.com`). They fell out of use once we moved to a real mass-mailing platform, but nobody ever went back and cleaned them up.

I built `Find-UnderscoreMailboxes.ps1` to find the rest of them and score each one for decommissioning. The interesting part isn't the script. It's how wrong my first two attempts were, and what the actual precision numbers looked like once I started measuring instead of assuming.

## The founding case had a name collision hiding in it

The mailbox that started this whole project, `tyler_gamble@domain.com`, looked like it belonged to a real person named Tyler Gamble. It didn't. There's a completely unrelated `tyler.gamble@` account elsewhere in the tenant: a different person, a different account type, created four years apart. Pure coincidence of a common name.

That's the trap with any "does this look like a real person's address" heuristic: name collisions are common enough at scale that you can't trust a match without checking. So the first design decision was to make the dot-equivalent lookup tenant-wide and unrestricted by domain, deliberately surfacing cases like this as data to review rather than auto-trusting a match as proof of the mailbox's real owner.

## Version 1: too slow to ever finish

The first version pulled the full tenant recipient directory live on every run and checked activity through the mailbox's own login timestamps. Two things killed it:

- A full recipient pull against our real domain count (89 accepted domains, tens of thousands of mailboxes) took over 18 minutes just for the directory piece, a single-run cost that made the whole approach a non-starter for anything beyond a one-off.
- Mailbox login timestamps are noisy. Service principals and background jobs log into shared mailboxes and update `LastLogonTime` without a human ever touching them. That made "looks inactive" unreliable as a signal on its own.

I fixed the first problem by caching the recipient/user directory instead of pulling it live every run, and the second by switching the activity signal to the Unified Audit Log, filtered to actions with real actor attribution (`Send`, `SendAs`, `MailItemsAccessed`) instead of mailbox-level login stats. A service principal reading a mailbox doesn't show up as a human sending mail from it.

## Version 2: measuring precision instead of assuming it

Once the pipeline worked end to end, I split it into two phases: a cheap Phase A that only uses signals already in memory (name-shape match, dot-equivalent lookup) to assign each candidate a confidence tier, and a more expensive Phase B (the audit log query, permission footprint, group-membership expansion) that only runs on the tier worth spending that budget on.

I hand-reviewed a 50-mailbox test batch against ground truth to see how the tiers actually performed:

- **High confidence** (name shape *and* dot-match both hit): 7 candidates, **100% real**. Every single one was an actual orphaned mailbox.
- **Medium confidence** (only one signal hit): 43 candidates, **44% real**. The other 56% turned out to be legitimate service or purpose mailboxes whose two-word local part happened to fit the same name-shaped regex (task-force distribution accounts, notification mailboxes, and similar).

That's a real, measured 100%-to-44% precision drop between two confidence tiers built from almost the same signals: not a guess, a number from hand-reviewing every result. It told me exactly where automation could act unsupervised and where it still needs a human in the loop.

## The bug that almost hid the founding case itself

The most humbling find came near the end: when I ran the finished tool, the mailbox that started the entire project, the one I named it after, didn't show up in the output at all. Not low-confidence. Not flagged. Just silently missing.

The name-shape matcher required each underscore-separated segment to start with a capital letter, and the real mailbox's local part was all lowercase. It failed the shape check, and there was no live dotted-address account left in the tenant to rescue it through the other signal either. It fell through both detection paths and never appeared in the candidate list at any tier.

The fix was a one-line change (drop the capitalization requirement, keep the rest of the shape check), but it only surfaced because I went back and specifically asked "why isn't my own example in the results?" It's a reminder that testing a detection tool against a curated, well-formed sample isn't the same as testing it against the actual message that made you build the tool in the first place.

## Where it stands

The tool now runs discovery and cheap triage across the full tenant, writes a triage CSV immediately (so a run that gets interrupted still leaves something usable), and only spends the expensive audit-log and permission calls on the confidence tier worth it. The 100%/44% split above is exactly the kind of number I want more of before I'd ever consider letting anything like this act without a human reviewing the middle tier first.
