IMPORT:
  - google-auth-library
    - OAuth2Client

client = new OAuth2Client(GOOGLE_CLIENT_ID)

authRouter:
ROUTE: POST /auth/google/verify
  VALIDATE:
    - idToken is a string
    - idToken is not empty
  CONTROLLER LOGIC:
    - TRY:
      - client.verifyIdToken({ idToken, audience: GOOGLE_CLIENT_ID })
      - token is valid:
        - IF new user: create user in the database
        - ELSE: update and retrieve user from the database
    - CATCH: throw error 'Invalid idToken' -> 401 UNAUTHORIZED
    - Generate tokens
      - access token
        - TTL 15m
        - ACCESS_TOKEN_SECRET
        - userId, role, ...
      - refresh token
        - TTL 7d
        - REFRESH_TOKEN_SECRET
        - userId
    - Return access and refresh +tokens -> 200 OK

ROUTE: POST /auth/refresh
  VALIDATE:
    - refreshToken is a string
    - refreshToken is not empty
  CONTROLLER LOGIC:
    - Verify refreshToken using jwt library
    - IF valid
      - find refresh token in DB
      - IF token not found or expires_at < now -> 403 FORBIDDEN: requires re-login
      - IF revoked_at == NULL: normal refresh
        - revoked_at = NOW
        - generate new accessToken and refreshToken
        -> 200 OK: tokens
      - ELSE: (revoked_at != NULL):
        - check grace period, time_diff = NOW - revoked_at
        - IF time_diff < GRACE_PERIOD (e.g. 15 seconds):
          - generate new accessToken and refreshToken
          - NOTE: if two requests hit server, I generate two token pairs, and the app will try to write both to secure storage. If timing is unlucky, app might store access token 1 and refresh token 2. Client-side queue should prevent this.
          -> 200 OK: tokens
        - ELSE: (time_diff > GRACE_PERIOD) -- stolen refresh token
          - revoke all refresh tokens for this user_id immediately
          -> 403 FORBIDDEN: requires re-login
    - ELSE: throw error 'Invalid refreshToken' -> 401 UNAUTHORIZED

ROUTE: POST /auth/logout
  VALIDATE:
    - refreshToken is a string
    - refreshToken is not empty
  CONTROLLER LOGIC:
    - Verify refreshToken using jwt library
    - IF valid: invalidate refreshToken -> 200 OK
    - ELSE: throw error 'Invalid refreshToken' -> 401 UNAUTHORIZED

CRONJOB:
- clean up expired refresh tokens which expired more than 7 days ago

MIDDLEWARE: authMiddleware
- CHECK HEADERS
  - IF no Authorization header -> 401 UNAUTHORIZED
- EXTRACT TOKEN
- VERIFY TOKEN
  - decode token with ACCESS_TOKEN_SECRET
  - CHECK signature
  - CHECK expiration
  - IF invalid: -> 401 UNAUTHORIZED
- ATTACH TOKEN to request
- PROCEED (next)