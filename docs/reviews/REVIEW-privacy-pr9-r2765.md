# Independent review: PR #9, Goodminton Tournament privacy page (R2765)

Reviewed-SHA: 0f2f3db4bc0d519b4eca9cb8c24155e5f58310c2
Verdict: APPROVE

- PR: Goodminton-Club/goodminton-support#9 (`docs/privacy-affiliation-r2687`), file `tournament/privacy/index.html`.
- Checked against goodminton-tournament `main` at **c4601c00b73ea8f7c0787d7c0a41ad7ea946dd7a**. That includes #62 (D45: one person, one deletion). I first read `main` at 136f2b1, before #62 merged. The merged-logins finding from that pass is now resolved; see "Resolved during review".
- Decisions checked: KB `TOURNAMENT-DECISIONS.md` D29 and D41–D45 (my-brainstorming `main`).
- Reviewer: independent (R2765, kind=review). I did not write this PR.

## Blockers (each one is a claim on the page that the code on `main` does not support)

### B1. Invitations paragraph: it understates who can see the invitee's email, and the guardian clause is wrong
- The page says: "The invitee's email address is visible only to the captain and to the tournament's organizers and desk staff."
- The code: `entry_invites_select` (`202612010901_F2_rls.sql`) allows `invited_by = auth.uid()` **or** `private.is_event_staff(...)`. `is_event_staff` calls `is_staff`, which is `has_role(..., array['owner','admin','desk','umpire'])`, so **umpires** of the tournament (and of the host club) can read `invitee_email` too. On a privacy page, understating who can see data is the dangerous direction.
- The page says: "The invitation link only works for an account signed in with that verified email address (or for that person's guardian)."
- The code: `private.require_invite_email` (`202612020950_R4_invite_email_binding.sql`) always checks the **caller's own primary confirmed** email. A guardian can accept for a ward only when the link was sent to the guardian's own email. The guardian of the person whose email is on the link cannot use it.
- Fix (suggested wording): "...only works for the account signed in with that verified email address (a guardian who receives the invitation can accept it for a junior they manage). The invitee's email address is visible only to the captain who created the invitation and to the tournament's organizers, desk staff and umpires (and platform administrators)."

### B2. The audit-log wording overclaims
- The page says (Technical records, the Retention line and the Changes note): "Audit records are designed to contain references (internal player and entry identifiers) rather than personal details such as names, contact details or birth year." The previous wording kept the caveat "the before/after snapshots of a change may contain personal details". This PR removes it.
- The code on `main` writes personal data into `audit_log`, which is kept indefinitely and not deleted on account deletion:
  - `unbind_membership` (`202612020980_R6_roster_claims_merge.sql:490-491`) stores the **member number** (`member_ref`) unredacted in the `before` snapshot. The page lists the member number as personal data that deletion clears, but this copy stays.
  - Free-text staff reasons (1–500 chars) go verbatim into `audit_log.reason`. `audit_args` redacts them only inside `args`. Affected RPCs: `unbind_membership`, `platform_merge_players`, `admin_set_level_tier`, `promote_waitlisted` (override), `pair_entries`, `desk_override_team_number`. Staff can type anything there, including a name.
  - `withdraw_entry` records `withdraw_reason` = `injury` (health-related) in `after` and `reason`.
- Fix: either restore an honest caveat, for example: "Audit records mostly use internal references instead of names, contact details or birth year, but some records can include a member number, a short reason typed by staff, or a withdrawal reason (such as injury)." Use it in all three places. Or change the code first (redact `member_ref`, stop copying free text into `reason`) and then keep the current wording.

### B3. Email sign-in is a 6-digit code, not a link (D41)
- The page says: "Sign in with an email link: you give us your email address and we send a one-time sign-in link." The Brevo row says "Sends sign-in link emails".
- The code: `supabase/templates/magic_link.html` and `confirmation.html` carry only `{{ .Token }}` ("Your Goodminton sign-in code"). `config.toml` sets `otp_expiry = 900`. The app calls `verifyOtp({ email, token, type: 'email' })` (`apps/web/src/app/core/auth/auth.service.ts`). KB D41 says the email has no link.
- This text was already on `main` and this PR did not touch it. But the page is now claiming to be accurate, and this is a false factual claim. Fix: "Sign in with an email code: you give us your email address and we email you a 6-digit code that works once and expires after 15 minutes." Brevo row: "Sends sign-in code emails".

