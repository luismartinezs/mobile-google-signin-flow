# Google Sign-In & Authentication Architecture

This document outlines the complete authentication flow for the React Native (Expo) mobile application and the corresponding backend service. It implements **OpenID Connect** for identity verification and **OAuth 2.0** best practices (Access/Refresh tokens) for session management.

---

## 1. Quick Setup Outline

### A. Google Cloud Console
* **Project Creation:** A Google Cloud Project is required to act as the OAuth provider.
* **Android Client ID:** Required to verify the specific Android app binary (requires SHA-1 fingerprint from Upload Key and Play Store Signing Key).
* **Web Client ID:** Required to represent the backend server. This ID is used by the mobile app to request an `idToken` that the backend can verify.

### B. Mobile App (Expo)
* **Dependencies:** Uses `@react-native-google-signin/google-signin` for native Google interaction and `axios` for networking.
* **Storage:** Uses `expo-secure-store` to encrypt tokens on the device.
* **Navigation:** Uses Conditional Navigation (Public vs. Private stacks) driven by React Context.

### C. Backend API
* **Validation:** Uses `google-auth-library` to verify the integrity of the Google ID Token.
* **Session:** Issues short-lived JWT Access Tokens (15 min) and long-lived JWT Refresh Tokens (7 days).
* **Security:** Implements Refresh Token Rotation with a server-side grace period to prevent race conditions.

---

## 2. Detailed Authentication Flow

### Phase 1: Initial Sign-In
This phase covers the user's interaction with the "Sign In" button and the initial exchange of credentials.

1.  **User Interaction:**
    The user clicks the "Sign in with Google" button on the `SignInScreen` (located in `mobile/screens/public.md`).

2.  **Google SDK Exchange:**
    The mobile app triggers the native Google SDK. The OS presents a system-level bottom sheet asking the user to select their Google account.
    * *Configuration:* The app requests the `idToken` specifically for the **Web Client ID** (configured in `mobile/config.md`).

3.  **Token Transmission:**
    Upon successful user selection, Google returns an `idToken` to the mobile app. The app immediately sends this token via an HTTPS POST request to the backend endpoint `/auth/google/verify` (defined in `backend/features/auth.md`).

### Phase 2: Backend Verification & Token Issuance
This phase occurs entirely on the server to establish trust and create the user session.

1.  **Token Verification:**
    The backend receives the `idToken`. It uses the Google Auth Library to verify the token's digital signature against Google's public keys.
    * *Audience Check:* It confirms the `aud` claim matches the backend's Client ID to ensure the token wasn't generated for a different application.

2.  **User Management:**
    * If the Google User ID (sub) is new, a new user record is created in the database.
    * If the user exists, their record is retrieved.

3.  **Session Generation:**
    The backend generates two JSON Web Tokens (JWTs):
    * **Access Token:** Valid for 15 minutes. Contains user roles/permissions.
    * **Refresh Token:** Valid for 7 days. Contains only the user ID.
    * *Database:* The hash of the Refresh Token is stored in the database to allow for future revocation.

4.  **Response:**
    The backend responds with both tokens (HTTP 200).

### Phase 3: Mobile Session Initialization
This phase updates the mobile app's state to reflect the logged-in status.

1.  **Secure Storage:**
    The `AuthContext` (in `mobile/features/auth.md`) receives the tokens. It stores **both** the Access Token and Refresh Token in `SecureStore`. This ensures the session persists across app restarts.

2.  **State Update:**
    The `AuthContext` updates its internal state (`userToken`) to the new Access Token.

3.  **Navigation Switch:**
    The `RootNavigator` (in `mobile/screens/app.md`) detects the state change. It unmounts the **PublicStack** (Login screens) and mounts the **PrivateStack** (Home/Profile screens). The user is now inside the app.

### Phase 4: Token Rotation (The Refresh Loop)
This phase handles the automatic renewal of expired access tokens without interrupting the user.

1.  **Request Interception:**
    The `axios` interceptor (in `mobile/features/auth.md`) monitors every outgoing API request. It attaches the current Access Token to the Authorization header.

2.  **Detecting Expiration:**
    If the backend rejects a request with a **401 Unauthorized** error, the interceptor catches the error before the UI sees it.

3.  **Client-Side Queue (Concurrency Protection):**
    * If a refresh is already in progress (`isRefreshing` is true), the interceptor "freezes" the request and adds it to a waiting queue.
    * If no refresh is in progress, the interceptor sets the `isRefreshing` flag and initiates a request to `/auth/refresh`.

4.  **Backend Rotation Logic:**
    The backend receives the Refresh Token.
    * *Validation:* It checks if the token is valid and exists in the database.
    * *Rotation:* It invalidates the old Refresh Token (sets `revoked_at` to NOW) and issues a completely new Access/Refresh pair.
    * *Grace Period:* If the backend receives an "old" token that was revoked less than 15 seconds ago (due to a network race condition), it permits the refresh to proceed instead of banning the user.

5.  **Retry:**
    The mobile app receives the new tokens, updates `SecureStore`, updates the defaults for future headers, and retries the original failed request (and any queued requests).

### Phase 5: Logout & Revocation
This phase ensures the session is securely terminated.

1.  **User Action:**
    The user taps "Logout" in the `PrivateScreen`.

2.  **Local Cleanup:**
    The `AuthContext` immediately wipes tokens from `SecureStore` and sets the state to null. This forces the `RootNavigator` to switch back to the **PublicStack**.

3.  **Server Revocation:**
    (Optional but recommended) The app sends a final request to `/auth/logout`. The backend finds the Refresh Token in the database and marks it as permanently revoked, ensuring it cannot be used again if it was compromised.




# OTHER CONSIDERATIONS

- Apple tax: on iOS for App Store, if app offers third party login, it MUST also offer Apple login
- Database bloat: setup cronjobs to cleanup refresh tokens expired more than 7 days ago
- Google Play Services & Emulators: Android emulators do not have Google Play Services. Use a system image that has Google Play Store, or test in a real android device
- Existing duped user:
  - request with google email: alice@example.com + google sub: 123
  - does provider: google and id: 123 exist in identities table?
    - YES: log in
    - NO: search users table for email
      - FOUND: create new identity and link to found user
      - NOT FOUND: create new user and new identity