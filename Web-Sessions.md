# Web Sessions: Refresh Token Fix

**Ticket:** Web sessions can't be refreshed. The refresh token is in an HttpOnly cookie, but the API only read it from the request body.
**Status:** Fixed. Merged into `dev` (PR #788) and `testing` (PR #796, 2026-11-23).

## The problem

- When a web user signs in, the refresh token is stored in a cookie that browser code can't read.
- The refresh endpoint only looked for the token in the request body, so web refresh always failed.
- The access token cookie also expired too early. After that, the API couldn't tell which session to renew.

Result: web users had to sign in again with an OTP each time their access token expired.

## What was fixed

1. **Web can refresh with cookies only.** If the request has `x-client-type: web` and no token in the body, the API reads the `refreshToken` cookie.
2. **The body can be empty.** Web can send `{}`. Mobile still sends `refreshToken` in the body, as before.
3. **The access cookie lasts longer.** It now lasts as long as the refresh token, so refresh works even after the access token expires. Other routes still reject an expired access token.
4. **Expiry settings are read correctly.** A plain number, such as `"92592000"`, is read as milliseconds.
5. **Logout always works.** Web can clear its cookies even after the access token expires.
6. **Swagger and tests are updated.** All 253 related tests pass.

## Acceptance criteria

| Criterion | Status |
|---|---|
| Web refresh with only cookies returns 200 and a new `accessToken` cookie | ✅ |
| Web refresh still works after the access token expires | ✅ |
| Mobile refresh (body token and Bearer token) still works | ✅ |
| Tokens from different sign-ins get 401 `AUTH_REFRESH_INVALID` | ✅ |
| Swagger explains where each client sends the tokens | ✅ |

## How to test

```http
POST /api/v1/auth/refresh-token
x-client-type: web
Cookie: refreshToken=...; accessToken=...

{}
```

**Expected result:** HTTP 200 and a new `accessToken` cookie in the `Set-Cookie` header.

## Why you may still see the old behavior

| Reason | What to do |
|---|---|
| The testing server hasn't been redeployed yet | Check Swagger: the `/refresh-token` description should say web can send `{}` |
| Your session started before the fix | Sign out, clear the site's cookies, and sign in again |
| The `x-client-type: web` header is missing | Add the header |
| The browser doesn't send cookies | Use `withCredentials: true` (axios) or `credentials: 'include'` (fetch) |
| The request uses plain HTTP | Use HTTPS (or `localhost`) |
| You're testing through the web app | The web app doesn't call `/refresh-token` yet. Test the endpoint directly |
| You're testing on `staging` or `main` | The fix isn't there yet |

**Tip:** in the browser's Network tab, check that the refresh request sends both the `refreshToken` and `accessToken` cookies.

## Still to do (outside this fix)

- **Web app (`reach-web`):** call `/refresh-token` and turn the 401 handling back on.
- **Security review:** confirm that cookie-only refresh is safe from CSRF. Today it relies on the `x-client-type` header and the CORS allow-list.
