---
title: "Two Silent Failures I Found Building an M365 License Report"
date: Mon, 21 Sep 2026 00:00:00 +0000
slug: 2026/9/21/two-silent-failures-building-an-m365-license-report
status: publish
tags:
  - powershell
  - microsoft-365
---

I built a couple of PowerShell/Graph reports to break down M365 license assignment by audience and usage: one general-purpose, one scoped to Copilot licensing. Both hit bugs, but the interesting thing about both is that neither one threw an error. Each produced output that looked completely fine until I checked it against reality. Here's what they were and what I changed because of them.

## Failure 1: a BOM silently nulled an entire column

I wanted friendly license names instead of raw SKU part numbers (nobody wants a report that says `ENTERPRISEPACK` instead of the actual product name), so the script fetches Microsoft's published SKU-to-friendly-name CSV at runtime and joins it against the tenant's subscribed SKUs.

The fetch succeeded: 200 OK, confirmed reachable, no exception anywhere. But after parsing, every single row's `Product_Display_Name` came back `$null`. Only 4 SKUs in the whole report got friendly names, and those were the hardcoded fallback values I'd written in for exactly this kind of failure, which is the only reason I noticed at all, rather than shipping a report where almost every license showed up unlabeled.

The cause was a UTF-8 byte-order-mark (BOM) at the start of the file, glued onto the first header name. `ConvertFrom-Csv` doesn't strip it, so the column that should have been named `Product_Display_Name` was actually named `<BOM>Product_Display_Name`, a different string, as far as PowerShell is concerned, than the property name my code was asking for. No error. No warning. Just a property that silently doesn't exist, returning `$null` for every row that reads it.

The fix was two parts: strip the BOM from the fetched content before parsing, and add an explicit check that raises a real, visible error if the parsed result comes back empty of expected data. That second part matters as much as the first. Without it, the *next* subtle encoding or schema change from Microsoft's side would fail exactly the same silent way.

One extra wrinkle from my own environment: `Invoke-WebRequest`'s `.Content` came back as a raw byte array rather than a string, which meant the BOM-stripping fix had to explicitly detect and decode bytes rather than assuming a string was already in hand. Worth checking which type you're actually getting before writing string-manipulation code against it.

## Failure 2: a native Excel PivotTable that Excel silently repaired away

The Copilot-specific version of this report needed a pivot breakdown of license counts by audience, region, and market. The `ImportExcel` module can generate a native Excel PivotTable directly from `Export-Excel -IncludePivotTable`, and on smaller test data it worked fine.

On the real production run (8,477 Copilot-licensed users), the script completed with no errors, and the file it wrote had a pivot table object with a `recordCount` claiming a handful of cached records. But when I opened it in Excel, both pivot tables came back empty, because Excel's own repair-on-open logic had silently stripped them. Excel had detected something malformed in the pivot cache and quietly fixed it by deleting the pivots, without surfacing that to the user unless you went looking at the repair log.

Digging into the file, the pivot cache's `worksheetSource` reference was empty despite the cache metadata claiming records existed: an internal inconsistency in what `ImportExcel` had generated, not a data problem on my end. Since the single-pivot code path and the multi-pivot path in that module version share the same underlying generation code, I couldn't be confident a narrower fix wouldn't hit the same failure mode again on the next larger run.

Rather than debug further into a third-party module's internals, I replaced native `-IncludePivotTable` pivots with computed cross-tab summary sheets, building the audience-by-region and audience-by-market breakdowns as plain formatted tables via a small helper function instead of relying on Excel's own pivot cache generation. More code on my side, but it's a format I can fully control and verify, rather than trusting a code path that had just demonstrated it can silently produce a broken file.

## The pattern behind both

Neither of these failed loudly. Both produced a file that opened, looked plausible, and was wrong. That's a worse failure mode than a thrown exception, because a thrown exception gets noticed. A plausible-looking wrong report gets emailed to someone.

The fix in both cases wasn't just patching the specific bug. It was adding a check that verifies the *shape* of the output, not just whether the code path completed without an exception. A parsed CSV with zero non-null values in a column you expect data in, or a pivot cache with a record count that doesn't match its own worksheet reference, are both detectable before the file ever gets exported. I'd rather the script fail loudly on those checks than let Excel's repair dialog be the first thing that notices.
