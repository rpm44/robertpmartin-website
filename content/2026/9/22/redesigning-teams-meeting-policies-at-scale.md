---
title: "From Opt-In to Opt-Out: Redesigning Teams Meeting Policies at Scale"
tags:
  - microsoft-teams
  - microsoft-365
  - governance
description: Flipping a tenant's default Teams meeting policy from restrictive-by-default to permissive-by-default — and replacing a manual ticket-and-assign loop with attribute-driven group policy.
---
## The problem with opt-in at scale

Eight near-duplicate Teams meeting policies, each requiring a manual assignment step from our managed-services partner before a user got the features everyone should have by default. Every new hire, every policy tweak, went through a ticket-and-assign loop that existed only because the tenant defaulted to *less* functionality until someone asked for more.

## The redesign

Flip the default. The Global (org-wide) policy becomes fully permissive: Recording, Voice Isolation, and Transcription all on. Restriction becomes the exception, applied via dynamic security groups instead of manual assignment:

- **External/vendor accounts**: gated on an account-type attribute already present in Entra, with a domain exclusion so the partner's own service accounts aren't caught by a filter meant for the vendors they manage.
- **A specific account population**: identified by a region-code attribute lookup, its own restrictive policy.
- **Franchisee/store accounts**: restricted by the same account-type attribute, doubling as a defensive gate in case future licensing changes would otherwise grant them features they shouldn't have.

Even though all three exception policies started identical, they're built as separate policy objects rather than one shared policy applied three ways. That's a deliberate cost: it means writing and maintaining three things instead of one, in exchange for not having to re-architect the moment one population's restriction level needs to diverge from the others. With three different populations, that was a matter of when, not if.

## What this makes obsolete

The existing approval workflow has three stages: ticket, routing, and manual policy assignment. Automation only ever covered the routing. The assignment itself was always a human doing it by hand. With group-based policy already driven by attributes that exist in Entra today, that entire request loop is a candidate for retirement: group membership does the assignment automatically, and the ticket disappears along with the wait.

One piece doesn't disappear on its own: a Terms of Service/consent step, previously implicit in the act of submitting a ticket, still needs a real mechanism once there's no ticket to submit. Entra's native Terms of Use feature, tied into new-hire onboarding, is the leading candidate, pending sign-off from Legal since that's a governance decision, not an engineering one.

## The wrinkle that almost stalled it

Group-based assignment for the Voice Isolation policy type specifically had been reported internally as unsupported. Current Microsoft documentation says otherwise: the exception list for group-assignable policy types has narrowed significantly since 2023, and Voice Isolation isn't on it anymore. Run a live `-WhatIf` test before trusting either your own prior notes or a partner's. This is exactly the kind of platform detail that goes stale silently.