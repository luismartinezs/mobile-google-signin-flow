PrivateStack

IMPORT useAuth

COMPONENT PrivateScreen
  PROPS: -
  STATE: -
  EFFECT: -
  HANDLER:
    logout:
    - useAuth().signOut()
  RENDER:
    <LogoutButton onPress={logout} />