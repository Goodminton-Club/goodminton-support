# Review: PR #8 (tournament home page + tournament privacy policy)

- **Verdict: REJECT** (changes requested; every blocker is a wording fix to `tournament/privacy/index.html` or `tournament/index.html`, no redesign)
- **Reviewed-SHA:** `53c432088fa71337da828a5c139eae7a8a565cad`
- **Reviewer:** independent reviewer (Opus), run R2675. Builder: Sonnet, run R2674.
- **Method:** fresh clone of this repo on the PR branch; detached worktree of `goodminton-tournament` at `origin/main` (`970f1bf`). Every claim was checked against the frozen spec as migrated (`supabase/migrations/202612010900_F2_schema.sql`, `..._F2_rls.sql`, `..._F2_rpc_stubs.sql`), `spec/sql/rls-tests.md` (read sweep), `spec/stack/TESTIDS.md`, `docs/runbooks/F4/F5`, `.github/workflows/backup.yml`, and the KB (`TOURNAMENT-DECISIONS.md` D1–D40, `docs/TOURNAMENT-PLAN.md` §4.5, §6.4, §6.6, §8.2). Where the plan and the frozen SQL disagree, the policy has to describe what the SQL does (or Kevin decides and the SQL changes first).
- **Note on publishing:** the site has `.nojekyll`, so this file will be served as-is at `goodminton.club/docs/reviews/...` if it is still on the branch at merge. Remove it before merging (or accept that it is public).

## What is right

- Google scopes: only `openid`, `userinfo.email`, `userinfo.profile` (F4 runbook §A), and `signInWithOAuth({provider:'google'})` passes no extra scopes (`apps/web/src/app/core/auth/auth.service.ts`). The "no Gmail/Drive/Calendar/contacts" sentence holds.
- The anonymous board cannot read any personal table: every authenticated-readable table carries an `AS RESTRICTIVE ... not jwt_is_anonymous()` policy; anon has no grant on `players`; personal data reaches the board only through `board_snapshot`.
- Board exclusions are correct: `board_snapshot` never emits birth year, gender, avatar, membership, rating or contact.
- Payments: no online payment (D14, D34; plan out of scope). Desk records paid and, for desk-taken payments, cash/zelle/venmo (`entries.payment_method` check).
- Push: endpoint, p256dh, auth, short device label (`user_agent`); 404/410 subscriptions deleted by push-send.
- Error reports: no user id, message truncated to 500 characters with emails and 6+ digit runs redacted, route template only, purged after 30 days (`log_client_error`, `purge_expired`). Rate-limit hits are purged after 2 days; their subject is the account id or the Cloudflare client IP.
- Deletion text matches the `my_delete_account` contract (anonymise names to Deleted/Player, clear contacts, birth year, avatar and member number, delete push subscriptions, delete `auth.users`, results stay, refused for the last owner of an unfinished tournament). It is still a `NOT_IMPLEMENTED` stub; the policy says "designed to" and gives an email fallback, which is acceptable.
- Hosting: Supabase us-east-1 (verified by the caller for gt-staging and gt-prod), Cloudflare Pages, GitHub Actions keep-alive and age-encrypted backups (F5 §5, step 14).
- HTML: both files parse with balanced tags; `lang`, charset and viewport are set; headings go h1 → h2 → h3 in order; there are no scripts, external stylesheets, images, iframes or trackers; the CSS is the same as `/privacy/`; it is responsive (max-width 760px, 20px gutter). All internal links resolve within the PR (`/tournament/`, `/tournament/privacy/`, `/privacy/`, `/terms/`, `/support/`).

## Blockers

**B1. The public board shows check-in numbers; the policy leaves them out and says check-in data is private.**
`board_snapshot` (rpc_stubs ~L3651): `Label = {"text": "A. Chen" | "#12", "checkin_numbers":[int]}` on every court, call-queue and recent-result row. Plan §8.2 says "The board's call queue shows player names **and** check-in numbers", and `TESTIDS.md` `board-call-queue-item-{i}` says "labels + check-in numbers". The policy's board list (first name + last initial, court, event, round, scores, call queue, recent results) omits them. Its signed-in bullet also says check-in is "visible only to you and to the organizers". A player's check-in number, shown next to their name, also tells the room that they have checked in.
*Fix:* add "each player's check-in number" to the board list, and change the signed-in bullet so that check-in status is no longer listed as private (for example "your check-in record (time and method)").

