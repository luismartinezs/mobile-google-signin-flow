IMPORT
- axios

apiClient = axios.create({baseUrl})

STATE
- isRefreshing = false
- failedQueue = []

axios request interceptor
- IF 401 error (access token not valid)
  - IF isRefreshing:
    - create a "handle" that allows request to be resumed later
    - push handle to failedQueue
    - freeze request (do nothing)
  - ELSE:
    - isRefreshing = true
    - API POST /api/auth/refresh
    - IF success:
      - update accessToken and refreshToken in secure storage
      - unlock refresh mechanism: isRefreshing = false
      - retry waitlisted calls, for each handle in failedQueue:
        - unfreeze request: pass it the new token and retry them
      - clear failedQueue
      - retry current request
    - ELSE:
      - isRefreshing = false, reject all waitlisted calls
      - do NOT call signout endpoint as it will trigger an infinite loop. signOut clientside only (AuthProvider)

AuthContext
COMPONENT AuthProvider
  PROPS: children

  STATE:
  - userToken = null
  - isLoading = true

  ONMOUNT (EFFECT):
  - retrieve access token
    - Check SecureStore for a saved token.
    - IF found: Update userToken state.
    - IF not found: Leave userToken as null.
    - Set isLoading = false.
  - setup axios interceptor here so it has closure access to signOut
    - IF refresh fails, call signOut()
    -> cleanup interceptor on unmount

  HANDLER:
  - signIn (accessToken, refreshToken)
    - save accessToken and refreshToken in SecureStore
    - userToken = accessToken, isLoading = false
  - signOut:
    - delete tokens from SecureStore
    - userToken = null

  CONTEXT:
  - handlers
  - userToken
  - isLoading

  RENDER:
  <AuthContext.Provider value={contextValue}>
    {children}
  </AuthContext.Provider>

HOOK useAuth -> useContext(AuthContext)