# Technical decisions

Each entry follows the same shape: **problem → decision → why**.

## Authentication: JWT in httpOnly cookies

- **Problem:** storing tokens in `localStorage` exposes them to any XSS on the page.
- **Decision:** JWTs are delivered as `httpOnly` cookies that JavaScript cannot read. Because the browser now sends credentials automatically, every authenticated write also requires a CSRF token.
- **Why:** it removes token theft via XSS without giving up a stateless API. The frontend and API live on different subdomains, so cookie attributes had to be configured carefully for cross-site requests in production while still working on `localhost` in development.

## Access token lifetime and refresh

- **Problem:** the default short access-token lifetime expired in the middle of checkout.
- **Decision:** a longer access token plus a single-flight refresh on the frontend — concurrent requests that hit an expired token share one refresh call instead of racing.
- **Why:** smooth checkout without making refresh tokens long-lived.

## Fail-closed configuration

- **Problem:** a missing environment variable in production should never silently produce an insecure deployment.
- **Decision:** debug mode defaults to off, and the app refuses to boot in production if required security settings are missing.
- **Why:** misconfiguration becomes a failed deploy instead of an open door.

## Abuse protection

- **Decision:** sensitive operations (login, registration, checkout, password reset and others) are rate limited, and the admin panel has its own brute-force protection.
- **Why:** a small store's API is still a public target; limiting repeated attempts is cheap insurance.

## Stock consistency

- **Problem:** two customers buying the last unit of a size at the same time.
- **Decision:** stock is validated and deducted per size variation inside atomic transactions when the order is created; cancellations return items to stock.
- **Why:** stock can't go negative, and the order and its stock movement always succeed or fail together.

## Payments via webhook

- **Decision:** the order is confirmed only by the payment provider's webhook, never by the browser returning from the payment page.
- **Why:** the customer closing the tab or tampering with the redirect cannot mark an order as paid.

## Admin performance

- **Problem:** admin list pages were issuing one query per row (N+1).
- **Decision:** explicit related-object loading on every admin list and inline, and a smaller page size on the only list with inline stock editing.
- **Why:** admin pages stay fast as the catalog grows, on a modest database plan.

## Images on object storage

- **Decision:** product images live on Cloudflare R2, served from a custom domain, with admin uploads going straight to storage.
- **Why:** the app server's disk is ephemeral, and images get served from the CDN edge.
