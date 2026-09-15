# Testing the tool before handing it off

This tool has real logic (matching, tolerance, aggregation) that's easy to get subtly wrong — don't just eyeball the code and call it done. The pattern below caught real bugs during development (a missing `}` that broke the whole script, a duplicate `const` declaration, an aggregation edge case) that would otherwise have surfaced only when the user tried real data.

## Recipe: synthetic data through the real upload path

Using a browser tool (navigate to the local file, then run JS in-page):

```js
function makeFile(rows, name){
  const ws = XLSX.utils.json_to_sheet(rows);
  const wb = XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(wb, ws, 'Sheet1');
  const arr = XLSX.write(wb, {type:'array', bookType:'xlsx'});
  return new File([arr], name, {type:'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'});
}

// build a small base dataset and a counterparty dataset with a deliberate mismatch
const baseRows = [{업체명:'테스트업체', 품목코드:'A001', 품목명:'테스트품목', 입고일자:'2026-08-05', 수량:100, 단가:1000}];
const otherRows = [{품목코드:'A001', 품목명:'테스트품목', 입고일자:'2026-08-05', 수량:105, 단가:1000}]; // +5 mismatch on purpose

function setFiles(inputId, files){
  const dt = new DataTransfer();
  files.forEach(f => dt.items.add(f));
  const el = document.getElementById(inputId);
  el.files = dt.files;
  el.dispatchEvent(new Event('change', {bubbles:true}));
}
setFiles('baseFile', [makeFile(baseRows, 'base.xlsx')]);
setFiles('vendorFiles', [makeFile(otherRows, 'other.xlsx')]);
```

Then fill in any manually-required fields (counterparty name text input when the uploaded file lacks a name column, the period field), click the run button, and assert on the result: does the expected mismatch actually get flagged, with the right computed diff? Check `document.getElementById('...').innerHTML` or `get_page_text` for the actual numbers rather than assuming the code is right because it reads right.

**Test the tolerance/custom-rule logic specifically** — pick at least one case designed to pass (within tolerance) and one designed to fail, since the off-by-one direction of a tolerance check (`>` vs `>=`, or comparing against the base value vs. the submitted value) is a classic silent bug.

**Test the admin/user gating** by directly setting the unlock state rather than clicking through PIN prompts — `prompt()`/`confirm()`/`alert()` are native dialogs that block a scripted browser session waiting for input nothing will supply. Monkeypatch around them for testing:

```js
isAdminUnlocked = () => true;   // or isUserUnlocked = () => true; getUnlockedUserName = () => '테스트';
currentView = 'admin';
applyView();
renderResults();  // if a run already happened
```

This is a legitimate way to exercise both branches of view-gated rendering without needing to actually answer a blocking dialog.

## Known false-positive: blank screenshots

`computer{action:"screenshot"}` can come back blank, or time out with "the page did not finish rendering" / "Claude's window is minimized or hidden," specifically when the preview pane isn't the frontmost/focused window in the desktop app. This has happened even when the DOM is completely correct. Before concluding a screenshot means something is broken:

1. Check `getBoundingClientRect()` and `getComputedStyle()` on the element in question — position, size, `display`, `transform` should all look right if it's actually rendering.
2. Cross-check with `get_page_text()` or `read_page` (accessibility tree) — these don't depend on the pane being visually focused.
3. Check console errors.

Only trust a screenshot as evidence of a real visual bug once you've ruled out the above, or after a successful screenshot from earlier in the same session shows the same kind of content rendering fine (i.e. it's not a systematic renderer issue, just a timing hiccup on that one capture).

## Excel/ZIP export sanity check

After a run, call the export functions directly and confirm they don't throw (they exercise `XLSX.write`, `JSZip.generateAsync`, and any string-sanitization for filenames):

```js
try { downloadExcel(); console.log('excel ok'); } catch(e) { console.log('ERROR', e.message); }
try { await downloadZip(); console.log('zip ok'); } catch(e) { console.log('ERROR', e.message); }
```

Note that in a sandboxed preview context, `localStorage`/`sessionStorage` may be unavailable (throws `SecurityError` under a `data:` URL sandbox) — the template already wraps all storage access in `try/catch` with safe fallbacks, so this shouldn't crash anything, but it does mean persistence itself (mappings/rules surviving a reload) can only be verified once the user opens the real file directly in a normal browser (`file://`), not inside an automated preview tool. Say this explicitly when handing off: "I verified the logic and UI; please confirm mappings/rules persist across a reload on your own machine."
