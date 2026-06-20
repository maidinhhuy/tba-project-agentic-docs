# Sprint Change Proposal — 2026-06-19

**Approved:** Yes (user confirmed)
**Change scope:** Moderate — New Epic 6 added, AD-12 reverted, PRD updated

---

## Section 1: Issue Summary

**Trigger:** Product decision to restore email verification flow (temporarily disabled 2026-06-13 per AD-12) and add change-password capability.

**Context:** Story 2.2 (Resend email infrastructure) and Story 5.1 (transactional services) are now complete, providing the necessary foundation. Resend API key provisioned and deployed to production. The system is ready to enable the full email verification flow.

**Scope of change:**
1. Email verification on registration — user must confirm email before account is active
2. Resend verification email — re-trigger verification for expired/missed emails
3. Change password — authenticated endpoint requiring current password for BCrypt verification

---

## Section 2: Impact Analysis

**Epic Impact:**
- Epic 2 (Authentication): Story 2.3 behavior superseded by Story 6.1 (no rollback needed — Story 6.1 overrides)
- New Epic 6 added with Story 6.1 and Story 6.2

**Artifact Conflicts:**
- PRD §4.1: Description updated to reflect email verification and change password in scope; §6.2 out-of-scope list updated
- Architecture AD-12: Reverted from "skip email verification" to full verification flow; new ports documented
- Epics.md: Epic 6 appended with full Story 6.1 and Story 6.2 specifications
- sprint-status.yaml: Epic 6 block added with `backlog` status

**Technical Impact:**
- `email_verifications` table already exists in V1 schema — no new Flyway migration needed
- `ResendEmailClient` already live — only new use cases and endpoints needed
- Frontend: new `/register/success` page, `/verify-email` page, `/profile` change-password form

---

## Section 3: Recommended Approach

**Path:** Direct Adjustment — add Story 6.1 and 6.2 without rolling back Story 2.3. Story 2.3 remains as historical record.

**Rationale:** Rolling back would disrupt existing registered test users. Story 6.1 overrides the `is_active=true` bypass behavior cleanly through new use cases.

**Effort estimate:** ~3-4 days BE+FE combined
**Risk:** Low — infrastructure (Resend, email_verifications table) already in place
**Timeline impact:** +1 sprint for Epic 6

---

## Section 4: Detailed Change Proposals

### PRD Changes (Applied)
- §4.1 Description: Removed "Không có account management UI cho Customer ở v1" — replaced with self-registration + email verification description
- §4.1 Out of Scope: Updated to reflect change-password is now in scope (FR-18); quên mật khẩu / 2FA / SSO remain deferred
- FR-16 (email verification), FR-17 (resend), FR-18 (change password) added to §4.1

### Architecture Changes (Applied)
- AD-12 fully rewritten: restores full email verification flow, documents new ports (VerifyEmailUseCase, ResendVerificationUseCase, ChangePasswordUseCase, EmailVerificationRepository), SecurityConfig endpoints, exception types

### Epics Changes (Applied)
- Epic 6 appended with full Story 6.1 (Email Verification BE+FE) and Story 6.2 (Change Password BE+FE)

### Sprint Status Changes (Applied)
- Added: `epic-6: backlog`, `6-1-email-verification-flow: backlog`, `6-2-change-password: backlog`

---

## Section 5: Implementation Handoff

**Scope classification:** Moderate

**Next steps for Developer agent:**
1. Run `bmad-create-story` for Story 6.1 (Email Verification Flow)
2. Implement BE: `VerifyEmailUseCase`, `ResendVerificationUseCase` + endpoints + SecurityConfig update
3. Implement FE: `/register/success`, `/verify-email`, login error handling
4. Run `bmad-create-story` for Story 6.2 (Change Password)
5. Implement BE: `ChangePasswordUseCase` + endpoint + refresh token revocation
6. Implement FE: `/profile` change-password form + post-change logout

**Success criteria:**
- New user registration → receives verification email → clicks link → can log in
- Unverified user login attempt → 403 EMAIL_NOT_VERIFIED error shown in FE
- Resend verification works for expired tokens
- Change password with correct old password → success + auto logout
- Change password with wrong old password → 400 WRONG_PASSWORD shown in FE
