MIDDLEWARE: for backends that can get requests from mobile and from web
- do not run express-session for mobile requests: create a wrapper middleware that skips this express-session if the request has a bearer token (jwt flow)
- setup cors to allow requests with not origin, e.g. mobile or curl
- helmet does not interfere with mobile requests, keep as is
- authMiddleware:
  - must be hybrid middleware with two paths: serversession and jwt
  - IF header has bearer token:
    - jwt flow
  - IF request has session
    - serversession flow
  - ELSE: 401 UNAUTHORIZED

ROUTER:
- authRouter

