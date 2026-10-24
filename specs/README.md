# Varsha Customer Service — Specifications

## Project Purpose

This microservice provides backend capabilities for customer authentication, shopping carts, checkout, orders, payments, delivery tracking, and customer feedback for Varsha Homemade (varsha-homemade.com).

## Business Flow

User → Cart → Checkout → Order → Payment → Delivery → Feedback

1. A customer authenticates or continues as a guest, where supported by the approved requirements.
2. The customer views products and adds items to a cart.
3. The customer reviews the cart and proceeds to checkout.
4. The system validates the cart and prepares the purchase.
5. An order is created according to the approved order and payment rules.
6. Payment is processed and verified through the selected payment provider.
7. Delivery information and status are tracked.
8. The customer can submit feedback when eligible.

**Decisions to be confirmed in the relevant domain specifications:**
- Whether guest checkout is supported (User, Cart, Checkout).
- Which payment provider is used (Payment).
- Any other behavior not explicitly specified here.

None of these are finalized.

## Domain Boundaries

These boundaries are provisional and can be refined through approved specifications.

| Domain | Responsibility |
|---|---|
| 01 — User & Authentication | Customer identity, Google authentication, JWT, and account-related behavior. |
| 02 — Cart | Cart creation, cart items, cart retrieval, and cart modification. |
| 03 — Checkout | Cart validation and purchase preparation. |
| 04 — Order | Order creation, order details, and purchase history. |
| 05 — Payment | Payment initiation, verification, transaction records, and payment status. |
| 06 — Delivery | Shipping details, delivery status, and tracking. |
| 07 — Feedback | Customer feedback, ratings, and eligibility rules. |
| 08 — Integration | Cross-domain coordination, external service integrations, and final system-level verification. |

## SDD Principles

Mandatory rules:

1. Requirements must be documented before implementation begins.
2. Each domain must have clear requirements and acceptance criteria.
3. API contracts and data models must be reviewed before implementing the relevant domain.
4. Implementation must follow the approved specifications.
5. Do not invent business rules when requirements are unclear.
6. Record unresolved questions, assumptions, dependencies, and conflicts for review.
7. Update specifications when an agreed requirement changes.
8. Verify implementation against acceptance criteria before declaring a domain complete.
9. Do not modify unrelated functionality or rewrite existing code unnecessarily.
10. Do not start the next domain until the current domain has been reviewed and its implementation verified, unless explicitly instructed otherwise.
11. Treat security, authorization, data validation, error handling, and sensitive-data protection as requirements to be addressed in the appropriate specifications.
12. Clearly distinguish confirmed requirements from proposed solutions and unresolved decisions.

## Specification Workflow

For each domain, follow this sequence:

1. Gather and document requirements.
2. Identify questions, assumptions, and unresolved decisions.
3. Define API contracts.
4. Define the data model and database requirements.
5. Document business rules and acceptance criteria.
6. Review the complete specification with the project owner.
7. Make changes based on feedback.
8. Wait for approval before implementation.
9. Implement only the approved scope.
10. Verify the implementation against the specification.
11. Report results and unresolved issues before moving forward.

## Domain Implementation Sequence

1. User & Authentication
2. Cart
3. Checkout
4. Order
5. Payment
6. Delivery
7. Feedback
8. Integration & Final Verification

Dependencies may be identified during specification, but the implementation sequence must not be changed without discussion and approval.
