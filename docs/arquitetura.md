# Architecture

## Overview

A classic split between a React single-page app and a Django REST API, each hosted where it fits best, both behind Cloudflare.

```mermaid
flowchart LR
    U[Customer browser] -->|HTTPS| CF[Cloudflare]
    CF --> FE[React SPA<br/>Cloudflare Pages]
    CF --> API[Django REST API<br/>Render]
    U -->|product images| R2[(Cloudflare R2)]
    API --> DB[(PostgreSQL<br/>Supabase)]
    API --> R2
    API -->|payment link| IP[InfinitePay]
    IP -->|webhook| API
    API -->|shipping quote| SF[SuperFrete]
    API -->|e-mail| RS[Resend]
```

## Backend

A single Django project split into domain apps:

| App | Responsibility |
|---|---|
| Catalog | Categories, brands, products, size variations with per-size stock, favorites |
| Orders | Orders and items, addresses, shipping quotes, payments, exchange/return requests |
| Accounts | Registration, profile, password change and reset, e-mail verification |
| Core | Settings, authentication and shared infrastructure |

Request flow: `Router → ViewSet → Serializer → ORM → business signals → database`.

## Frontend

A React 19 single-page app built with Vite, styled with Tailwind CSS 4 and shadcn/ui components.

- **Routing:** React Router, with public pages (catalog, product, institutional) and a protected customer area.
- **State:** two React contexts instead of a global store — `AuthContext` (session, login/logout, profile) and `CartContext` (cart kept client-side in `localStorage`, capped by per-size stock).
- **Forms:** React Hook Form + Zod, one schema per form, with API validation errors mapped back onto the matching fields.
- **HTTP:** a single Axios instance that sends cookies, attaches the CSRF header on writes and refreshes an expired session transparently.
- **Theme:** a dark visual identity built on CSS variables, so a light theme can be turned on without touching components.

## Checkout flow

```mermaid
sequenceDiagram
    participant C as Customer
    participant FE as Frontend
    participant API as API
    participant SF as SuperFrete
    participant IP as InfinitePay

    C->>FE: Enter ZIP code
    FE->>API: Request shipping quote
    API->>SF: Quote
    SF-->>API: Options and prices
    C->>FE: Place order
    FE->>API: Create order (stock validated and reserved)
    FE->>API: Request payment link
    API->>IP: Create checkout
    IP-->>C: Payment page (Pix / card)
    IP->>API: Webhook: payment confirmed
    API->>API: Order marked as confirmed
```

## Deployment

- **Frontend:** built by Vite and served as static files from the CDN.
- **Backend:** built and deployed on a managed platform, with static files served by the application itself.
- **Configuration:** all secrets and environment-specific settings come from environment variables; nothing sensitive is versioned.
- **Database:** managed PostgreSQL behind a connection pooler, with persistent connections and health checks on the Django side.
