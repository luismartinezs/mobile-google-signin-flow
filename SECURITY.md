Mobile apps are invulnerable to XXS or CSRF attacks.

However, the mobile auth flow still needs PKCE, which is usually handled by the google auth SDK

Mobile apps are vulnerable if server secrets are hardcoded and used in the mobile app, so never do that. Do not ever put server secrets in the .env file of the mobile app.

If the mobile app needs to do something that requires a server secret, send a request to the backend and let the backend do it.