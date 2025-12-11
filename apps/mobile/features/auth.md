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
      - apiClient.defaults.headers.common['Authorization'] = 'Bearer ' + newAccessToken
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
  - isLoggingIn = false
  - error = null

  ONMOUNT (EFFECT):
  - retrieve access token
    - Check SecureStore for a saved token.
    - IF found: Update userToken state.
    - IF not found: Leave userToken as null.
    - Set isLoading = false.
  - setup axios interceptor here so it has closure access to signOut
    - IF refresh fails, call signOut()
    -> cleanup interceptor on unmount

  HANDLERS:
  - signIn (accessToken, refreshToken) // keep this as a general token handler
    - save accessToken and refreshToken in SecureStore
    - userToken = accessToken, isLoading = false
  - signOut:
    - delete tokens from SecureStore
    - userToken = null
  - signInWithGoogle (idToken)
    isLoggingIn = true
    error = null
    TRY:
      - send userInfo.idToken to HTTPS POST /api/auth/google/verify
      - retrieve accessToken and refreshToken from response
      - signIn(accessToken, refreshToken)
    CATCH:
      - set error message
    FINALLY:
      - isLoggingIn = false

  CONTEXT:
  - handlers
  - userToken
  - isLoading
  - isLoggingIn
  - error

  RENDER:
  <AuthContext.Provider value={contextValue}>
    {children}
  </AuthContext.Provider>

HOOK useAuth -> useContext(AuthContext)