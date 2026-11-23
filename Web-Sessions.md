# Web Sessions: Refresh Token Fix

**Ticket:** API: Web sessions can't be refreshed; refresh token is set as HttpOnly cookie but read only from body
**Environment:** Testing (`api-tslp-testing`), backend, affects web
**Branch:** `fix/web-cookie-based-token-refresh` (merged into `dev` in PR #788, and into `testing` in PR #796 on 2026-11-23)

## The problem

- When a web user signs in (`verify-otp` or `verify-magic-link`), the API sets the refresh token only in an HttpOnly cookie, `refreshToken`. Browser code can't read this cookie.
- `POST /api/v1/auth/refresh-token` read the refresh token only from the JSON body. A web request that sent only cookies failed with `Validation failed: refreshToken Required`.
- The `accessToken` cookie expired at the same time as the access token. After it expired, the browser stopped sending it, so the API couldn't tell which session to renew (`400 Access token not found`).

## The fixes

1. **The API now reads the refresh token from the cookie for web.**
   `JwtHelperService.getRefreshToken` in `api/src/jwt-helper/jwt-helper.service.ts` chooses the token like this:
   - If the body has `refreshToken`, the API uses it.
   - If not, and the request has `x-client-type: web`, the API reads the `refreshToken` cookie.
   - Mobile never falls back to the cookie. It must send the token in the body.
   - If no token is found, the API answers `400 Refresh token not found`.

2. **The body field is now optional.**
   `refreshTokenSchema` in `api/src/auth/dto/auth.dto.ts` now marks `refreshToken` as optional, so web can send `{}`. If the field is sent, it still must not be empty.

3. **The service uses the new method.**
   `AuthService.refreshAccessToken` in `api/src/auth/auth.service.ts` takes an optional body token and passes the result of `getRefreshToken` to `resolveRefreshSession`.

4. **The web access cookie lasts longer.**
   `getAccessTokenCookieOptions` now gives the `accessToken` cookie the same lifetime as the refresh token (`REFRESH_TOKEN_EXPIRY`). The browser keeps sending it after the access token expires, so `/refresh-token` can find the session. This doesn't give extra access: the guards still reject an expired access token on all other routes. With `rememberMe`, the cookie lasts at least one month.

5. **Expiry settings that are plain numbers are read as milliseconds.**
   `parseExpiryToMs` now reads a value like `"92592000"` as milliseconds, the same way `jsonwebtoken` reads it. This means a cookie and its JWT expire at the same time. Values like `60d`, `12h`, `30m` and `45s` still work.

6. **Logout works after the access token expires.**
   `POST /api/v1/auth/logout` is now public. Before, an expired access token caused a 401 before the cookies were cleared, so the `refreshToken` cookie stayed in the browser. The route reads no user and only expires the caller's own cookies.

7. **Swagger is updated.**
   The `/verify-otp`, `/verify-magic-link`, `/refresh-token` and `/logout` descriptions now say where web and mobile get and send each token.

8. **Tests were added.**
   - `api/src/jwt-helper/jwt-helper.refresh.spec.ts` (with real tokens)
   - `api/src/jwt-helper/jwt-helper.service.spec.ts`
   - `api/src/auth/auth.controller.spec.ts`
   - `api/src/auth/auth.service.spec.ts`
   - `api/src/auth/dto/auth.dto.spec.ts`

   Result: 5 test suites, 253 tests passed.

## Acceptance criteria

| Criterion | Status | Test |
|---|---|---|
| Web refresh with only the cookies and no body token returns 200 and sets a new `accessToken` cookie | Done | "web: refreshes with the cookies alone and an empty body" |
| This also works after the web access token has expired | Done | "web: refreshes when the access token in the cookie has expired" |
| Mobile refresh with a body token and an expired Bearer token still works (TSLP-2809) | Done | "mobile: refreshes with the body token and an expired Bearer (TSLP-2809)" |
| Tokens from different sign-ins are rejected with 401 `AUTH_REFRESH_INVALID` | Done | "web: refuses cookies from two different sign-ins…", "mobile: refuses tokens from two different sign-ins…" |
| Swagger says where each client gets and sends the refresh token | Done | Swagger descriptions updated |

## How to test

Web request:

```http
POST /api/v1/auth/refresh-token
x-client-type: web
Content-Type: application/json
Cookie: refreshToken=...; accessToken=...

{}
```

Expected result:
- HTTP 200
- The body has only `message` and `expiresIn`.
- The `Set-Cookie` header has a new `accessToken` cookie.

## Possible reasons for seeing the old behavior

1. **The testing environment isn't running the new build yet.**
   The fix reached the `testing` branch on 2026-11-23 (PR #796). If the test ran before that build was deployed, `api-tslp-testing` still had the old code.
   *Check:* in Swagger, the `/refresh-token` description should say web can send `{}`.

2. **The session was created before the deploy.**
   Older sessions have the old, short-lived `accessToken` cookie. When it expires, the browser stops sending it.
   *Fix:* sign out, clear the site's cookies, and sign in again.

3. **The `x-client-type: web` header is missing.**
   Without this header, the API doesn't read the `refreshToken` cookie and answers 400.

4. **The browser doesn't send the cookies.**
   The cookies are `SameSite=None; Secure`. The request must include credentials: `withCredentials: true` (axios) or `credentials: 'include'` (fetch).

5. **The request doesn't use HTTPS.**
   Browsers don't send `Secure` cookies over plain HTTP, except on `localhost`.

6. **The web origin isn't allowed by CORS.**
   Only the origins listed in `api/src/main.ts` (including `FRONTEND_URL`) can send cookies with credentials.

7. **The web app doesn't call the endpoint yet.**
   In `reach-web`, `src/lib/api.ts` has no call to `/refresh-token`, and its 401 interceptor is commented out. Until this is added, testing through the UI will still look broken. Test the endpoint directly for now.

8. **The fix isn't in `staging` or `main` yet.**
   Testing on those environments will show the old behavior.

*How to find the cause:* in the browser's Network tab, check that the refresh request has a `Cookie` header with both `refreshToken` and `accessToken`, and look at the status code and response body.

## Still open (outside this API change)

- **Web client:** `reach-web` must call `/refresh-token` and turn its 401 handling back on.
- **CSRF review:** no code was changed for this. The protection comes from the `x-client-type` header, which forces a CORS preflight, and from the CORS origin allow-list with credentials. Nobody has formally reviewed this yet.
- **Optional:** return `refreshToken` in the `verify-otp` body for mobile. The ticket says this isn't required.