### B4. Backups are described as running today
- The page says (provider table): GitHub Actions "Runs automated backups" with "Daily database backups ... encrypted before they are stored". The Changes note says "database backups are kept for at most 30 days".
- The code: `.github/workflows/backup.yml` is **dispatch-only**. The `schedule:` block is commented out ("uncomment ONLY after the secrets exist"). `docs/DEPLOYED.md` shows the workflow as "merged — not applied". Hosted daily backups are not enabled (coordinator, this run). Kevin's decision is that backups must be on before go-live, and that the page must not say they run today unless the wording makes that true. Only the intro's general line, "this policy describes how it is designed to work", would make the present tense true. That is too weak for the item Kevin singled out.
- Fix (suggested wording): "Will run automated daily database backups before launch (not yet enabled while the app is in testing)". In the Retention line: "once backups are enabled, encrypted database backups are kept for up to 30 days...". Alternatively, hold this PR until the backup schedule is enabled on prod.

## What I verified as accurate (code on `main` c4601c0)
- **Account deletion** (`my_delete_account`, latest body `202612021050_D45_delete_whole_person.sql`):
  - Names become "Deleted" / "Player".
  - `email_normalized`, `phone_e164`, `birth_year`, `avatar_path` and **`affiliation`** are nulled (D42).
  - Club memberships are retired (`status = rejected`, `member_ref` replaced).
  - Membership-claim typed name and number are replaced. Claim and entry `review_note` are cleared.
  - Push subscriptions are deleted. The `auth.users` row is deleted.
  - `team_name`, gender, `level_tier`/`self_level`, ratings, check-ins and score reports are untouched, so "kept" is correct.
  - The last-owner rule is club-level (`WRONG_STATE: you are the last owner of a club`), matching D43.
  - **Merged logins:** called from either login, it anonymises the survivor and every player merged into it, and deletes every login (D45 #3).
- **Profile photo file removal:** `avatar_path` is nulled, so the R5 `avatars_select` policy makes the object unreadable at once (`avatar_object_current`). The `avatar-sweeper` Edge Function, run by `gt_sweep_avatars` daily at 04:47 UTC with a 24 h grace period, then removes the file through the Storage API.
- **Invitations:**
  - `create_entry_invite_v2` sets a 7-day expiry.
  - Accept, decline and cancel each set `invitee_email = null` immediately.
  - `purge_expired` (daily 04:17 UTC) nulls the email of timed-out links, so "at most one day after" holds.
  - The email is redacted in audit args, so "not written to logs" holds for `audit_log`.
- **Roster retention:** `purge_expired` deletes `member_roster` rows of imports with `not is_current and superseded_at < now - 30 days` (`202612021000_D44_roster_names_retention.sql`). `upload_roster` sets `superseded_at`.
- **Other retention periods:** rate-limit 2 d, idempotency 7 d and client errors 30 d are all in `purge_expired`.
- **Backups contain database records only:** `tools/ops/backup.sh` is a `pg_dump --format=custom` of the database, encrypted with `age`. The artifact has `retention-days: 30`. No Storage objects are included.
- **Photo visibility per RLS:** `can_view_avatar` covers self, guardian, and `staff_can_see_player` (tournament owner/admin/desk/umpire; club owner/admin/desk). This matches the "Profile photos" bullet.
- **Changes since my first pass:** #63–#71 changed no privacy-relevant data path. `purge_expired`, the invite RPCs, audit helpers, templates and `backup.yml` are unchanged. The R7 setup RPCs audit only venue and tournament settings. `db.yml` gained CI steps only.

## PII and internal names
- The diff adds no email address. `kevinliu217@gmail.com` is the existing site contact and already appears on `main` pages (support, terms, privacy, tournament, delete-account).
- The diff contains no internal names, run IDs, decision IDs or function names.

## HTML, links, rendering
- A static parse found balanced tags and no stray `li`. The doctype, `lang`, charset and viewport are present.
- Links `/tournament/`, `/privacy/`, `/support/` and `/tournament/privacy/` all return 200 from a local static server, and all four exist in the repo.
- Rendering: the Playwright MCP browser was refused (already in use), so I used headless Chrome instead. The desktop screenshot (1200 px) renders correctly, including the nested Cleared/Kept list and the provider table. The narrow-width screenshot was inconclusive because headless Chrome enforces a minimum window width. The PR does not change the CSS.

## Follow-ups (not blocking, but recommended in the same revision)
1. **Wards:** a guardian's deletion also anonymises every junior profile for whom they are the only guardian (`orphan_ward` branch). The page does not say so, and it is a material effect.
2. **Club roster:** the club's uploaded roster (name and member number) is not changed by account deletion. It is deleted only 30 days after a newer import supersedes it. Say so under Deleting your account.
3. Roster wording: "(only the current roster is kept)" reads as a contradiction of the 30 days. Suggested: "after that, only the current roster is kept; a summary of each import (row counts, file fingerprint, column headings) remains".
4. Invitations: if the entry ends some other way (withdrawn or rejected), the link is dead but the email is kept until the 7-day expiry plus up to one day. An invitation addressed **to** a person who then deletes their account is not cleared early either. Either is fine to state as "at most 8 days after it was created".
5. Photo timing: "removed from storage within about two days, and unreachable immediately" would be more precise.
6. In-app notifications, staff roles and guardian links are deleted on account deletion (FK cascade or explicit). They could be listed under "Cleared".
7. Go-live gate, not page text: `docs/DEPLOYED.md` shows R5 (pg_cron jobs, avatar-sweeper, Vault secrets) as "merged — not applied". None of the retention promises is enforced on any hosted project until runbook R5 has been run.

## Resolved during review
- At 136f2b1, deleting through the alias login of a merged person left the survivor's profile intact, and deleting through the survivor left the other login in `auth.users`. That contradicted "your names ... are cleared ... your sign-in account is deleted". #62 (`202612021050_D45_delete_whole_person.sql`) has merged and fixes this. I verified the new body on c4601c0.

## To reach APPROVE
Fix B1–B4 (all text-only, unless you choose the code route for B2), then ask for a re-review bound to the new head.

---

# Round 2 (R2800)

- Bound to PR head **845fcd4fe7e8d6df4bd7a2133267553909b9deed** (one builder commit on top of 1b94df3: `845fcd4`, R2799).
- Checked against goodminton-tournament `main` **c4601c00b73ea8f7c0787d7c0a41ad7ea946dd7a**. It was still the tip at review time, and nothing under `supabase/`, `.github/` or `tools/ops/` had changed since round 1.
- Scope: `git diff 1b94df3 845fcd4` touches only `tournament/privacy/index.html` (9 lines added, 8 removed). I checked every changed sentence against the code.
- **Round 2 verdict: REJECT.** Two new claims are inaccurate. Both are one-phrase fixes.

## Round 1 blockers: status
- **B1 invitations: fixed.**
  - The link works only for the account whose primary confirmed email matches (`private.require_invite_email`). A guardian who receives the invitation at their own email can accept it for a ward. Both are correct.
  - The visibility list now matches `entry_invites_select`: the creator (`invited_by`), plus `is_staff` = tournament_staff admin/desk/umpire or host-club club_staff owner/admin/desk/umpire. `club_staff_role_chk` does include `umpire`. Platform administrators have operator access.
  - "Deleted as soon as accepted, declined or cancelled" is correct: each of those RPCs sets `invitee_email = null`.
  - "Otherwise deleted after the 7-day expiry, at most about 8 days after creation" is correct: `purge_expired` runs daily at 04:17 UTC, and `create_entry_invite_v2` sets `+ interval '7 days'`.
- **B2 audit log: fixed.**
  - The caveat is restored in all three places.
  - I re-checked the latest body of every `write_audit` call. Names, birth year, gender, email, phone, affiliation, typed claim names, notes and team names are redacted by `audit_args`.
  - The exceptions the page now names are exactly the ones in the code: the unredacted `member_ref` in `unbind_membership` `before`; free-text `reason` in unbind, merge, tier override, waitlist override, pairing and team-number override; and `withdraw_reason` (`injury`).
  - The new R7 setup audits contain only venue and tournament data. Venue addresses are reduced to `has_address`.
- **B3 sign-in: fixed.** "6-digit one-time code, works once, expires after 15 minutes" matches `supabase/templates/magic_link.html`, `confirmation.html`, `otp_expiry = 900` and D41. The Brevo row now reads "sign-in code emails".
- **B4 backups: fixed.**
  - The provider row, the Retention line and the Changes note now say backups are planned before launch and not running yet. That matches `backup.yml`, whose `schedule:` block is still commented out.
  - "Kept for 30 days" matches `retention-days: 30`. "Database records only" matches the `pg_dump` in `tools/ops/backup.sh`.
  - "Runs backup and keep-alive jobs" describes the provider's role. The same cell says backups are not running yet, so it is acceptable.

## Round 1 follow-ups: status
- **Roster wording: correct.** "A summary of each import: row counts, a file fingerprint and column headings" matches `roster_imports` (`row_count`, `file_sha256`, and a header map holding headers and field names, never values). `uploaded_by` is a reference.
- **"Not changed by deletion" bullet: correct.** `my_delete_account` does not touch `member_roster`, and `purge_expired` deletes superseded rows after 30 days. The audit log is not cleared.
- **Invitations about 8 days: correct** (see B1).
- **Photo removed within about two days: correct for the file.** The sweeper picks objects older than 24 h (by upload time) that no player references. It runs daily at 04:47 UTC and removes up to 200 per run, so the worst case is about 48 h. The sentence before it is wrong; see R2-B1.
- **Guardian deletion and wards: inaccurate as worded.** See R2-B2.

## Round 2 blockers

### R2-B1. "Your profile photo becomes unreachable immediately" is not true
- After deletion, `avatar_path` is null and the `avatars_select` policy (`avatar_object_current`) refuses **new** signed URLs at once.
- But the app reads photos through **signed URLs valid for one hour**: `libs/data/src/avatars.ts`, `SIGNED_URL_SECONDS = 3600`, `createSignedUrl`. A signed URL is not re-checked against RLS. Any URL already issued, for example to an organizer's open desk page, keeps serving the photo until it expires (up to 1 h) or the sweeper deletes the file, whichever comes first.
- Fix (suggested wording): "After deletion no one can get a new link to your profile photo; any link already shown expires within an hour, and the photo file itself is removed from storage by a daily cleanup job within about two days." Or shorten `SIGNED_URL_SECONDS` in the code and state that number instead.

### R2-B2. "If you are the only guardian of a junior, deleting your account also anonymises that junior's profile" is too broad
- The `orphan_ward` branch of `my_delete_account` (`202612021050_D45_delete_whole_person.sql`) anonymises a ward only when `w.auth_user is null`, meaning the junior has **no sign-in of their own**, and no other guardian remains.
- A ward can gain a login. For example, `platform_merge_players` moves the login onto the surviving player when only the merged-away player had one. That happens when a teen's own sign-up profile is merged into the guardian-made profile, and the guardian links move with it. The guardian's deletion then does **not** anonymise that junior, and it should not.
- Fix (suggested wording): "If you are the only guardian of a junior profile that has no sign-in of its own, deleting your account also anonymises that profile in the same way." Make the same change to the Changes note ("juniors they alone manage" becomes "...that have no sign-in of their own").

## Nits (not blocking)
- Spelling: "anonymises" in two new sentences, while the rest of the page uses US spelling ("anonymized"). Pick one.
- "The captain who created the invitation": for a junior captain, the creator (`invited_by`) is the guardian acting for them. That is covered by the page's general guardian language, but "the captain (or their guardian) who created the invitation" would be exact.
- Supabase Auth sign-in logs (IP, browser) are not removed by account deletion, and the page states no retention period for them. This was already the case before this PR. Consider adding it to "Not changed by deletion" in a later revision.

## Checks
- **No other changes since 1b94df3.** Only `tournament/privacy/index.html` changed. The review file and every other page are untouched.
- **HTML:** a static parse found balanced tags and no `li` nested in an `li`. The new nested `<li>` sits inside the existing inner `<ul>`.
- **Links:** `/tournament/`, `/privacy/`, `/support/` and `/tournament/privacy/` all map to files in the repo.
- **Rendering:** headless Chrome at 1200 px shows the new bullets and the provider-table cell laid out correctly.
- **PII and internal names:** the added lines contain no email address, no internal names, no run or decision IDs and no function or table names. The contact address is the existing site contact.

## To reach APPROVE
Fix R2-B1 and R2-B2 (text only), then ask for a round-3 re-review bound to the new head. Kevin's conditional merge approval does **not** apply to this head, because round 2 is not APPROVE.

---

# Round 3 (R2805)

- Bound to PR head **0f2f3db4bc0d519b4eca9cb8c24155e5f58310c2**. Two builder commits sit on top of 5beb522: `ee9ffbc` (photo-link and guardian wording) and `0f2f3db` (spelling).
- Checked against goodminton-tournament `main` **c4601c00b73ea8f7c0787d7c0a41ad7ea946dd7a**, still the tip. `supabase/`, `libs/data/src/avatars.ts`, `.github/` and `tools/ops/` are unchanged since round 2.
- Scope: `git diff 5beb522 0f2f3db` touches only `tournament/privacy/index.html`, 3 lines changed. The review file and the other pages are untouched.
- **Round 3 verdict: APPROVE.** There are no blockers.

## Round 2 blockers: status
- **R2-B1 photo link: fixed.** The page now says: "No one can get a new link to your profile photo, any photo link already shown expires within an hour, and the photo file itself is removed from storage by a daily cleanup job within about two days." Each part matches the code:
  - **No new link:** after `my_delete_account`, `avatar_path` is null. The `avatars_select` policy (`avatar_object_current` plus `can_view_avatar`) refuses `createSignedUrl` to every app user. Platform operator access is disclosed elsewhere on the page.
  - **Existing links expire within an hour:** the app's only photo link is `createSignedUrl(path, SIGNED_URL_SECONDS)` with `SIGNED_URL_SECONDS = 3600` (`libs/data/src/avatars.ts`). No other `createSignedUrl` or `getPublicUrl` call exists in `apps/` or `libs/`.
  - **File removed within about two days:** the `avatar_sweep_candidates` grace period is 24 h from upload, and `gt_sweep_avatars` runs daily at 04:47 UTC. The worst case is about 48 h.
- **R2-B2 guardian deletion: fixed.** "If you are the only guardian of a junior whose profile has no sign-in of its own" now matches the `orphan_ward` branch (`w.auth_user is null`, `w.anonymized_at is null`, no guardian left outside the deleting person's logins). The Changes note was updated to match.

## Round 2 nits: status
- **Spelling: fixed.** Every occurrence on the page is now "anonymized" or "anonymizes" (lines 118, 119, 132).
- **Invitation creator: fixed.** "The captain (or their guardian) who created the invitation" matches `invited_by` and `is_entry_captain` (`my_player_ids` includes wards).
- **Supabase Auth sign-in-log retention: still not stated.** This was already the case before the PR, and it is not a false claim. Optional for a later revision.

## Checks
- **HTML:** a static parse found balanced tags, no `li` nested in an `li`, and no leftover unclosed elements.
- **Links:** `/tournament/`, `/privacy/`, `/support/` and `/tournament/privacy/` all map to files in the repo.
- **PII and internal names:** the added lines contain no email address, no internal names, no run or decision IDs and no function names.
- **Rendering:** round 2 rendered correctly in headless Chrome, and the round 3 text edits are inside existing elements.

## Not part of this PR (operational go-live gates, for the record)
- The page states retention that is enforced only after runbook R5 has been applied on the hosted projects: pg_cron jobs, avatar-sweeper deploy and Vault secrets. `docs/DEPLOYED.md` still shows all of these as "merged — not applied".
- The page also commits to enabling daily backups before launch: the `backup.yml` schedule is still commented out.
- The page words both as testing-phase or "before launch", so they do not block this merge. They do block go-live.

## Merge
Under Kevin's conditional approval this head is eligible to merge (round 3 APPROVE, no blockers). The reviewer does not merge.
