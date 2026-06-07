# The Vibe Shopping

Vibe is the AI styling engine and repository behind **The Vibe Shop**. Vibe is
the shipped Shopify storefront experience: a conversational stylist, PDP
virtual try-on gateway, saved looks surface, profile page, wardrobe manager, and
creator story-share discount loop.

The current production surface is Shopify-first. 

Try it yourself: https://thesigmavibe.shop/apps/vibe/style

## Demo
1. Getting started: https://www.instagram.com/s/aGlnaGxpZ2h0OjE3ODY4NTc3NTMwNTQ4NDA0?story_media_id=3913616707499443110&igsh=dXVsNDRtMHJqNG5t
2. How it works: https://www.instagram.com/s/aGlnaGxpZ2h0OjE4MDY5MDIwMDA5NjkxODM0?story_media_id=3913622270497123874&igsh=MXA1ZmZ5dDhmNzN5OA%3D%3D
3. Share to earn discount: https://www.instagram.com/s/aGlnaGxpZ2h0OjE4MDgwMTIzOTAzMjE3ODgx?story_media_id=3913790138220756654&igsh=MTExMm1lam9ib3NneQ%3D%3D


## Current Shipped System

| Surface | Current behavior |
|---|---|
| Storefront | `thesigmavibe.shop`, with PDP theme blocks that open Vibe at `/apps/vibe/style?productId=<shopify product id>` |
| Vibe app | Remix/React app under `vibe-app/`, deployed on Vercel and rendered through Shopify App Proxy as `application/liquid` |
| Engine | FastAPI service on Fly.io Mumbai, backed by Supabase/Postgres, pgvector, and persistent media storage |
| Identity | Anonymous localStorage session by default, merged into `shopify:<customer_id>` when Customer Account identity is available |
| Catalog | Per-tenant Shopify catalog enrichment, embeddings, retrieval, PDP card hydration, and Shopify variant/cart integration |
| Try-on | On-demand only, using `gemini-3.1-flash-image-preview`; recommendation turns do not pre-render try-on images |

Active customer routes:

| Route | Purpose |
|---|---|
| `/apps/vibe/style` | Conversational styling, PDP seeded entry, onboarding, recommendation cards, on-demand try-on |
| `/apps/vibe/looks` | Saved recommendation history and persistent virtual try-on gallery |
| `/apps/vibe/profile` | Editable gender, photos, fit details, visual analysis status, and re-analysis |
| `/apps/vibe/wardrobe` | Wardrobe upload and item management |
| `/apps/vibe/s/<token>` | Story-share short-link redirect |

## Core Customer Flows

- **Onboarding:** Vibe asks for gender first. Photos are optional for generic chat, but a full-body photo is required for try-on and body-frame visual analysis. Fit details are asked only after image upload and use height, waist, and age, not date of birth.
- **Outfit recommendations:** Active chat intents let you find just the right looks for you covering intent across occasion
  recommendations, pairing requests to a piece you already own, shopping decisions, style discovery,
  explanations, feedback, and wardrobe ingestion. Shoes and accessories are not
  supported as pairing anchors today.
- **PDP virtual try-on:** A Shopify PDP opens Vibe with `productId`. If the
  customer asks about "this" item, the seed product is hydrated as the current
  anchor. Fresh unrelated asks do not inherit that stale PDP context.
- **Story-share discount:** Saved looks can open the story-share modal, issue a
  short link, guide Instagram sharing, and claim a Shopify discount code through
  the honor-system claim flow.
- **Knowledge graph signals:** Recommendation cards record canonical interaction
  signals in `catalog_interaction_history`: `save`, `add_to_cart`, `buy`, and
  `dismiss`. `save` and `dismiss` also mirror to legacy `feedback_events`
  `like`/`dislike` rows for compatibility.

## Runtime Architecture

