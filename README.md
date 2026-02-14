
1. **Add a co-authors junction table via a new MikroORM migration.** Create an `article_co_authors` table with `article_id` and `user_id` foreign keys to establish a many-to-many relationship between articles and users.

2. **Add article locking columns via the same migration.** Add `locked_by` (nullable FK to `user`) and `locked_at` (nullable datetime) columns to the `article` table to support the ADVANCED locking mechanism.

3. **Update the Article entity** to include a `@ManyToMany` co-authors collection and `lockedBy`/`lockedAt` properties. Update the `toJSON` method to serialize co-authors.

4. **Update the User entity** to include the inverse side of the co-authors `@ManyToMany` relationship.

5. **Update the CreateArticleDto** to accept an optional `coAuthorIds: number[]` field.

6. **Update the Article service:**
   - `create()`: Accept and persist co-author user IDs when creating an article.
   - `update()`: Allow co-authors (not just the original author) to update articles.
   - Add `lockArticle()` / `unlockArticle()` methods that set/clear the `lockedBy` and `lockedAt` fields.
   - Add a `heartbeat()` method to refresh `lockedAt`, keeping the lock alive while the user is editing.
   - Before allowing edits, check the lock: if another user holds an unexpired lock (< 5 minutes since `lockedAt`), return an error.

7. **Update the Article controller:**
   - Add `PUT /articles/:slug/lock` endpoint to acquire the lock.
   - Add `DELETE /articles/:slug/lock` endpoint to release the lock.
   - Add `POST /articles/:slug/heartbeat` endpoint to refresh the lock.
   - Register these new routes with the auth middleware.

8. **Expose a user list API for the dropdown.** The existing `GET /users` endpoint already returns all users with pagination — use this for the co-author multi-select dropdown.

9. **Update frontend TypeScript types:** Extend `Article` to include `coAuthors: Profile[]`. Extend `ArticleForEditor` to include `coAuthorIds: number[]`.

10. **Update the frontend article decoder** to handle the new `coAuthors` field from the API response.

11. **Add frontend API functions:** Add `lockArticle()`, `unlockArticle()`, `heartbeat()`, and `getUsers()` service functions.

12. **Update the ArticleEditor component and its Redux slice:**
    - Add a multi-select dropdown populated from the user list API to select co-authors by username.
    - Add slice actions for adding/removing co-authors.

13. **Update the EditArticle page:**
    - Allow co-authors (not just the original author) to access the edit page.
    - On mount, call `lockArticle()`. If the lock is held by another user, show an error message and prevent editing.
    - Start a periodic heartbeat (every 60 seconds) to keep the lock alive.
    - On unmount (navigate away) or save, call `unlockArticle()`.
    - If a heartbeat fails (lock lost), show an error and redirect the user away.

14. **Manual acceptance testing:** Run the three acceptance tests, take screenshots, and place them in the `submission/` folder.

## Decisions

- **Decision: Use a `article_co_authors` junction table (ManyToMany) for co-authorship.**
  - Alternative: Store co-author IDs as a JSON array column on the `article` table (similar to how `tagList` is stored).
  - Alternative: Store co-author emails as a comma-separated string column.
  - Rationale: A proper junction table provides referential integrity via foreign keys, allows efficient querying from either direction (e.g., "find all articles I co-author"), and follows the same relational pattern already used in the codebase for `user_to_follower` and `user_favorites`. JSON arrays or comma-separated strings would make it difficult to join or query and would bypass database-level integrity checks.

- **Decision: Use database columns (`locked_by`, `locked_at`) on the `article` table with HTTP-polling heartbeat for the locking mechanism.**
  - Alternative: Use a separate `article_locks` table to track locks independently from articles.
  - Alternative: Use WebSockets for real-time lock status updates instead of HTTP polling.
  - Rationale: Since the backend runs on ECS Fargate with 1–10 auto-scaled instances behind a load balancer, all state must be in the database — no in-memory or WebSocket-based solutions would work reliably without sticky sessions or a shared pub/sub layer (e.g., Redis), which is not part of the current architecture. Adding columns directly to the `article` table is simpler than a separate table since the lock is a 1:1 relationship with the article and avoids an extra join. A 60-second HTTP heartbeat is lightweight, stateless, and compatible with the load-balanced setup.

- **Decision: Define "online" as having the edit page open (heartbeat-based), with a 5-minute expiry.**
  - Alternative: Track active keystrokes/mouse events and only count the user as "online" when actively editing.
  - Rationale: A heartbeat sent while the edit page is open is simpler, more reliable, and avoids edge cases where a user is reading/thinking but not actively typing. The 5-minute expiry after the last heartbeat naturally handles cases where the user closes the browser, loses connection, or navigates away without explicitly unlocking.

## Notes

- **No AWS architecture changes required.** The locking mechanism uses database columns and HTTP polling, both of which work within the existing CloudFront → ALB → ECS Fargate → Aurora MySQL architecture. No new infrastructure (Redis, WebSocket servers, etc.) is needed.
- **Lock expiry is handled server-side.** When checking if an article is locked, the backend compares `locked_at` against the current time. If more than 5 minutes have passed, the lock is considered expired and a new user can acquire it. This avoids relying on client-side timers.
- **The existing `GET /users` endpoint** already supports pagination and returns user data suitable for the multi-select dropdown — no new endpoint is needed for listing users.
