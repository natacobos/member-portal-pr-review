# Review of pull request #418 (MYT-2417)

Write short answers. Plain words are fine.

## T1. Your decision and the blocking problems

Decision (merge as it is / ask for changes / reject): **Ask for changes.** Do not merge before the freeze as it is; it does not fix MYT-2417 and it breaks other things.

Problem 1: **It does not fix the vulnerable output.** `summary.asp` still writes `DisplayName` raw into the `<h1>` and into a JavaScript string, so the 37 rows already stored are still exploitable, and `Server.HTMLEncode` does not encode `'`, so a name like `';alert(1)//` still breaks out of the JS string (doubling the quote does not stop it either). The fix belongs at output in `summary.asp` (HTML-encode in HTML, JS-encode or remove the inline script).

Problem 2: **Encoding before saving corrupts the name for every other reader.** The display name is also read by the API (Angular site and mobile app, which already escape, so users would see `&amp;`) and by the monthly mailing export, where the print vendor prints letters exactly as received ("Dear Mary &amp; Joe," is already in last month's file). Encoding can also push a 100-character name over the 100-character `@DisplayName` parameter.

Problem 3: **The `CleanInput` change is out of scope and damages data on every page that includes it.** All SQL here already uses stored procedures with typed parameters, so there is no SQL injection to fix; doubling quotes would store `O''Brien` in display names and mailing addresses (printed on letters) and break support agents' last-name search. Also: the evidence does not prove anything (the test calls a `Server_HTMLEncode` that does not exist under cscript and only tests the encoder, `tests/` is not in the repo, the SAST scan skipped `summary.asp`, no pipeline run or rescan).

## T2. What else reads or depends on the code or data this PR changes

- `dbo.Members.DisplayName` (the stored data) and `dbo.usp_SaveDisplayName` (writes it)
- `dbo.usp_GetMemberProfile`, called by `web/includes/db.inc` and by the .NET API
- `web/account/summary.asp`: shown to members and to support agents on calls (HTML and inline JS)
- `api/Controllers/MembersController.cs` `GET api/members/{id}/profile`, used by:
  - Angular `client/src/app/profile-card/profile-card.component.ts`
  - the mobile app
- `dbo.usp_StatementMailingExport` → SQL Server Agent job "Statement mailing" (`jobs/monthly_mailing.md`) → CSV to the print vendor → printed letters ("Dear <DisplayName>,")
- `web/includes/validation.inc` `CleanInput`, included by every page that reads a form field, including:
  - `web/profile/save_profile.asp` (display name)
  - `web/address/update_address.asp` → `dbo.usp_SaveAddress` → addresses on the printed letters
  - `web/search/member_search.asp` → `dbo.usp_SearchMembers` (support agent search)
  - any other page including `validation.inc` (search the whole codebase, not just these files)
- The 37 existing rows with `<` or `>`, and rows already HTML-encoded (e.g. `Mary &amp; Joe`, `Ana &lt;3`)
- **An unknown writer that already stores encoded names.** The 2026-10-01 letters file shows `Dear Mary &amp; Joe,` before this PR existed (opened 2026-11-14), and the current code does not encode. So another page, job, import or an earlier SEC-12 fix writes `DisplayName`. Find it before changing how the name is stored.

## T3. Acceptance criteria to close MYT-2417

Security:
- `summary.asp` encodes `DisplayName` for its context everywhere it is output: HTML-encoded in the page body, and JS-safe (or not put in inline script at all; e.g. set `innerText` from an HTML-encoded data attribute) in the script.
- A script payload in a display name (HTML payload like `<script>`/`<img onerror>` and JS-string payload like `';alert(1)//`) does not run on `summary.asp`, including for the 37 already-stored rows.
- Other places that render `DisplayName` are checked and also encode on output (Angular `{{ }}` is fine; no `innerHTML`/`[innerHTML]`).
- Optional defense in depth: length/character validation on input, but no encoding before saving.

Behavior (what must keep working):
- Names with `&`, `'`, `<` (e.g. "Mary & Joe", "Tom O'Brien", "Ana <3") are stored as typed and show correctly on `summary.asp`, the Angular profile card, the mobile app and the printed letters (no `&amp;`, no `''`).
- `CleanInput` is unchanged (revert); addresses and member search work for names with apostrophes.
- A 100-character name still saves.
- Data plan for existing rows: decide what to do with the 37 rows and with values already stored encoded (decode to plain text in a reviewed script, before the next mailing run on the 1st).
- SEC-12 is updated (or an exception recorded with AppSec) to "encode on output", so the next fix does not repeat this.

Evidence (scanner, pipeline, tests):
- Merged through the Azure DevOps pipeline (build and checks green), not hand-deployed.
- New scan by the client's security tool on the deployed site that no longer reports MYT-2417, covering `summary.asp`, attached to the ticket.
- Tests that are in the repo and run in the pipeline and exercise the real page: payload saved then `summary.asp` rendered shows it encoded, plus regression cases for `&`, `'`, `<` names, address save and search.
- A manual retest of the original proof-of-concept on `summary.asp` (member view and support agent view), with screenshots attached.
- A sample mailing export run showing salutations print correctly.

## T4. Your review comment to Luis (as you would post it on the PR)

Hi Luis, thanks for jumping on this fast. I can't approve it as it is, because it doesn't close MYT-2417 and it would break some other things. Requesting changes:

1. **The XSS is in `summary.asp`, not in the save.** It still prints `DisplayName` raw in the `<h1>` and inside the `var displayName = '...'` script, so the 37 rows already in the database still run, and `HTMLEncode` doesn't encode `'`, so `';alert(1)//` still breaks out of the JS string. Please encode at output: `Server.HTMLEncode` in the HTML, and for the script either JS-encode the value or drop the inline script (set the greeting from the already-encoded `<h1>` text or a data attribute).
2. **Please remove the encode-before-save in `save_profile.asp`.** Other things read this name too: the API (Angular site and mobile app, which already escape, so they'd show `&amp;`) and the monthly mailing, where the vendor prints letters exactly as received. This bug is already in production from another source: last month's file (2026-10-01, before this PR) has "Dear Mary &amp; Joe,". So this PR would make it worse, and we need to find what is already writing encoded names. I know SEC-12 says encode before saving; that rule is out of date and I'll take it up with AppSec. The comment "Nothing else reads it" is wrong too, please fix or remove it.
3. **Please revert the `CleanInput` change.** It's shared by every form page, and all our SQL already goes through stored procedures with typed parameters, so there's no injection to fix. Doubling quotes would save `O''Brien` in names and addresses and break agent search. If you think there's a SQL issue, let's open a separate ticket.
4. **Evidence:** the test calls `Server_HTMLEncode`, which doesn't exist under cscript, and only checks the encoder; `tests/` isn't in the repo; the scan didn't include `summary.asp`. To close the ticket we need: tests in the pipeline that save a payload and check `summary.asp` (plus `&`, `'`, `<` names showing correctly), the Azure DevOps pipeline run, and a fresh scan from the client's tool attached to MYT-2417.
5. **Existing data:** let's agree on a plan for the 37 rows and the already-encoded names (like `Mary &amp; Joe`) before the mailing job runs on the 1st.

The real fix is small (mostly `summary.asp` plus reverts), so I think we can still make Friday. Ping me when it's updated and I'll review it right away, or let's pair on it for 30 minutes if that's quicker. If it isn't ready and verified by Friday, it waits for the next release instead of going out half-fixed.
