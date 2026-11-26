# Journey bugs: Borrower Watchlist run (2026-11-26)

Found while running JOURNEY-RUNBOOK.md. Logged only; none fixed yet.

## 1. Tile "Open" button goes to a broken link

- **Where:** front door, Apps → My apps, Borrower Watchlist tile (status Ready), **Open** button.
- **What happens:** the browser opens `http://localhost:4400/api/v1/frontdoor/apps//open` and shows `{"success":false,"error":{"code":"ENDPOINT_NOT_FOUND","message":"Not found"}}`.
- **Expected:** the dev app opens at `http://borrower-watchlist.localhost:4400`.
- **Likely cause (unverified):** the app slug is empty in the link (`apps//open`): the tile's action builds the URL from a field that this record doesn't fill in.
- **Workaround:** open `http://borrower-watchlist.localhost:4400` directly.

## 2. Tile shows no owner name (to confirm)

- **Where:** same tile.
- **What happens:** the "Owner" label has no value under it, and the next line is `6d5710ceefb4`, which matches the deployed commit (`Live at ... commit 6d5710ce...`).
- **Expected:** Fiona Walsh as owner (runbook step 2 check).
- **Possibly related:** the earlier finding that an owner field showed a raw user id; and bug 1, if the tile record is missing fields.

## 3. Build locally command fails: `Cannot find module .../AcuityLaptop/platform/acuity/scripts/connect.mjs`

- **Where:** step 3, running the command from the Build locally dialog: `OS_ACUITY_HOME=~/AcuityLaptop sh acuity/scripts/fde-onboard.sh --connect http://localhost:4400 --code <code>`.
- **What happens:** the script prints `Updating /Users/jasdeep/AcuityLaptop/platform (acuity/platform)`, then node fails with MODULE_NOT_FOUND for `connect.mjs`.
- **Cause (checked on the work Mac):**
  - `fde-onboard.sh` line 29: `REF="${OS_ACUITY_PLATFORM_REF:-acuity/platform}"`. With no override it updates the laptop's platform copy (`~/AcuityLaptop/platform`) to `acuity/platform` (now at cb6d667d9, PR #20).
  - `connect.mjs` exists only on `feature/front-door` (local and origin both at 816d3c018f6c). `git cat-file` confirms it is not in `origin/acuity/platform`.
  - Line 128 then runs `node acuity/scripts/connect.mjs` inside that `acuity/platform` copy, so the file isn't there.
- **Workaround:** set the branch and take it from the platform checkout that runs the cloud:
  `OS_ACUITY_PLATFORM_REF=feature/front-door OS_ACUITY_PLATFORM_SOURCE=~/dev/frontdoor/platform OS_ACUITY_HOME=~/AcuityLaptop sh acuity/scripts/fde-onboard.sh --connect http://localhost:4400 --code <new code>`
- **Fix to make:** the front door should decide the platform version, not a default in the script. For example, `--connect` asks the front door which platform branch and commit it runs and checks the laptop copy out at that commit. Until `feature/front-door` is merged into `acuity/platform`, any default branch is wrong.

## 4. Build locally dialog: the close (×) button does nothing

- **Where:** front door, Borrower Watchlist tile → **Build locally** dialog, the × at the top right.
- **What happens:** clicking × doesn't close the dialog (reported by the user).
- **Expected:** the dialog closes, the same way **I have copied the command** does.
- **Not yet checked:** whether Escape or clicking outside closes it, and whether the console logs an error.

## 5. Build locally doesn't notice when its code expires

- **Where:** front door, Borrower Watchlist tile → **Build locally**.
- **What happens:** after the 10 minutes run out, the dialog and tile still show the old, expired code and command. You only get a fresh code after reloading the page (reported by the user).
- **Expected:** at expiry the dialog says the code has expired and offers a new one, or Build locally issues a fresh code when it's opened again, with no page reload.
- **Not yet checked:** whether the old code is cached in the page or the server returns the same expired code until reload.

## 6. Cursor says the platform can't hide one field from one role (it can)

- **Where:** Fiona's Cursor, prompt 1 (Borrower Watchlist). Its first question was: "The platform can't hide one field from one role. How should internal notes be kept from auditors?" Its options were a separate notes list, or a platform request with auditors blocked until then.
- **Why that's wrong (checked in the platform code):**
  - field-level security exists: a permission set's field permissions take `readable: false` (`packages/spec/src/security/permission.zod.ts`, `FieldPermissionSchema`);
  - it's enforced on the server: `packages/plugins/plugin-security/src/security-plugin.ts`, step "4. Field-level security", masks restricted fields in the results of find, findOne, insert and update; field writes are refused upstream;
  - `security-plugin.test.ts` covers it (for example `'task.ssn': { readable: false }`).
