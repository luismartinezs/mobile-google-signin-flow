table refresh_tokens
  id
  user_id: needed to revoke all refresh tokens for a user when they log out
  token_hash: hashed refresh token
  expires_at
  revoked_at
  created_at
indexes: token_hash, expires_at, user_id