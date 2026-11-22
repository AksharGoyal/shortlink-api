# Glossary

Plain-language explanations of the terms used in this project. No prior knowledge assumed. Each entry says what the thing is, why you would care, and **Seen in:** where to find it in this repo or the real world.

## How a web request travels

Imagine a restaurant. Keep this picture in mind for the next four entries.

- **Socket** — the "phone line" between two programs. When you open a website, your browser dials your server's address and port (for example `127.0.0.1:8000`). The socket carries raw bytes back and forth and understands nothing about what they mean.
  **Seen in:** the line "Uvicorn running on http://127.0.0.1:8000" printed when the dev server starts.

- **Web server** — the receptionist. It answers the phone, listens to the raw bytes, and writes a clean order slip: "a `GET` request for `/health`". It later reads your answer back down the phone. It does not cook; it knows nothing about what your app does.
  **Seen in:** `uvicorn`, installed with `fastapi[standard]` in `pyproject.toml`.

- **Framework** — the kitchen's workflow. A framework does the repetitive work (matching a request to the right function, checking the input, turning results into JSON) and calls *your* code at the right moment. You write the recipes, and the framework decides when each one runs. That "it calls you" idea is called *inversion of control*.
  **Seen in:** FastAPI, and the route functions such as `/health`.

- **ASGI** — the standard order-slip format. It is a rule that says how a server hands a request to an app. Because every server and app follows it, you can swap one for another without changing your code. The "A" (asynchronous) means one worker can serve many customers while it waits on something slow, instead of standing idle. Its older cousin, WSGI, handles one thing at a time.
  **Seen in:** the `app` object that the server imports and calls on every request.

## Designing the API

- **Resource-oriented design (REST)** — name the *things*, and let the HTTP method be the action. Instead of `/createLink`, you use `POST /links`. Think of a filing cabinet: `/links` is the whole drawer, `/links/abc123` is one folder.

  | Action | Request |
  |---|---|
  | Create a link | `POST /links` |
  | Read one link | `GET /links/{code}` |
  | List links | `GET /links` |
  | Delete a link | `DELETE /links/{code}` |

  **Seen in:** the routes in `src/shortlink_api/`.

- **Redirect: 301 vs 302** — both say "go to this other address". `301` means "moved for good", so browsers remember it and may stop asking you. `302` means "just for now", so browsers ask you every time. We use `302` so we can count clicks and change a link's target later.
  **Seen in:** the response to `GET /{code}`; browser DevTools → Network → Status column.

- **Idempotency** — a fancy word for "safe to repeat". Pressing an elevator's call button five times is the same as pressing it once. Creating a record five times is not. Networks drop replies, so clients retry, so the server must handle repeats.
  **Seen in:** HTTP method rules in the MDN HTTP reference.

- **Idempotency key** — a unique ticket number the client attaches to a request. If the same ticket shows up again, the server says "I already did that one" and returns the saved result instead of doing it twice.
  **Seen in:** the Stripe API docs on idempotent requests.

- **Pagination** — serving a long list a page at a time, instead of dumping everything in one go. *Offset* style (`?limit=20&offset=40`) is simple, like "skip 40 and give me 20", but slows down on huge lists and can repeat or skip items if new ones arrive. *Cursor* style (`?after=abc`) is like a bookmark: fast and stable, but you can't jump straight to page 50.
  **Seen in:** `GET /links` once it is implemented.

## Security

- **SSRF (Server-Side Request Forgery)** — tricking *your* server into making a request for an attacker. Your server sits inside a trusted network, so it can reach places outsiders cannot, like `http://169.254.169.254/`, an address on cloud machines that hands out credentials. It is like a stranger asking a hotel employee to open a staff-only door for them. The defence is to check where a URL really points before following it.
  **Seen in:** the URL validation for `POST /links`; the OWASP SSRF page.

- **Open redirect** — a site that sends visitors wherever a user asks. A link shortener is exactly that by design, so scammers like to hide bad links behind a trusted-looking domain. We limit the damage by accepting only `http` and `https` links, capping the length, and offering a way to report links.
  **Seen in:** the scheme check on submitted URLs.

- **Enumeration attack** — guessing codes one by one (`/1`, `/2`, `/3`) to read things you were never given. If codes are sequential, anyone can list every link. If they are random, guessing is hopeless.
  **Seen in:** how short codes are generated.

## Packaging and versions

- **Semantic versioning (SemVer)** — a version number `MAJOR.MINOR.PATCH` that tells you how risky an upgrade is:
  - `PATCH` (`1.0.0 → 1.0.1`): a bug fix. Safe.
  - `MINOR` (`1.0.1 → 1.1.0`): a new feature that doesn't break anything. Safe.
  - `MAJOR` (`1.1.0 → 2.0.0`): something changed that can break you. Read the notes.

  A version starting with `0.` means "still unstable, anything may change".
  **Seen in:** `version` in `pyproject.toml`; the Git tags.

- **Breaking change** — an update that makes code that worked yesterday fail today. Renaming a JSON field breaks clients that read the old name. Adding a new optional field usually doesn't, because old clients simply ignore it.
  **Seen in:** `CHANGELOG.md`.

- **Constraint vs lockfile** — two files, two jobs. `pyproject.toml` holds *constraints*: "any `fastapi` version from 0.165.0 upward is fine." `uv.lock` holds the *exact* versions that were installed, like a receipt. Everyone who installs from the receipt gets the same result. Commit both.
  **Seen in:** `pyproject.toml` and `uv.lock`; `uv sync --locked` in CI.

- **Extra** — an optional add-on bundle for a package. `fastapi` alone is the bare framework. `fastapi[standard]` adds the usual extras, such as the server and the dev command. It's convenient, but it means more code installed, and more code to keep up to date.
  **Seen in:** `fastapi[standard]` in `pyproject.toml`; `uv tree --depth 1`.

- **Dependency group** — a list of tools that are for *developing* the project, not for *running* it, such as `pytest` and `ruff`. They are installed on your machine and in CI, but they are left out of what users install.
  **Seen in:** the `dev` group under `[dependency-groups]` in `pyproject.toml`.

## Working habits

- **YAGNI ("you aren't gonna need it")** — don't build or install something just in case. Add it when you actually need it. Every extra piece is something to maintain and keep safe.
  **Seen in:** dependencies added only when a feature needs them.

- **TDD (test-driven development)** — write the test first and watch it fail (*red*), write just enough code to make it pass (*green*), then tidy up (*refactor*). Seeing it fail first proves the test can catch a mistake. A test that has never failed might be testing nothing.
  **Seen in:** `tests/test_health.py` before `src/shortlink_api/main.py` existed.

- **Arrange / Act / Assert** — the three-step shape of a readable test: *arrange* the inputs, *act* by doing the one thing under test, *assert* that the result is what you expect.
  **Seen in:** any function in `tests/`.
