# Varsha Customer Service

## Purpose

Backend microservice for Varsha Homemade (varsha-homemade.com), providing customer authentication, shopping carts, checkout, orders, payments, delivery tracking, and customer feedback.

## Technology Stack

- Backend runtime: Node.js; framework: Express.js; language: JavaScript (no TypeScript)
- Database: PostgreSQL, hosted on Supabase and accessed directly with a standard PostgreSQL driver (the Supabase client library and Data API are not used)
- Frontend: React, hosted on Render; the backend is also hosted on Render
- API style: REST
- Authentication: Google sign-in; short-lived JWT access tokens and opaque, single-use, rotating refresh tokens issued by the Express service
- Configuration: environment variables and server-side secret configuration (no secrets in source control)
- No Docker; no unit tests at this stage

## Project Status

The project is in the **specification phase**. Authentication specifications exist under [specs/01-user/](specs/01-user/); the other domains have no specifications yet. **No application code, dependency manifest or database migration exists in this repository.** Dependencies have been approved in the specifications but are not installed.

## Specification-Driven Development

This project follows SDD. See [specs/README.md](specs/README.md) for the workflow and the document index, and [specs/decisions/](specs/decisions/) for decision records.

Application implementation begins only after the relevant specifications have been reviewed and approved. Implementation is not currently authorized.