**B2. The policy understates what any signed-in user can see.**
- `entry_labels(p_tournament_id)`: any signed-in full account can call it for **any non-draft tournament of any club**. `rls-tests.md` row "entry_labels (non-draft / draft)" gives `nobody: ok / PD` and `adminB: ok / PD`. It returns `entry_id, event_id, entry_number, label` for every entry. So from the moment registration opens, before any draw is published, every signed-in user can see who entered which event (first name + last initial, or `#n`). The policy says draws and results become visible only after publication, and that "your ... registration ... [is] visible only to you and to the organizers".
- `matches_select`: matches with `draw_id is null` are readable as soon as their event is visible (any non-draft tournament), not only after a published draw.
- `tie_lineups`: visible to every signed-in user once locked (`tie_lineup_visible`).
- Ladders (D16, in v1): `ladder_standings` gives labels, positions and last-active dates to the club's verified members and entrants (`ladder_visible`). Ladders are not mentioned anywhere in the policy.
- `my_next_match` shows each player their opponent's label.
*Fix (suggested wording):* "Signed-in users of the app can see the entry list of any tournament that is open for registration or later (each entrant's first name and last initial, or entry number, per event), the brackets, groups, schedules and results once published, team lineups once locked, and, for club members, their club's ladder standings. Your contact details, birth year, gender, photo, ratings, payment status and check-in record are not shown to other players." Then drop "registration" from the private list.
*Kevin decision:* if a cross-club entry list for any signed-in user is **not** intended, tighten `entry_labels` in `goodminton-tournament` first (for example, require a published draw or staff), and then the narrower wording can stay.

**B3. "Who can see your information" leaves out platform administrators, umpires and guardians.**
- `app_admins` (platform operators) read **all** rows of `players` (email, phone, birth year), `club_memberships`, `member_roster`, `player_ratings`, `entries`, `draws.inputs_snapshot`, `audit_log` and more (rls-tests column `plat`: "all").
- Umpires: `private.is_staff` includes `umpire`, so a tournament umpire can read the `players` rows (contact, birth year, gender), entries, check-ins and score reports of that tournament's players (`staff_can_see_player`, `entries_select`, `checkins_select`). The policy's Organizers bullet names only "owners, admins and desk staff".
- Guardians read their wards' data (`my_player_ids`).
- Club owners, admins and desk see their club's members' player rows even without a tournament entry (`staff_can_see_player` via `club_memberships`). "Organizers ... of the relevant ... club" covers this; keep it.
*Fix:* name umpires alongside desk staff, add "Goodminton platform administrators (who operate the service) can access all data for support, security and operations", and add "a junior's guardian".

**B4. The profile-photo sentence does not match the storage policy.**
The policy says photos are "shown only to signed-in viewers within the same tournament". `avatars_select` → `private.can_view_avatar` = the player or their guardian, **or** `staff_can_see_player`: staff (including umpires) of a tournament the player entered or checked in to, a club's owner/admin/desk for its members, and a club admin reviewing the player's membership claim. Other players in the same tournament **cannot** see it, and club staff **can** see it outside any tournament. (Plan §4.5 says "signed-in viewers in that tournament". The builder cited the plan, but the frozen SQL is narrower.)
*Fix:* "Profile photos are stored privately and are visible only to you (and a junior's guardian) and to the organizers, desk staff and umpires of tournaments you enter or clubs you belong to. They never appear on the public board." Or Kevin picks the plan behaviour and the RLS changes first.

**B5. `/terms/` is the mobile app's Terms of Service, but the tournament home page lists it as the tournament app's policy.**
`terms/index.html` says it governs "the Goodminton mobile application ... By downloading or using the app", with on-device storage and App Store or Play Store purchases. Listing it under the tournament app's "Policies" (and in the footer) presents mobile terms as this web app's terms. This contradicts the careful "separate product" framing used for the privacy policy.
*Fix:* remove the Terms link from the tournament pages, or label it "Goodminton mobile app Terms of Service (separate product)". Do not enter `/terms/` as the Terms URL on the Google consent screen. That field is optional, so leave it blank until tournament terms exist.

## Follow-ups (non-blocking; best fixed in the same pass)

