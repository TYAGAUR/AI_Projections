# Day 14: Logging in and protecting your API

This guide explains `note14` in beginner-friendly language. It brings together the backend pieces from the earlier lessons and adds user registration, login, and protected routes.

## What this lesson adds

By the end, the AI Platform Gateway can:

- Register users without saving their passwords as readable text.
- Check a user's email and password when they log in.
- Give a successful user a signed access token.
- Require that token before allowing access to protected routes.
- Connect FastAPI, Pydantic, SQLAlchemy, and PostgreSQL in one workflow.

The AI response is still simulated. This lesson adds authentication; it does not yet connect the app to a real AI model.

## The parts of the app

Think of the app as a set of helpers with different jobs:

- **FastAPI** receives web requests and sends responses.
- **Pydantic** checks that the data in a request has the expected shape.
- **Service code** handles tasks such as registering or logging in a user.
- **SQLAlchemy** lets Python code read and write database records.
- **PostgreSQL** stores users and request history.

When someone visits a protected route, the app checks their access token before running the route's main job.

## Passwords: save a hash, not the password

The app should never store a user's actual password in the database. Instead, it uses **password hashing**:

1. When a user registers, the app turns their password into a hard-to-reverse hash.
2. The database stores that hash.
3. When they log in, the app checks the password they entered against the saved hash.

This lesson uses Argon2 through the `pwdlib` package. Hashing is not the same as encrypting: the app verifies a password without needing to turn the saved hash back into the original password.

The public registration response leaves out the password and its hash.

## JWT: a temporary proof of login

After a successful login, the app creates a **JWT** (JSON Web Token). The client sends it with later requests:

```text
Authorization: Bearer YOUR_ACCESS_TOKEN
```

The token has three parts: a header, a payload, and a signature. In this lesson, its payload includes the user's ID and an expiration time.

Important: the payload is encoded, not encrypted. Someone who gets the token may be able to read its contents. Never put passwords, API keys, or other secrets in it.

The signature helps the app detect whether the token was changed. The expiration time means the token should stop working after its allowed lifetime.

## The login journey

Here is the whole process in plain language:

1. A user registers with a name, email, and password.
2. The app saves the name, email, and password hash.
3. The user logs in with their email and password.
4. The app checks the password against the stored hash.
5. If it matches, the app gives the user a signed JWT.
6. The user sends that token with requests to protected routes.
7. FastAPI checks the token and looks up the user before allowing access.

For the login form in Swagger, type the email in the field labelled `username`. That field name comes from the OAuth2 form format.

## New authentication files and settings

The lesson adds:

- `config/security.py` for password hashing and token creation/verification;
- `services/auth_service.py` for registering and checking users;
- `api/auth.py` for the register, login, and profile routes.

It also adds configuration values to `.env`:

- a JWT secret key;
- a signing algorithm;
- the access-token lifetime in minutes.

Generate a long random secret as shown in the lesson. Keep `.env` out of GitHub. Do not copy real passwords or secrets into source code, examples, or public files.

The lesson also adds a `password_hash` column to the existing `users` table. Creating the SQLAlchemy model does not add a column to a database table that already exists, so apply the provided database migration/SQL change before using the updated model.

## New authentication routes

| Route | What it does |
| --- | --- |
| `POST /auth/register` | Creates a user and returns public details, not the password |
| `POST /auth/token` | Checks login details and returns an access token |
| `GET /auth/me` | Returns the profile for the user represented by the token |

If an email is already registered, registration returns **409 Conflict**. If login details are incorrect, the API returns **401 Unauthorized**.

## Protecting an existing route

FastAPI can use a dependency called `get_current_user` to check the token and find the matching user. Adding it to a route makes the route require a valid token.

For example, the lesson protects request history. Without a valid token, the request should receive **401 Unauthorized**.

But there is an important extra step: **authentication is not the same as authorization**.

- **Authentication** asks, “Who are you?”
- **Authorization** asks, “Are you allowed to see or change this?”

The lesson points out that simply requiring a token does not automatically limit history to that user's records. A multi-user app must also filter records by the signed-in user's ID and attach new requests to that user. The lesson shows this as a follow-up improvement.

## Try it in Swagger

Start the app from the project folder:

```powershell
fastapi dev main.py
```

Open `http://127.0.0.1:8000/docs`, then:

1. Use `POST /auth/register` to create a test user.
2. Use `POST /auth/token` to log in. Enter the email in the `username` field.
3. Click Swagger's **Authorize** button and provide the login details.
4. Call `GET /auth/me` to see the logged-in user's public profile.
5. Try a protected route with a token, without a token, and with an invalid or expired token.
6. Try registering the same email twice; the second attempt should return 409.

The profile response should not include a password or password hash.

## Status codes to remember

- **401 Unauthorized**: the user is not logged in, or their token is missing, invalid, or expired.
- **403 Forbidden**: the user is logged in, but does not have permission for that action.
- **404 Not Found**: the requested record does not exist.
- **409 Conflict**: the request conflicts with existing data, such as a duplicate email.
- **422 Unprocessable Entity**: the submitted data failed validation.

## Security reminders

- Use HTTPS when deploying an app so tokens and passwords are protected in transit.
- Keep JWT secrets and database passwords out of GitHub.
- Use short-lived access tokens and plan token renewal carefully.
- Add limits to login attempts to make password guessing harder.
- Check permissions and filter data by user on every private route.
- Use database migrations, such as Alembic, to change existing tables safely.
- Do not show private internal error details to users.

## Week 2 review

This lesson ties together the main backend ideas from the week:

- Pydantic validates input and shapes output.
- FastAPI defines the routes and dependencies.
- SQLAlchemy maps Python objects to database records.
- PostgreSQL stores data after the app stops.
- Password hashing protects stored passwords.
- JWTs let the app recognize logged-in users on later requests.

## Practice ideas

Try implementing and testing these pieces yourself:

1. Register a user and log in to receive a token.
2. Require a token on a protected route.
3. Return only the signed-in user's request history.
4. Save new requests with the signed-in user's ID.
5. Handle duplicate emails and invalid tokens.

The next lesson moves from simulated responses to connecting the app to a real AI provider. Keep API keys in environment variables and handle provider errors, timeouts, and usage costs carefully.
