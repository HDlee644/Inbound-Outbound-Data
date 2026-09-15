# Photo evidence attachment and Excel/ZIP export

## Photos attach to the specific counterparty-file entry

The first version of this had one global "attach evidence photos" area, disconnected from which counterparty or submission it was evidence for. That's wrong — evidence is almost always about one specific submission (e.g. a damage photo tied to what one vendor sent). Fix: each uploaded counterparty-file entry gets its own small photo dropzone, built inline right in that entry's UI card:

```js
function buildVendorPhotoUI(entry){
  // creates a dropzone + grid scoped to this one entry; pushes into entry.photos = [{file, url, caption}]
  // entry.photoGridEl stores a reference so re-renders (add/remove) target the right grid
}
```

`entry.photos` lives only in memory (`URL.createObjectURL`), not `localStorage` — images are too large and volatile for that. Tell the user explicitly: refreshing the page loses attached photos, so attach → run → download should happen in one sitting.

Results rendering groups photos by counterparty (`STATE.vendors.filter(v => v.photos.length)`), each under its own label, not as one flat gallery — mirroring how they were attached.

## Excel workbook shape

Sheets, in this order, each only added if relevant data exists:

1. **업체별요약 (per-counterparty summary)** — base qty/amount, counterparty-submitted qty/amount, diffs, pass/fail counts. **Column names here are load-bearing**: if month-over-month comparison is wanted, the tool re-parses its own previous month's output looking for these exact columns (`업체명`, `총입고수량`, `총입고금액` in the original domain — rename consistently for a new domain, but keep whatever the new names are exactly stable across versions).
2. **통과내역 / 불합격상세** — pass rows plain, fail rows with base/counterparty/diff columns plus a `사유` (reason) column joining all triggered reasons.
3. **전월비교** (only if a previous-period file was supplied) — prior vs. current qty/amount and % change per counterparty.
4. **규칙설정** — a snapshot of the tolerance values and custom rules that were active for this run, for audit purposes (so a later dispute about "why did this pass/fail" has an answer).
5. **증빙사진목록** (only if photos attached) — counterparty label, filename, caption — a manifest, since the actual image bytes go in the ZIP, not the spreadsheet (this codebase's Excel library, SheetJS community edition, doesn't support embedding images in cells; don't try to work around that with a different library mid-build unless the user specifically asks for embedded images, which would mean switching to something like ExcelJS).

## ZIP bundling (Excel + photos together)

Use JSZip (CDN) to bundle the workbook and per-counterparty photo folders into one download when photos exist:

```js
const wb = buildWorkbook();                 // same workbook-building function used for plain Excel export
const wbArray = XLSX.write(wb, {type:'array', bookType:'xlsx'});
const zip = new JSZip();
zip.file(`결과_${period}.xlsx`, wbArray);
const root = zip.folder('증빙사진');
STATE.vendors.filter(v => v.photos.length).forEach(v => {
  const vf = root.folder(labelFor(v).replace(/[\\/:*?"<>|]/g, '_'));  // sanitize for filesystem safety
  v.photos.forEach(p => vf.file(p.file.name, p.file));
});
const blob = await zip.generateAsync({type:'blob'});
// then trigger a normal <a download> click on an object URL
```

Keep a plain "Excel only" download button always available, and only show the ZIP button when at least one photo is attached — don't force people who don't use photos through an extra ZIP step.

## Print / PDF path

A "인쇄 / PDF 저장" button just calls `window.print()`, backed by a `@media print` stylesheet that hides upload/rule-editing UI (`.no-print`) and strips shadows/backgrounds so it prints cleanly for a physical approval workflow. This is cheap to add and was specifically asked for ("상사 결재용" — for manager sign-off) — include it by default for any reconciliation tool that produces an approval-style output.