1. **Payment method scope:** `entries_payment_method_paid` stores a method only when `paid and source = 'desk'`. Online entries carry only paid/unpaid. Also, "fees are handled by the club at the desk" may not cover online registrants who pay ahead. Suggest "handled by the club outside the app (for example at the desk by cash, Zelle or Venmo)".
2. **Roster contents:** `member_roster` holds name, member number, expiry date, active/inactive status and membership name. It covers members who may never use the app. `membership_claims` stores the name the player typed, the number and the outcome. List these.
3. **Deletion leftovers:** under the contract, gender, self level, **affiliation** (free text, possibly identifying), ratings and rating history, check-in records and score-report notes stay attached to the anonymised player. Either extend `my_delete_account` to clear `affiliation` (tournament repo) or say these remain. The audit log is never deleted (the schema says "the audit outlives everything"), so say "kept indefinitely".
4. **Audit log content and purpose:** `write_audit` de-identifies `args` only (K5). `before`/`after` jsonb snapshots have no PII rule yet. Verify this when M1 lands, or soften the "internal reference" sentence. Also disclose the second purpose: de-identified replay in the Debug Lab (K5).
5. **Unlisted records:** in-app `notifications` (kind, payload, read time, kept until the account is deleted), `idempotency_keys` (stored RPC responses, purged after 7 days), and `rest_waivers` (D38 skip-rest opt-ins).
6. **Backups:** GitHub artifact `retention-days: 30` (`backup.yml`), so say "up to 30 days". The schedules are still commented out (dispatch-only), which the "designed to work" framing covers.
7. **Anonymous board and auth logs:** kiosk sessions expire after 24 h and are purged, but the Supabase anonymous `auth.users` row persists (no name or email). Supabase Auth also logs sign-ins (IP, user agent). Neither is mentioned; the only IP mention is under Cloudflare.
8. **"Processed in the United States":** Cloudflare's edge, the browser push services and Supabase Edge Functions run outside us-east-1. Prefer "stored in the United States (AWS us-east-1)".
9. **Entry-number-only option:** `my_set_board_display` lets the player **or** a guardian choose it. The policy offers it only to a junior's guardian (this follows plan §4.5). It understates a user option, which is harmless; say "you (or a junior's guardian)" if that is the intended behaviour.
10. **Desk-added junior guests:** the `desk_add_guest` contract records no guardian link or consent. The Children section's consent claim is grounded only for the online path (`review_entry` refuses a junior without consent). Confirm how the desk handles consent.
11. **"Last owner" rule:** K10 says there is no per-tournament owner (owner exists only at club level), but `my_delete_account` speaks of "the last owner of a non-complete tournament". Align the RPC contract and the policy wording when M1 implements it.
12. **Material changes:** "we will tell signed-in users in the app" promises a mechanism that is not designed yet (no announcement or banner in the spec). Either add it to the backlog or soften the sentence.
13. **Email provider:** it is unnamed because Resend vs Brevo is not decided. Name it once chosen (D20).
14. **HTML nits:** `tournament/privacy/index.html` has no `<meta name="description">` (the home page has one). The contact address is plain text rather than a `mailto:` link, which matches `/privacy/`, so that is fine.

## Claims Kevin must confirm before merge

1. The **30-day** response commitment for deletion and data-copy requests sent by email to the site contact address.
2. The **under-13 posture:** "not directed at children under 13", and no age gate exists on self sign-up; guardian-created ward profiles may be under 13. Is this acceptable, or should an age gate or parental-consent step be added?
3. **Entry lists visible to every signed-in user across clubs** (B2). Intended, or tighten `entry_labels`?
4. **Avatar audience** (B4): plan §4.5 (signed-in viewers in the tournament) vs RLS (self, guardian and staff only). Which one is the decision?
5. **Platform administrators:** who they are (only Kevin?), and that the policy may say they can access all data.
6. **Terms of Service** (B5): drop the link, or write tournament terms.
7. **How online registrants pay** (follow-up 1), and **how the desk handles junior consent** (follow-up 10).
8. **Promises:** "we will delete it" (children) and "tell signed-in users in the app" (material changes).
9. **Email delivery provider** choice, and that anonymous sign-in will be enabled on gt-prod for the board.
10. **Already verified, no action needed:** the Supabase region is us-east-1 for gt-staging and gt-prod (verified by the caller), and reusing the existing contact address is intentional.
