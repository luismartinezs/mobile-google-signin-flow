Contains the logic (the queue, the refreshing state, and the function that handles the 401). It imports the client to manipulate it

This file holds the complex state (isRefreshing, failedQueue) inside a closure. It exports a setup function

PSEUDOCODE:

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

TYPESCRIPT CODE:
```ts
import { apiClient, setClientToken } from '@/shared/api/client';
import * as SecureStore from 'expo-secure-store';

// 1. PRIVATE STATE (Closed scope)
let isRefreshing = false;
let failedQueue = [];

// Helper to process queue
const processQueue = (error, token = null) => {
  failedQueue.forEach((prom) => {
    if (error) prom.reject(error);
    else prom.resolve(token);
  });
  failedQueue = [];
};

// 2. THE SETUP FUNCTION
// We accept 'signOut' as a parameter to avoid circular dependencies with Context
export const setupAuthInterceptors = (signOutCallback) => {

  // Create the interceptor
  const interceptorId = apiClient.interceptors.response.use(
    (response) => response,
    async (error) => {
      const originalRequest = error.config;

      // IF 401 and NOT already retried
      if (error.response?.status === 401 && !originalRequest._retry) {

        if (isRefreshing) {
          // QUEUE LOGIC
          return new Promise((resolve, reject) => {
            failedQueue.push({ resolve, reject });
          })
          .then((token) => {
            originalRequest.headers['Authorization'] = 'Bearer ' + token;
            return apiClient(originalRequest);
          })
          .catch((err) => Promise.reject(err));
        }

        originalRequest._retry = true;
        isRefreshing = true;

        try {
          // REFRESH LOGIC
          const oldRefreshToken = await SecureStore.getItemAsync('refreshToken');

          // Note: use a separate axios instance or fetch to avoid interceptor loops
          const response = await axios.post('URL/auth/refresh', { refreshToken: oldRefreshToken });

          const { accessToken, refreshToken } = response.data;

          // UPDATE STORAGE & HEADERS
          await SecureStore.setItemAsync('accessToken', accessToken);
          await SecureStore.setItemAsync('refreshToken', refreshToken);
          setClientToken(accessToken);

          // FLUSH QUEUE
          processQueue(null, accessToken);
          isRefreshing = false;

          // RETRY ORIGINAL
          originalRequest.headers['Authorization'] = 'Bearer ' + accessToken;
          return apiClient(originalRequest);

        } catch (refreshErr) {
          // FAIL LOGIC
          processQueue(refreshErr, null);
          isRefreshing = false;

          // CALL THE CALLBACK
          signOutCallback();

          return Promise.reject(refreshErr);
        }
      }

      return Promise.reject(error);
    }
  );

  // Return a cleanup function (to eject interceptor on unmount)
  return () => {
    apiClient.interceptors.response.eject(interceptorId);
  };
};