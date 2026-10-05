# Security Audit – Recipe Box API

Date: (fill in today’s date)

## 1. Password Storage

- The `/register` endpoint accepts `username`, `email`, `password`, and optional `display_name`.
- Passwords are never stored in plaintext:
  - `hash_password()` uses `werkzeug.security.generate_password_hash(password)` to compute a one‑way hash.
  - The `users` table stores only `password_hash`, not the original password.
- The `/register` response includes only:
  - `id`, `username`, `email`, `display_name`
  - It does **not** include the plaintext password or `password_hash`.
- The `__main__` test block confirms hashing behavior:
  - `verify_password(test_hash, "test-password-123")` returns `True`.
  - `verify_password(test_hash, "not-the-password")` returns `False`.

**Result:** Passwords are stored securely as one‑way hashes and are never exposed via the API.

---

## 2. Authentication & Identity (JWT)

- Environment:
  - `JWT_SECRET_KEY` is loaded from `.env`. The app raises a `RuntimeError` if it is missing.
  - `JWT_ALGORITHM` is set to `"HS256"`.
- **Login flow – `/login`**:
  - Requires `username` and `password`. Missing fields → `400 {"error": "username and password are required"}`.
  - Looks up the user by `username` in the `users` table, retrieving `password_hash` and `role`.
  - Uses `verify_password()` (`check_password_hash`) to validate the password.
  - On failure (no such user or bad password), returns:
    - `401 {"error": "invalid username or password"}`
    - The error is the same for “user not found” and “wrong password” to avoid leaking which part is wrong.
  - On success:
    - Builds a non‑secret `user` payload: `id`, `username`, `email`, `display_name`.
    - Builds JWT claims:
      - `sub`: user `id` (identity)
      - `username`
      - `role`
      - `exp`: `now + 1 hour`
    - Signs the token using `jwt.encode(claims, JWT_SECRET_KEY, algorithm="HS256")`.
    - Returns `200` with:
      ```json
      {
        "user": { ...non-secret fields... },
        "token": "<JWT>"
      }
      ```

**Result:** Identity is established by username/password at `/login`, and represented by a signed, expiring JWT including `sub` (user ID) and `role`.

---

## 3. Central Authentication Guard

- `get_token_from_header()`:
  - Reads `Authorization` header.
  - If it does not start with `Bearer `, returns `None`.
- `require_token()`:
  - Uses `get_token_from_header()` to read `Bearer <token>`.
  - If no valid `Bearer` header is present:
    - Returns `401 {"error": "missing or malformed Authorization header"}`.
  - Decodes the token via:
    - `jwt.decode(token, JWT_SECRET_KEY, algorithms=[JWT_ALGORITHM])`.
  - Handles failure cases:
    - Expired token → `401 {"error": "token expired, please log in again"}`.
    - Any other invalid token → `401 {"error": "invalid token"}`.
  - On success:
    - Returns `(payload, None)` where `payload` contains `sub`, `username`, `role`, and `exp`.

**Usage:**

- `PATCH /recipes/<id>` and `DELETE /recipes/<id>` both call `require_token()` at the start:
  - If `error_response` is not `None`, they immediately return that `(json, status)` tuple (401).

**Result:** Authentication is centralized in `require_token()` for modifying/deleting recipes. Missing/expired/invalid tokens consistently yield 401 responses with clear error messages.

---

## 4. Access Control: Ownership and Roles

### 4.1 Creating Recipes – `POST /recipes`

- This route manually enforces authentication (inline):
  - Reads `Authorization` header and expects `Bearer <token>`.
  - If missing or not starting with `Bearer `:
    - Returns `401 {"error": "missing or invalid Authorization header"}`.
  - Decodes the JWT using `jwt.decode(token, JWT_SECRET_KEY, algorithms=["HS256"])`.
  - On `ExpiredSignatureError`:
    - Returns `401 {"error": "token expired, please log in again"}`.
  - On `InvalidTokenError`:
    - Returns `401 {"error": "invalid token"}`.
