# Journey fixes: implement every item in JOURNEY-BUGS.md now

Source list: ~/Downloads/JOURNEY-BUGS.md (bugs 1-12, enhancements E1-E6, the to-do). Every item is in scope tonight; nothing is "later". The headings there still say "later"; ignore that. The user is asleep; the supervisor (Claude, watching this window) answers your questions. Ask only when a decision is really the user's; otherwise pick the option that fits the decisions below, write it down and carry on.

## Repos (work on these branches, commit locally, NEVER git push, never merge, no PRs)

- Platform: ~/dev/frontdoor/platform, branch feature/front-door (front door plugin, plugin-acuity-host roles, plugin-security, CLI, acuity/scripts, acuity/cloud, starter, skills).
- Console UI: the acuity-objectstack-ui checkout used by the platform (branch feature/front-door-ux; find it via .objectui-sha / acuity/compat.json and the local cloud scripts). If you change it, commit there and bump the pin in the platform the same way earlier commits did (e.g. ce16fff14).
- D&K: ~/dev/frontdoor/dk, branch feature/front-door-trust. It has an uncommitted pnpm-lock.yaml change that is not ours: leave it out of every commit.
- App used to verify: ~/AcuityLaptop/apps/borrower-watchlist (laptop) and its dev at http://borrower-watchlist.localhost:4400. Update the laptop with acuity/scripts/fde-update.sh borrower-watchlist (log to ~/AcuityLaptop/logs). You may redeploy the local cloud and run pnpm acuity:push for Borrower Watchlist to the LOCAL dev environment to verify fixes; that is not a git push.

One commit per item (or tight group), message starting with the item id, e.g. "bug 9: ...", "E5: ...". Each commit has tests. Run the touched packages' test suites before committing; never commit with a failing test you introduced.

## Rules

- No browser automation (no Playwright/Puppeteer). The supervisor checks screens by hand; end with a browser checklist per item.
- Never print secrets, tokens or passwords. No admin or shared service keys on the laptop (only its own session, its own D&K key, a Windmill token scoped to its own workspace).
- Never change source data. Never modify the Juniper Mills record.
- Business approvals in tests are seeded test personas only; don't approve Borrower Watchlist's real pending requests on behalf of Grace or Sam.
- Authorization changes need a human security review before merge: list every such commit in the final report under "Needs security review".
- Don't build things nobody asked for. Fix the cause in the platform, not in apps.

## Order

1. bug 9 (security: `records: 'all'` can delete). Fix in the role generator; tests per record level; check own / own_readonly / all_readonly; a direct API delete is refused.
2. bugs 6, 7, 8, 10 (roles layer).
   - 6: field rules on roles (`fields: { <object>: { <field>: { readable, editable } } }`), generated into the role's permission set, schema-validated, tested (auditor API read omits the field, also in exports/list views), documented in the app skill and AUTHORING.md. Then the escape hatch (role may link a hand-written permission set; lint allows only that link; acuity:check flags it for review).
   - 7: record conditions for edit and for starting workflows (`edit: { records: 'own', when: "status = 'Open'" }`, `starts: { escalation: { records: 'own', when: ... } }`), enforced on the server at workflow start; tests for both refusals.
   - 8: check every row of the table against roles.ts and the generator; implement each confirmed gap in the role format (mixed access per role, view/tab visibility per role, state-based edit, role inheritance, data scope per role, partial masking, agent read vs write, temporary access, exports/reports/API keys honouring field rules) using ObjectStack's own permission sets, sharing, view rules and masking. Write a short table of what each row became, with tests.
   - 10: auditors can read workflow/approval history read-only (a role grant), and a workflow step can stamp the acting user and comment on the record. Tests.
3. bug 11 (option 1, decided): the CLI acts as the builder through their broker session (drop the separate laptop-cli account), and the front door's token for a builder on THEIR laptop environment carries "data access approved", which the laptop app turns into `<app>_source_reader` for that session only; never in dev. Fix the skill text and connect message that point at Setup > Users.
4. bug 12: on a laptop workspace, `acuity:assets apply` (or `pnpm pipeline create`) creates scripts and flows that are in the repo but not in the workspace and adds them to the manifest; refuses dev/admin workspaces; skills updated so Cursor creates, tests and exports pipelines with no manual step. Then remove Borrower Watchlist's stopgap scripts/create-pipeline.mjs (laptop app commit, not pushed to dev unless verifying).
5. bug 3 + the to-do: the front door names the platform commit; `--connect` checks the laptop's platform copy out at that commit; the front door serves the onboarding script (`curl -fsSL http://localhost:4400/api/v1/frontdoor/onboard.sh | sh -s -- --code <code>`), and the Build locally dialog shows that one command.
6. bugs 1, 2, 4, 5 (front door tile and Build locally dialog): Open goes to the dev app; Owner shows the owner's name; the x closes the dialog (also Escape); an expired code is noticed without reload and a fresh one offered.
7. E3: after onboarding, when the laptop app passes its health check, open Cursor on the app folder and the app in the browser; skip gracefully without `cursor` on PATH; `--no-open` flag.
8. E2 + E5 + E6 + E1 (data access, decided):
   - E2: New app turns D&K on from the start (registered app, own key). The app's D&K tab is useful before approval: the canonical datasets it may request (what each contains), its access requests and status, its mappings and status, what to do next.
   - E5: only a builder requests data access, from that tab, with a confirmation; the server refuses `pnpm acuity:access request` (CLI) for raising requests (status/explain stay). The approved request is the record of what the app may read; roles may only read approved datasets and acuity:check fails otherwise. After approval Cursor proposes mappings; the builder reviews and submits them; the steward approves. Dev inherits the laptop's approved grant AND approved mappings at push: no second request or review; history shows dev received them and from which approval; a push needing an unapproved dataset/purpose or an unapproved mapping stops with a clear message; revocation covers laptop and dev together.
   - E6: the grant is per app + dataset + purpose, without reader roles; the approval hash no longer includes roles' reads; per-person suspension stays. Check whether D&K uses reader positions at read time for agent/pipeline calls; if so switch those to the app's identity.
   - E1: one request can list several datasets; the governor approves or refuses each; grants/subscriptions only for approved ones; the builder sees refusals and reasons.
9. E4: the app skills cover the whole journey so prompts can be business-only (roles incl. field rules and conditions, D&K access via the screen, mappings, test personas, pipelines and agents, checks and tests, never run acuity:push, no browser automation, don't change source data). Put the standing rules in the starter's agent rules. Don't edit the runbook (the supervisor owns it).

## After each item

Re-run what the journey needs (laptop app, dev app, Borrower Watchlist tests via fde-update when the platform changed). At the end, Borrower Watchlist should pass its own tests with no failures, including the credit-manager delete test and the trend test, without workarounds.

## Final report (in this chat)

Per item: what changed, commits (repo + sha), tests added, how verified, browser checklist. Then: "Needs security review" list; anything you could not finish and exactly why.