| Layer | Main files and services |
|---|---|
| Shopify app | `vibe-app/app/routes/apps.vibe.*`, `vibe-app/app/components/*`, `vibe-app/app/lib/*` |
| Engine API | `modules/agentic_application/src/agentic_application/api.py` and related services |
| Recommendation runtime | Planner, architect, retrieval, composer, rater, formatter, and response-safety modules under `modules/agentic_application/` |
| Platform core | Config, repositories, Supabase REST clients, schemas, and shared platform services under `modules/platform_core/` |
| Catalog | Shopify catalog ingestion, enrichment, embeddings, and retrieval under `modules/catalog/` |
| Profile and wardrobe | Onboarding/profile/wardrobe utilities under `modules/user/` and `modules/user_profiler/` |
| Data | Supabase migrations under `supabase/migrations/`; persistent media mounted on the Fly app volume |
| Prompts and knowledge | `prompt/`, `knowledge/`, and `archetypes/` |

## Deployment

| Component | Command / owner |
|---|---|
| Engine | `fly deploy` from the repo root |
| Vibe app | Vercel auto-deploys from `main`; manual production deploy uses the configured Vercel project |
| Shopify app config | `cd vibe-app && npm run deploy` when app extensions or Shopify app config change |
| Database schema | Supabase migrations in `supabase/migrations/`; apply with `supabase db push --yes` when schema changes |
| Catalog and ops scripts | `ops/scripts/` and `scripts/`, run with the appropriate `APP_ENV` |

The current production engine is `vibe-engine` on Fly.io. The Vibe app runs on
Vercel in the Mumbai region and is installed on TheSigmaVibe's Shopify store.

## Observability

The system is intentionally traceable across Vercel, Fly, and Supabase.

- Vibe app structured logs are emitted through
  `vibe-app/app/lib/logger.server.ts`.
- Try-on requests generate an app `requestId` and forward it to the engine via
  `X-Request-Id` and `request_id`.
- Engine metrics are exposed from `/metrics`.
- Recommendation-card signals emit `vibe_recommendation_signal_*` logs in the
  app and `recommendation_interaction_outcome` logs in the engine.

## Documentation Map

| Document | Purpose |
|---|---|
| [`docs/CURRENT_SYSTEM.md`](docs/CURRENT_SYSTEM.md) | Source of truth for shipped Shopify Vibe behavior |
| [`docs/OPERATIONS.md`](docs/OPERATIONS.md) | Dashboards, log events, metrics, and first-50 rollout queries |
| [`docs/APPLICATION_SPECS.md`](docs/APPLICATION_SPECS.md) | Detailed implementation spec with a current-system banner |
| [`docs/WORKFLOW_REFERENCE.md`](docs/WORKFLOW_REFERENCE.md) | Human-facing execution flows for intents, recommendation cards, and persistence |
| [`docs/DESIGN.md`](docs/DESIGN.md) | Product design system and UX rules |
| [`docs/PRODUCT.md`](docs/PRODUCT.md) | Product definition, personas, and user stories |
| [`docs/RELEASE_READINESS.md`](docs/RELEASE_READINESS.md) | Release checklist |
| [`docs/REORG_PROPOSAL.md`](docs/REORG_PROPOSAL.md) | Proposed future repo reorganization, not current layout |

## Repository Layout

```text
.
|-- vibe-app/                  # Shopify Remix app, App Proxy routes, theme extension, UI tests
|-- modules/
|   |-- agentic_application/   # FastAPI engine, planner/orchestrator, recommendation runtime
|   |-- catalog/               # Shopify catalog ingestion, enrichment, retrieval, embeddings
|   |-- platform_core/         # Config, repositories, schemas, Supabase clients
|   |-- style_engine/          # Style-engine configs
|   |-- user/                  # Onboarding, wardrobe, profile utilities
|   `-- user_profiler/         # User profile analysis utilities
|-- prompt/                    # Runtime prompts
|-- knowledge/                 # Knowledge assets
|-- archetypes/                # Style archetype data
|-- supabase/migrations/       # Database migrations
|-- ops/scripts/               # Operational scripts
|-- scripts/                   # Developer and validation scripts
|-- tests/                     # Engine and integration tests
`-- docs/                      # Current system docs and historical design/spec docs
```
