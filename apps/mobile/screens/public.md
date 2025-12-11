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
  STATE: -

  HOOKS:
  - {isLoggingIn, error, signInWithGoogle} = useAuth()

  EFFECT: -

  HANDLER: handleGoogleButtonPress
    TRY:
      - await GoogleSignin.hasPlayServices()
      - userInfo = await GoogleSignin.signIn()
      - await signInWithGoogle(userInfo.idToken)
    CATCH:
      - show error message

  RENDER:
    <GoogleLogoButton onPress={handleGoogleButtonPress}>Google Sign in</GoogleLogoButton>