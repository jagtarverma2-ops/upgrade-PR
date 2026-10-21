# resq reviewer notes

## Architecture

RESQ is an npm-workspaces monorepo with an Express/Mongoose/TypeScript backend and a React/Vite client. Backend HTTP layers are organized into routes, thin controllers, middleware, validators, models, and services; deterministic assessment logic lives in `server/src/analyzers/`. The client is currently a landing page foundation, while the API and assessment engine are substantially implemented.

## Conventions

- Keep business logic out of controllers: `README.md` explicitly requires controllers to parse input, call services, and return responses; scoring belongs in `server/src/analyzers/disasterAssessment.ts`.
- Treat the assessment engine as pure, deterministic, and database/network independent. Configuration belongs in `server/src/analyzers/assessmentConfig.ts`; weights, thresholds, hazard indexes, signals, resources, and actions should not be hard-coded in the algorithm.
- Preserve the shared API envelope documented in `README.md`: successful responses use `{ success: true, data }`, and errors use `{ success: false, message, error }`.
- Use strict TypeScript settings. Both `server/tsconfig.json` and `client/tsconfig.json` enable `strict`, `forceConsistentCasingInFileNames`, and unused-code checks; client imports use the `@/*` alias configured in `client/vite.config.ts`.
- Use ES modules and explicit `.js` extensions for server relative imports, as shown in `server/src/ai/index.ts` and `server/src/app.ts`.
- Deduplicate generated assessment recommendations while preserving order. `disasterAssessment.ts` uses `unique()` for risk factors, resources, and actions.
- Keep AI provider integrations behind `AiProvider` in `server/src/ai/types.ts`; register implementations through the provider map in `server/src/ai/index.ts`.
- Validate AI output with Zod before storage or display. `server/src/ai/providers/anthropicProvider.ts` parses through `aiAssessmentSchema`.
- Client styling uses Tailwind semantic tokens backed by CSS variables. `client/vite.config.ts` defines the palette and explicitly separates `severity-*` colors from `destructive`.
- React Query defaults are centralized in `client/src/main.tsx` (`staleTime: 30_000`, one retry, no focus refetch), rather than configured independently per page unless needed.

## Intentional non-standard choices

- AI is strictly optional and additive: deterministic results remain authoritative even when the provider is missing, malformed, refused, timed out, or errors. `server/src/ai/index.ts` deliberately converts failures into `{ available: false }`.
- The frontend is intentionally incomplete. `client/src/main.tsx` documents that unimplemented routes fall through to `NotFound`; do not flag missing dashboard/report/admin pages as accidental omissions.
- Tailwind is configured inline in `client/vite.config.ts` rather than via `tailwind.config.js`, intentionally keeping build configuration in one place.
- In development, an unreachable MongoDB may trigger the bundled local `mongod` fallback described in `README.md`; production treats database failure as fatal.

## Watch out for

- Never let AI output override deterministic severity, score, or recommendations; `server/src/ai/prompt.ts` explicitly makes the deterministic assessment authoritative.
- Do not send unrestricted report content or documents to AI. Preserve the field allow-list and truncation in `server/src/ai/types.ts` and `server/src/ai/prompt.ts`.
- Flag changes that bypass Zod validation, JWT/RBAC middleware, upload restrictions, rate limits, or the API error handler; these are documented security boundaries in `README.md` and wired in `server/src/app.ts`.
- Preserve incident lifecycle validation and timeline/audit behavior when changing status endpoints; illegal transitions must be rejected server-side.
- Be careful with assessment scoring changes: score components must remain bounded, thresholds ordered, breakdown totals consistent with the final score, and identical inputs deterministic.