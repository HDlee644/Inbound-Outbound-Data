# Matching & tolerance data model

This is the core algorithm from `assets/template.html`. Adapt the field names and matching key to the new domain, but keep the shape — it's what survived real feedback (see SKILL.md "Hard-won lessons").

## Row shape after parsing

Each row from either dataset normalizes to:

```js
{
  date: normDate(rawDateCell),      // 'YYYY-MM-DD', handles Excel serial dates, JS Date objects, and strings
  vendor: '...',                     // counterparty name — from a mapped column, OR a manually-typed override
                                      // per uploaded file when that file has no counterparty column
  itemCode: '...',
  itemName: '...',                   // optional
  qty: number,
  price: number,
  amount: number|null                // optional; if absent, computed as qty*price downstream
}
```

`normDate` must handle three input shapes because Excel cells vary: a JS `Date` (when `XLSX.read` is called with `cellDates:true`), a raw Excel serial number (fallback via `XLSX.SSF.parse_date_code`), or a plain string. Don't skip the serial-number fallback — real exports mix formats.

## Column mapping (why it exists)

`FIELD_DEFS` is a list of `{key, label, synonyms}` — one entry per logical field (date, vendor, itemCode, itemName, qty, price, amount). `guessMapping(headers)` best-guesses which uploaded column maps to which logical field by checking the synonym list, but the UI always shows this mapping as editable dropdowns before anything runs, and the confirmed mapping is saved to `localStorage` keyed by dataset type (base vs. counterparty) so it's remembered next time the same file shape is uploaded. When adapting this skill to a new domain, extend the synonym lists — don't hardcode a single expected header string anywhere.

## Aggregation and key

```js
function makeKey(r){ return `${r.itemCode}||${r.vendor}||${r.date}`; }

function aggregate(rows){
  const map = new Map();
  for(const r of rows){
    const k = makeKey(r);
    if(!map.has(k)) map.set(k, {...r, qty:0, amount:0, n:0});
    const e = map.get(k);
    e.qty += r.qty;
    e.amount += (r.amount!=null ? r.amount : r.qty*r.price);
    e.n += 1;
  }
  for(const e of map.values()) e.unitPrice = e.n ? e.amount/(e.qty||1) : 0;
  return map;
}
```

Aggregating (summing) rather than requiring one row per key matters: a real dataset can have multiple line items for the same item/counterparty/date (e.g. two deliveries same day), and summing them before comparing is the only sane way to reconcile against a single-line submission on the other side.

**When adapting the matching key**: the key needs to be specific enough that a false match (two genuinely different things treated as the same) doesn't happen, but not so specific that legitimate duplicates fail to combine. Ask the user what uniquely identifies "the same transaction" in their domain if it's not obviously item+counterparty+date.

## Comparing matched rows

For every key present in either side's aggregate map:

- **In both** → compute `qtyDiffPct = pct(vQty-bQty, bQty)` and `amtDiffPct` the same way for amount; if `abs(diffPct) > tolerancePct` for either, it's a fail with a human-readable reason string (`기초 X vs 업체 Y, ±Z%`). `pct(diff, base)` returns `0` if both are zero, `100` if base is zero but diff isn't (avoids divide-by-zero while still flagging a real mismatch).
- **Only on the base side** → automatic fail, reason "업체자료 누락" (counterparty didn't submit this).
- **Only on the counterparty side** → automatic fail, reason "기초자료 누락" (counterparty submitted something not in the master record — equally important to flag, since this is often the more suspicious direction).

## Tolerance and custom rules (admin-configurable, not hardcoded)

Base tolerances are two numbers admins can change at runtime: quantity tolerance % and amount tolerance %. Default to 0 (exact match required) unless the user says otherwise — don't assume a lenient default.

On top of that, an admin-defined custom rule list — `{field, op, value, label}` where `field` is one of `qtyDiffPct, amountDiffPct, baseQty, vendorQty, baseAmount, vendorAmount, baseUnitPrice, vendorUnitPrice` and `op` is a comparison operator — lets admins flag domain-specific conditions (e.g. "any single line over ₩1,000,000 needs manual sign-off") without needing a code change. Evaluate these against every matched (both-sides-present) row, appending a `[커스텀] ...` reason when triggered. This generalizes to whatever the new domain's "also flag this" cases are — keep the custom-rule mechanism rather than hardcoding one-off business rules directly into the matching function, because business rules like this change more often than the matching logic does.

## Per-counterparty rollup

Group matched+unmatched rows by counterparty name, summing base qty/amount and counterparty-submitted qty/amount, and counting pass/fail. This is what renders as the summary table and what becomes the "previous period" input file for month-over-month comparison (see `references/photos-and-export.md` for the export shape — the summary sheet's exact column names are load-bearing, since the *same* tool re-parses its own prior output next period; don't rename those columns casually).
