privateRouter:
MIDDLEWARE: authMiddleware
ROUTE: GET /private
  - grab user id from access token
  - query DB with user id
  - IF user not found -> 404 NOT FOUND
  - ELSE -> 200 OK {data}
