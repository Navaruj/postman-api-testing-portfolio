# JWT Authentication Test Cases

## Scope

This document covers the JWT authentication flow exercised by the Postman collection.

| ID | Scenario | Preconditions | Expected Result |
| --- | --- | --- | --- |
| API-AUTH-001 | Login with valid credentials | A local environment contains valid demo credentials. | `POST /auth/login` returns `201 Created` and provides an access token. |
| API-AUTH-002 | Retrieve profile with a valid token | The Login request has stored `accessToken` in the active environment. | `GET /auth/profile` returns `200 OK` and the returned email matches `demoEmail`. |
| API-AUTH-003 | Retrieve profile without a token | No Authorization header is sent. | `GET /auth/profile` returns `401 Unauthorized`. |

## Execution Order

Run the requests in this order:

1. `POST Login - Obtain JWT Tokens`
2. `GET Authenticated User Profile`
3. `GET User Profile - Missing Token (Negative)`

The first request captures `accessToken`, which the authenticated-profile request sends as a Bearer token.