- **Likely cause:** the app's skills (`.cursor/skills`) say nothing about field-level security, so the agent guessed. Grep of `.cursor`, AUTHORING.md and README for field permissions found nothing.
- **Update:** Cursor later narrowed it to "Hiding internal notes from auditors has no supported path **in an Acuity app**", and offered a platform request or "a documented contract exception: hand-written auditor ...". So the gap may be in the Acuity app layer: how business roles are declared and turned into permission sets may not expose field permissions, even though the core platform enforces them. **Not verified yet:** read Cursor's "details above" and check how Acuity app roles become permission sets.
- **Confirmed cause (Cursor's details, then git history):** `acuity.config.ts` roles are turned into positions and permission sets by `plugin-acuity-host/src/roles.ts`. The role schema there (lines 27-39) is strict and has no slot for field rules, so the generated permission set gets object rights only. The core platform's field-level security is never reached. This role layer was added on 2026-11-22 (commit 087cd6284, "Declare app roles and data needs instead of hand-made permissions"). The starter (`acuity/starter/acuity.config.ts`) dates from 2026-10-02.
- **Origin:** the roles block came from our permissions streamlining work (`permissions-streamlining-proposal.md`, 22 Nov 2026, after the Credit Line Renewal run). There, one business role had to be set up by hand in 5 places: position, permission set and its link, `allowedPositions` in the config, the D&K grant's typed reader positions, and policy-text switches. The goal was "one authority, one declaration, one approval". Field-level rules were never in scope: the proposal's role shape has `records` (own / all_readonly), `reads`, `policyText`, `startsAgents` and `decides` only. Its "kept" list covers object and row rules, not field rules. So this is an omission, not a deliberate limit. The generator plus the lint against hand-made permission sets removed the old hand-written route without carrying field rules over.
- **Decision: keep the abstraction, don't remove it.** Removing it would break or move to hand work:
  - generated positions and permission sets, including the owner defaults (`roles.ts`, `owner-roles.ts`);
  - the single D&K access request and grant reader positions built from `reads`, `policyText` and `dataNeeds` (`access-cli.ts`, `data-access.ts`, `data-access-plugin.ts`, `resources-cli.ts`, `assets-cli.ts`, scripts `data-knowledge.mjs`, `onboard.mjs`, `update-app.mjs`);
  - `startsAgents` and `decides` wiring (`config.ts`, `create-app.mjs`, `update-app.mjs`, Setup overview, CLI `dev` / `run` / `doctor`);
  - the lint against hand-made permissions (`lint/src/validate-retired-permission-residue.ts`);
  - which services load (`features`);
  - Cursor rules and skills (`acuity-app.mdc`, `acuity-sources.mdc`, AUTHORING.md, assessment checks).
- **Fix to make:**
  1. **Field rules on roles (do first):** let a role declare field permissions, for example `auditor: { label: 'Auditor', fields: { entry: { internal_notes: { readable: false } } } }`, and generate them into that role's permission set. Add schema validation (unknown object or field refused), a test that an auditor's API read leaves out the field, and the example in the app-building skill and AUTHORING.md.
  2. **Escape hatch (later, as insurance):** a role may name a hand-written ObjectStack permission set that adds to its generated one. The lint allows it only through that link and flags it for review in `acuity:check` output.
- **Then:** Borrower Watchlist's auditor uses option 1 instead of whatever workaround was chosen in Cursor. Ties in with E4.

## 7. Roles can't put conditions on records: "edit only own while Open", "only the owner may escalate"

- **Where:** Fiona's Cursor, prompt 1. Its question 2: "The platform can't limit starting an escalation to the entry's owner. Until it can, is it acceptable that any analyst can escalate any Open entry (the run records who started it)?" Cursor's requirements assessment marks it as needing a platform change.
- **What's missing:** an Acuity role can say *which* records (own / all) but not *under what condition*, and starting a workflow is granted per role only. So "analysts may escalate" means any analyst can escalate any entry. The same gap likely blocks "analyst edits own entries only while status is Open" (to confirm).
- **Different from bug 6:** bug 6 is about data (which fields a role may see or edit). This one is about actions (who may change or start something, on which records).
- **Fix to make:** record conditions on roles, used both for edits and for starting workflows, for example:

  ```ts
  analyst: {
    records: { entry: { read: 'all', create: true, edit: { records: 'own', when: "status = 'Open'" } } },
    starts: { escalation: { records: 'own', when: "status = 'Open'" } },
  }
  ```

  Generated into record rules, and checked on the server when a workflow run starts (the same way record rules are), so a direct API call can't get around it. Add tests: an analyst refused when escalating someone else's entry, and when editing their own entry after it has left Open.
- **Workaround until then:** model escalation as a change to the entry (status Open → Escalated, with the justification) that triggers the workflow. The analyst's own-record edit rule then blocks escalating someone else's entry. This still depends on "own" plus "while Open" being expressible.

## 8. Other likely gaps in the roles layer: check them against the code

The permissions streamlining (see bug 6, origin) designed roles around `records` (own / all_readonly), `reads`, `policyText`, `startsAgents` and `decides`. These are needs a role probably can't express yet. **None are verified.** First step: read the role schema (`plugin-acuity-host/src/roles.ts`, lines 27-39) and the generator (around line 500), and mark each one as supported or a gap.

| Likely gap | Example | Hits Borrower Watchlist? |
| --- | --- | --- |
| Mixed record access per role | "Sees all, creates, edits only own" (presets may bundle all-or-own) | Yes, the analyst |
| Which roles see a view, tab or dashboard | "High risk view: credit managers and auditors only" | Yes, prompt 4 |
| State-based edit rights | "Edit only while Open" | Yes (bug 7) |
| Record conditions on actions | "Only the owner may escalate" | Yes (bug 7) |
| Data scope inside a role | "Analysts see only their region or portfolio"; D&K row scopes were dropped by default | Not yet; needed for a real bank rollout |
| Partial masking instead of hiding | Show only the last 4 digits; ObjectStack has masking rules | No |
| Roles that include other roles | Credit manager can do everything an analyst can | Duplicated by hand today |
| What an agent may do for a person | `startsAgents` is yes/no; no read-only vs write | Later |
| Temporary or emergency access | Cover while someone is away; support read access to fix a problem | Later |
| Exports, reports and API keys | Hidden data must not leak through an export, report or API key | Ties in with bug 6 |

For each confirmed gap: add it to the role format (generated into ObjectStack's own permission sets, sharing or view rules, as with bug 6), add a test, and document it in the app-building skill.

## 9. SECURITY: a role with `records: 'all'` can delete records, even though delete is meant to be off

- **Severity:** high. It affects every Acuity app that gives any role `all` record access, not just Borrower Watchlist. Earlier apps (Covenant Desk's credit officer, Credit Review and others) probably had the same hole; to check.
- **Found by:** Fiona's Cursor on Borrower Watchlist (requirements R14 and R18, "nobody can delete"); its failing test shows a credit manager can delete an entry.
- **Cause (checked in the code):**
  - `packages/plugins/plugin-acuity-host/src/roles.ts` line 90: `all: { allowRead: true, allowCreate: true, allowEdit: true, allowDelete: false, viewAllRecords: true, modifyAllRecords: true }`;
  - `packages/plugins/plugin-security/src/permission-evaluator.ts`: `modifyAllRecords` is the super-user write bypass. On its own it covers edit, delete and transfer (around lines 56 and 245-256, and line 289: a write with `modifyAllRecords` returns org-wide access). So `allowDelete: false` never gets a say.
  - This is ObjectStack working as designed, similar to Salesforce's "Modify All". The bug is our role generator using a super-user bit for ordinary "edit everything".
- **Fix to make (in the role generator, not in apps):**
  - stop setting `modifyAllRecords` for `all`. Give "read and edit every record" through record sharing instead (`viewAllRecords` for reading, plus an org-wide or role-wide edit grant that respects `allowDelete`);
  - if ObjectStack has no edit-all without delete, add an explicit deny for delete that even the bypass honours (as it already does for `restore` / `purge`, the `DESTRUCTIVE_OPERATIONS` set);
  - add a test per record level: `all` can edit every record but can't delete; `own` can't delete; a direct API delete is refused;
  - check the other presets (`own`, `own_readonly`, `all_readonly`) for the same kind of surprise.
- **Answered in Cursor:** "record it as a platform request, keep the failing test" (option A), with a note giving this cause.

## 10. Auditors can't see who escalated, who decided, or the send-back comment (reported by Cursor, unverified)

- **Where:** Fiona's Cursor on Borrower Watchlist. "The trail on the entry can hold the justification, the decision and when, but not who escalated, who decided or the send-back comment."
- **What's missing:** an auditor role can't read the workflow or approval history read-only, and a workflow step apparently can't write the acting person or the comment onto the entry. So the audit trail auditors see is incomplete.
- **Answered in Cursor:** "both": build the partial trail on the entry now, and raise a platform request for a read-only approval history for auditors (option C).
- **To check:** whether a role can be granted read-only access to approval and workflow run history (probably another roles-layer gap, see bug 8), and whether a workflow step can stamp the acting user and the comment on the record.

## 11. After Grace approves, the builder still can't read the data on the laptop

- **Seen:** prompt 2 stopped. `pnpm acuity:access status` showed the request (raised by Fiona Walsh) approved, the grant written and the `cre_financials` subscription active, but `pnpm acuity:resources --need cdm --sample` said NOT READY. Cursor's CLI account `laptop-cli@acuity.test` doesn't hold `borrower_watchlist_source_reader`, the builder reader position the grant names. Fiona doesn't hold it either (the laptop app's D&K page told her "You hold no reader position").
- **Two problems:**
  1. **Two identities on the laptop.** Front-door calls (request, push) use Fiona's broker session, but reads in the laptop app run as a separate local account, `laptop-cli@acuity.test` (probably left over from before the front door; code path not checked). Reads don't trace to Fiona.
  2. **Builder access needs a hand assignment.** Someone has to give the reader position in the laptop app's Setup → Users after every approval.
- **Fix (decided 2026-11-27, option 1: derive it from the token):**
  - the CLI acts as Fiona through her broker session; drop the separate local CLI account;
  - when a builder signs in to their laptop environment, the front door's token says both "builder of this app" (already there) and "this environment's data access is approved";
  - the laptop app treats such a builder as holding `<app>_source_reader` for that session. Nothing is stored locally, so the host stays the only authority;
  - revocation: the next sign-in or session refresh drops it (12 hours at most), and D&K refuses reads as soon as its grant is gone;
  - dev: builders get no reader position there, so the token never adds it.
- **First step:** check how the laptop app maps front-door token claims to positions today, and where `laptop-cli@acuity.test` is created.
- **The workaround Cursor asked for is refused (2026-11-27):** in the laptop app's Setup → Users → FDE (`laptop-cli@acuity.test`) → Assign position, picking "Borrower Watchlist source reader" fails with: "Only the business roles of Borrower Watchlist are assigned here (analyst, credit_manager, auditor); borrower_watchlist_source_reader is not one of them. Platform and Data & Knowledge positions are given on the front door." But `laptop-cli` is a local laptop account that doesn't exist on the front door. So the skill (`acuity-workflow/references/data-knowledge.md`, AUTHORING.md:95) and the Setup rule contradict each other.
- **Why the position was missing:** Build locally's `connect.mjs` (line 216) calls `enrollFde(..., { sourceReader: true })` (`onboard.mjs` line 225), which gives the CLI account `<app>_source_reader`, but only if that position already exists. At onboarding the starter declares no roles, so the position didn't exist and was skipped. Its own message says "comes once roles are declared (connect again, or Setup > Users)", and the Setup route is now closed by `owner-roles.ts` line 111 (non-admins can only assign business roles).
- **Workaround used (2026-11-27):** reran the same `enrollFde` step as the CLI account (the laptop copy's seeded admin, so `owner-roles` lets it through), from the app folder, with credentials read from `.env.local` and not printed:
  `node --env-file=.env.local -e "import('<platform>/acuity/scripts/onboard.mjs').then(m=>m.enrollFde('http://127.0.0.1:4500','borrower_watchlist',{email:process.env.OS_ACUITY_CLI_EMAIL,password:process.env.OS_ACUITY_CLI_PASSWORD},{sourceReader:true}))"` → `reader: true`.
- **Interim fix if option 1 waits:** have `acuity:access status` (or the laptop app on startup) run this `enrollFde` reader step once the approval is applied, and fix the skill and the connect message to stop pointing at Setup → Users.

## 12. BLOCKER: Cursor can't create a new pipeline; someone has to make it by hand in the Pipelines editor

- **Seen (prompt 3, 2026-11-27):** Cursor wrote `pipelines/coverage_trend.ts` and stopped: "The app's own commands can't create a new pipeline in the laptop workspace". It asked Fiona to create `f/borrower_watchlist/coverage_trend` in the laptop's Pipelines editor, paste the code, test-run and save, and only then run `pnpm acuity:assets export`.
- **Cause:** `pnpm acuity:assets export` only pulls pipelines that already exist in the workspace into `services/`; `pnpm acuity:assets apply` only pushes files in the export's manifest and refuses anything added by hand ("Export again rather than editing exported files"). That suits building in the editor first, not building in Cursor first. The laptop already holds a builder Windmill token for its own workspace (what the Pipelines editor uses), so access isn't the problem.
- **Earlier attempt:** Cursor wrote a helper in `/tmp` that would call Windmill directly with that token; I stopped it because it was ad hoc and outside the repo. It never ran.
- **Workaround used (2026-11-27):** Cursor wrote `scripts/create-pipeline.mjs` in the app repo. It creates one script path in the workspace named in `.env.local` (`borrower_watchlist_fiona_main`) with that file's builder token (never printed), and refuses admin workspaces. Then `pnpm acuity:assets export` pulled it into `services/` (no tokens or connection strings there). All six test inputs gave the expected trend. Remove the helper once the fix lands.
- **Fix:** on a laptop workspace, `acuity:assets apply` (or a `pnpm pipeline create`) also creates scripts and flows that exist in the repo but not in the workspace, and adds them to the manifest. Dev stays push-only through `pnpm acuity:push` and the factory. Update the skills so Cursor creates, tests and exports pipelines without a manual step. Add tests: a new pipeline in the repo appears in the laptop workspace and nowhere else; apply still refuses dev and admin workspaces.

# Enhancements (later)

## E1. Data access requests: several datasets per request, decided one by one

- **Today (unverified):** `pnpm acuity:access request` sends what the app declares. We haven't checked whether one request can cover several canonical datasets, or whether the governor can approve some datasets and refuse others.
- **Wanted:**
  - one request can list several datasets (for example CRE financials and rent rolls), each with its purpose and reader roles;
  - the governor approves or refuses each dataset separately, and D&K writes grants and subscriptions only for the approved ones;
  - the FDE sees which datasets were refused and why.
- **First step:** check the D&K request and approval code (`feature/front-door-trust`) for what it already supports.

## E2. Data & Knowledge on by default for new apps, with a useful default tab

- **Today:** a new app from New app has Pipelines and Agents but no Data & Knowledge tab. It appears only after Cursor turns D&K on (runbook prompt 2).
- **Wanted:**
  - New app turns D&K on from the start: registers the app with D&K and gives it its own key. Access to data still needs the request, Grace's approval and Sam's mapping approval, as today;
  - the D&K tab has something useful before any data is approved, since a new app has nothing in it yet. To decide; ideas:
    - the canonical datasets the app could request, with what each one contains;
    - the app's access requests and their status (waiting for Grace, approved, refused);
    - its mappings and their review status (waiting for Sam, approved);
    - a short "what to do next" that points the FDE at `pnpm acuity:access request` and `pnpm mappings propose`.
- **Then:** drop "Turn on Data & Knowledge" from runbook prompt 2.

## E3. After Build locally's command finishes, open Cursor and the app

- **Today:** `fde-onboard.sh --connect` clones the app, starts `pnpm dev` and stops there. Fiona opens the app folder in Cursor herself and types the laptop app's address into the browser.
- **Wanted:** once the laptop app answers its health check, the script opens:
  - Cursor on the app folder (`~/AcuityLaptop/apps/<slug>`);
  - the laptop app (`http://127.0.0.1:<port>`) in the default browser, ready for "Continue with Acuity".
- **Details:**
  - only after the app is up, not while it's still starting;
  - if Cursor isn't installed or its `cursor` command isn't on the PATH, print the folder and carry on;
  - add a flag such as `--no-open` for headless or scripted runs.

## E4. Cursor prompts in business words only; the technical steps live in the skills

- **Today:** the runbook's Cursor prompts (JOURNEY-RUNBOOK.md section 9) name commands and mechanics: `pnpm acuity:check`, `pnpm acuity:access request`, `pnpm mappings propose`, `pnpm acuity:persona add`, `pnpm workflow push`, "object, field and record rules".
- **Wanted:**
  - FDE prompts describe only the business need: what the app does, who does what, which data it needs, what to check;
  - Cursor finds the how in the app's skills, which already form a fairly complete framework (the skill list on the work Mac includes `acuity-app`, `acuity-workflow`, `acuity-from-brief`, `acuity-acceptance`, `acuity-assessment`);
  - the rules too (never push, no browser automation, don't change source data or the Juniper Mills record) belong in the skills or the app's agent rules, not in every prompt.
- **To do:**
  - go through the existing skills and check each step of the journey is covered: building with roles, D&K access and mappings, test personas, checks and tests, pushing;
  - add whatever a skill is missing, rather than putting it in a prompt;
  - rewrite the four prompts in business words and confirm Cursor still does the same work from them.
- **Note:** "Don't run the push yourself, I will" can stay in the prompt if the skills don't already say it, because it's a decision about who does what.

## E5. Data access is requested by the FDE on screen; Cursor only proposes mappings after approval

- **Today:** Cursor works out which dataset the app needs (from the prompt, `pnpm acuity:resources` and the skills), writes it into `acuity.config.ts` and runs `pnpm acuity:access request` under Fiona's laptop session. Whether it asks first is up to the model (in this run it asked; question 3). With business-only prompts (E4) it would have to match business words to the catalogue itself, which is untested.
- **Wanted (decided 2026-11-26):**
  1. The D&K tab is there from the start (E2) and lists the canonical datasets the app could request.
  2. Only a builder of the app (the FDE) requests data access, from that screen, with a confirmation. Grace approves it in Approvals as today; the requester still can't approve.
  3. After approval, Cursor reads what was granted (`acuity:resources`) and proposes the mappings.
  4. Fiona reviews the proposed mappings and submits them; Sam approves them.
- **Needed so it really is deterministic:**
  - **Server-enforced:** the request endpoint accepts only the on-screen request from a builder. `pnpm acuity:access request` is removed or refused; a skill saying "don't" isn't enough, because Cursor runs under Fiona's session.
  - **One record:** the approved request is the record of which datasets the app may read. Roles in `acuity.config.ts` may only read approved datasets, and `pnpm acuity:check` fails otherwise (replaces building the request from `dataNeeds`).
  - **Dev inherits the laptop approval (decided 2026-11-26).** At the first push the factory gives dev the same grant the laptop's approved request has, with no second request and no second governor approval. Today each environment needs its own request (runbook 7.2). Details:
    - Grace's Approvals history shows that dev received the grant, and from which approved request;
    - inheritance covers only what was approved; if a push needs a dataset or purpose that wasn't approved, the deploy stops and says a new request is needed;
    - revoking or suspending the app's access applies to laptop and dev together;
    - **mappings carry over too (decided 2026-11-26):** Sam's approved laptop mappings go to dev with the push, with no second review. Only approved mappings carry over; if the push has a mapping Sam hasn't approved, the deploy stops and says it needs review. Sam's review history shows dev received them;
    - drop step 7.2 (Grace's and Sam's dev approvals) from the runbook once this ships.
- **Then:** update the skills (`acuity-sources.mdc`, `acuity-app`), runbook prompt 2 and section 5.
- **Goes with:** E1 (several datasets per request), E2, E4, E6.

## E6. The data access grant is per app, not per role

- **Today:** the grant lists reader roles ("analyst, credit manager and auditor read CRE financials"), built from the roles' `reads` in `acuity.config.ts`. This repeats what the app's own roles already decide.
- **Why the grant itself stays:** it's the bank data owner's independent yes. Without it the builder could give their own app bank data by writing the config. It also covers reads that don't go through app records (agents, pipelines, policy documents), and per-person suspension stops someone everywhere at once.
- **Wanted (decided 2026-11-26, option A):**
  - the grant says only "this app may read this dataset for this purpose", approved by Grace;
  - inside the app, the app's roles alone decide who sees the data and which fields (object, record and field rules, see bugs 6-8);
  - keep per-person suspension of data access.
- **What we give up (accepted):** Grace no longer sees which roles read the data, and a new reading role later needs no fresh approval. A widened dataset or purpose still does.
- **Changes:** the D&K grant writer (reader positions), the approval card, the hash that pins the approval (no longer includes roles' `reads`), `acuity:resources` "data access changed" check, `validate`/`acuity:check` rules, the skills and the permissions-streamlining docs.
- **To check:** whether D&K uses reader positions at read time for agent or pipeline calls; if so, those need the app's identity instead.

# To do (later)

- **Serve the onboarding script from the hosted platform.** Today Fiona needs a platform checkout on her laptop just to run `fde-onboard.sh`. The Build locally dialog should give one command that fetches the script from the front door (for example `curl -fsSL http://localhost:4400/api/v1/frontdoor/onboard.sh | sh -s -- --code <code>`), and that script then clones the platform at the commit the front door names (ties in with bug 3). The dialog's command already reads `sh fde-onboard.sh --connect ...` as if the script were standalone.