- On success:
  - Extracts `user_id = payload["sub"]`.
  - Validates body: requires `title` and `ingredients`; else `400 {"error": "title and ingredients are required"}`.
  - Inserts new recipe with:
    - `owner_id = user_id`
    - `is_public` based on the incoming `is_public` flag (defaults to `True`).
- Responds with `201` and the new recipe JSON (via `recipe_to_dict`).

**Result:** Only authenticated users with a valid, non‑expired JWT can create recipes; each recipe is owned by the `sub` (user ID) in the token.

---

### 4.2 Updating Recipes – `PATCH /recipes/<id>`

- Begins with:
  - `payload, error_response = require_token()`
  - If `error_response` is not `None` → returns the appropriate `401` response.
- Uses `payload["sub"]` as `user_id` and `payload.get("role")` as `user_role`.
- Checks database:
  - If no recipe with that `id` → `404 {"error": "recipe not found"}`.
- Authorization rule:
  - If `recipe["owner_id"] != user_id` **and** `user_role != "admin"`:
    - Returns `403 {"error": "forbidden"}`.
  - Otherwise, the user is allowed to update (owner or admin).
- (Business logic for the actual update happens after these checks.)

**Result:** Only the owner of a recipe or a user with `role = "admin"` can update it. Unauthorized authenticated users get `403` rather than `401`.

---

### 4.3 Deleting Recipes – `DELETE /recipes/<id>`

- Same authentication pattern:
  - `payload, error_response = require_token()`; returns `401` for missing/expired/invalid JWT.
- Fetches the recipe:
  - If not found → `404 {"error": "recipe not found"}`.
- Authorization rule:
  - If `recipe["owner_id"] != user_id` **and** `user_role != "admin"`:
    - Returns `403 {"error": "forbidden"}`.
- On success:
  - Deletes the recipe and returns `204` with an empty body.

**Result:** Destructive actions are restricted to owners and admins; cross‑user delete attempts get a 403.

---

## 5. Public Endpoints

- `GET /`:
  - Public welcome route with a simple JSON message.
- `GET /recipes`:
  - Public listing of all recipes: `SELECT * FROM recipes ORDER BY id`.
  - No authentication or ownership checks.
- `GET /recipes/<id>`:
  - Public read of a single recipe:
    - `404 {"error": "recipe not found"}` if it doesn’t exist.
  - No authentication or ownership checks.

**Result:** Reading recipes is currently public. Any client can list or read recipes without a token.

---

## 6. Centralization & Remaining Gaps

- **Centralized auth**:
  - `require_token()` provides a single place for JWT validation and is used by:
    - `PATCH /recipes/<id>`
    - `DELETE /recipes/<id>`
- **Inline auth**:
  - `POST /recipes` performs its own header parsing and JWT decode logic instead of calling `require_token()`. This duplicates auth behavior and could diverge if the helper changes.
- **Roles**:
  - JWT includes a `role` claim.
  - The `role` is used to allow admins to update or delete any recipe.
  - There are no dedicated admin-only endpoints beyond this (e.g., admin-only listing).
- **Known limitations / not implemented**:
  - No rate limiting.
  - No HTTPS/TLS configuration in this app (assumed to be handled by deployment environment).
  - No password reset flows or email verification.
  - Reading recipes is public; there is no per-user visibility restriction on `GET /recipes` or `GET /recipes/<id>`.

---

## 7. Summary

- **Safe storage**: Passwords are stored as secure one‑way hashes and never exposed in API responses.
- **Authentication**: `/login` uses username/password with hashed verification and issues signed, expiring JWTs.
- **Identity enforcement**: `require_token()` and inline JWT checks ensure that modifying and deleting recipes require a valid, non‑expired token.
- **Access control**: Ownership and `role` claims are enforced:
  - Only the owner or an admin can update/delete a recipe.
- **Public data**: Listing and reading recipes are intentionally unauthenticated.
- **Gaps**: Some auth logic is duplicated; public read access may or may not match desired policy; no advanced controls like rate limiting or password reset are implemented.