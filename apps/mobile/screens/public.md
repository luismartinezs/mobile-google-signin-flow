PublicStack

IMPORTS
  - GoogleLogoButton, GoogleSignin
    - @react-native-google-signin/google-signin
  - useAuth

GoogleSignin.configure({
  webClientId: 'YOUR_WEB_CLIENT_ID',
})

COMPONENT SignInScreen
  PROPS: -

  STATE:
  - isSigningIn = false

  HOOKS
  - {isLoggingIn, error, signInWithGoogle} = useAuth()

  EFFECT: -

  HANDLER: handleGoogleButtonPress
    isSigningIn = true
    TRY:
      - await GoogleSignin.hasPlayServices()
      - userInfo = await GoogleSignin.signIn()
      - signInWithGoogle(userInfo.idToken)
    CATCH:
      - show error message
    FINALLY:
      - isSigningIn = false

  RENDER:
    <GoogleLogoButton onPress={handleGoogleButtonPress}>Google Sign in</GoogleLogoButton>