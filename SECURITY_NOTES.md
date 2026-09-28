# Security baseline – 2026-09-28

- Anonymous requester can **read** recipes:
  - `GET /recipes` → returns a list of recipes including recipe `id: 1` (Shakshuka).

- Anonymous requester can **update** a recipe:
  - `PATCH /recipes/1` with `{"title": "Totally Hacked Shakshuka"}` → returns the updated recipe showing the new title.

- Anonymous requester can **delete** a recipe:
  - First `DELETE /recipes/1` → recipe is removed.
  - Reloading `GET /recipes` shows recipe `id: 1` is gone.
  - Second `DELETE /recipes/1` → “no recipe found” (because it was already deleted).