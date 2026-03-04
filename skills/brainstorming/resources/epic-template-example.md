# Complete Epic Template Example

This is a complete `br create` example showing the full structure a well-formed epic should have, including requirements, success criteria, anti-patterns, approach, architecture, design rationale, and open concerns.

## Example: OAuth Authentication Epic

```bash
br create "Epic: OAuth Authentication" --design "
## Requirements (IMMUTABLE)
- Users authenticate via Google OAuth2
- Tokens stored in httpOnly cookies (NOT localStorage)
- Session expires after 24h inactivity
- Integrates with existing User model at db/models/user.ts

## Success Criteria
- [ ] Login redirects to Google and back
- [ ] Tokens in httpOnly cookies
- [ ] Token refresh works automatically
- [ ] Integration tests pass WITHOUT mocking OAuth
- [ ] All tests passing

## Anti-Patterns (FORBIDDEN)
- ❌ NO localStorage tokens (security: httpOnly prevents XSS token theft)
- ❌ NO new user model (consistency: must use existing db/models/user.ts)
- ❌ NO mocking OAuth in integration tests (validation: defeats purpose of testing real flow)
- ❌ NO skipping token refresh (completeness: explicit requirement from user)

## Approach
Extend existing passport.js setup at auth/passport-config.ts with Google OAuth2 strategy.
Use passport-google-oauth20 library. Store tokens in httpOnly cookies via express-session.
Integrate with existing User model for profile storage.

## Architecture
- auth/strategies/google.ts - New OAuth strategy
- auth/passport-config.ts - Register strategy (existing)
- db/models/user.ts - Add googleId field (existing)
- routes/auth.ts - OAuth callback routes

## Design Rationale
### Problem
Users currently have no SSO option - must create accounts manually.
Manual signup has 40% abandonment rate. Google OAuth reduces friction.

### Research Findings
**Codebase:**
- auth/passport-config.ts:1-50 - Existing passport setup, uses session-based auth
- auth/strategies/local.ts:1-30 - Pattern for adding strategies
- db/models/user.ts:1-80 - User model, already has email field

**External:**
- passport-google-oauth20 - Official Google strategy, 2M weekly downloads
- Google OAuth2 docs - Requires client ID, callback URL, scopes

### Approaches Considered

#### 1. Extend passport.js with google-oauth20 ✓

**Chosen because:** Consistent with auth/strategies/local.ts pattern, minimal changes to existing code.
**Pros:** Matches existing codebase pattern, session handling already works, well-documented.
**Cons:** Adds npm dependency.

#### 2. Custom JWT-based OAuth ❌

**Why explored:** User mentioned 'maybe we should use JWTs'
**Why rejected:** Scope creep — would require rewriting 15 files using req.session. OAuth feature shouldn't rewrite existing auth system.
**🚫 DO NOT REVISIT UNLESS:** We're already rewriting the entire auth system in a separate epic.

#### 3. Auth0 integration ❌

**Why explored:** Third-party service might reduce implementation complexity.
**Why rejected:** Overkill for single OAuth provider. Introduces vendor dependency inconsistent with codebase.
**🚫 DO NOT REVISIT UNLESS:** We need 3+ OAuth providers AND are okay with vendor dependency.

### Scope Boundaries
**In scope:** Google OAuth login/signup, token storage in httpOnly cookies, profile sync with User model.
**Out of scope:** Other OAuth providers (deferred), account linking (deferred), custom OAuth scopes.

### Open Questions
- Should failed OAuth create partial user record? (decide during implementation)
- Token refresh: silent vs prompt? (default to silent, user can configure)

## Design Discovery (Reference Context)

### Key Decisions Made

| Question | User Answer | Implication |
|----------|-------------|-------------|
| Token storage preference? | httpOnly cookies for security | Anti-pattern: NO localStorage |
| New user model or extend existing? | Use existing at db/models/user.ts | Must add googleId field, not new table |
| Session duration? | 24h inactive timeout | Need refresh token logic |
| What if Google OAuth is down? | Graceful error message | No fallback auth required |

### Dead-End Paths

#### Custom JWT Implementation
**Why explored:** User mentioned 'maybe we should use JWTs'
**Why abandoned:** Scope creep — 15 files use req.session, 2 weeks migration, security complexity.

#### Auth0 Integration
**Why explored:** Third-party service might be simpler.
**Why abandoned:** Overkill for single provider, vendor dependency.

### Open Concerns Raised
- 'What if Google OAuth is down?' → Graceful degradation to error message, no fallback auth
- 'Should we support account linking later?' → Deferred to future epic, out of scope for now
- 'Token refresh - silent or prompt?' → Default silent, user can configure
"
```
