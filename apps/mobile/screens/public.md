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
  EFFECT: -
  HANDLER: signInWithGoogle
    TRY:
      - await GoogleSignin.hasPlayServices()
      - userInfo = await GoogleSignin.signIn()
      - send userInfo.idToken to HTTPS POST /api/auth/google
      - retrieve accessToken and refreshToken from response
      - useAuth().signIn(newToken)
    CATCH:
      - show error message
  RENDER:
    <GoogleLogoButton onPress={signInWithGoogle}>Google Sign in</GoogleLogoButton>