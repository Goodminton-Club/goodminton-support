# Independent review: PR #9, Goodminton Tournament privacy page (R2765)

Reviewed-SHA: 832ae95ceac6d3d32aecc0706c09f1ca91b56b76
Verdict: REJECT

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
