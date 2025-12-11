IMPORT AuthProvider, useAuth

COMPONENT App
  RENDER:
    <AuthProvider>
      <RootNavigator />
    </AuthProvider>

COMPONENT RootNavigator
  STATE:
  - isLoading, userToken = useAuth()

  RENDER:
  - CASE A: isLoading
    -> SplashScreen (logo + spinner)
  - CASE B: userToken
    -> PrivateStack
  - CASE C: !userToken
    -> PublicStack
