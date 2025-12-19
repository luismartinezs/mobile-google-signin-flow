Contains only the raw Axios instance. It knows nothing about tokens, queues, or refreshing

This file is clean and reusable by any module

```ts
import axios from 'axios';

// 1. Create the instance
export const apiClient = axios.create({
  baseURL: 'https://api.yourdomain.com',
  headers: {
    'Content-Type': 'application/json',
  },
});

// 2. Optional: Allow other modules to set the token easily
export const setClientToken = (token) => {
  apiClient.defaults.headers.common['Authorization'] = `Bearer ${token}`;
};