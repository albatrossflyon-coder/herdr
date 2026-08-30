# Herdr Command Center

## What is included

This archive contains the React, Tailwind, Express, tRPC, Drizzle, and Manus OAuth source for the Herdr Command Center. It includes the durable monitoring schema, seeded demo data, snapshot-driven dashboard, health calculations, guarded approval workflow, and automated tests.

The project is a **full-stack Node application**, not a single Markdown document. Markdown is included as documentation; the application source remains in its original folders so it can be opened and modified by Claude Code or another coding assistant.

## Run locally

Use Node.js 22 or a compatible current Node.js release and pnpm.

```bash
unzip herdr-command-center-source.zip
cd herdr-command-center
pnpm install
pnpm check
pnpm test
pnpm dev
```

Open the local URL printed by the development server. The dashboard will use its durable database snapshot when the configured database is available and will seed the demo monitoring workspace on first access.

## Environment variables

Do not commit secrets to Git or place them in the ZIP. Configure the required values in the environment used by the target host. The scaffold expects a database connection and Manus OAuth/runtime values, including `DATABASE_URL`, `JWT_SECRET`, `VITE_APP_ID`, `OAUTH_SERVER_URL`, `VITE_OAUTH_PORTAL_URL`, `OWNER_OPEN_ID`, `OWNER_NAME`, and the built-in Forge API variables. The exact values are account- and deployment-specific.

The archive intentionally excludes `.env` files, local secrets, dependency folders, build output, and local server logs.

## Use with Claude Code

After extracting the archive, open the project directory in Claude Code:

```bash
cd herdr-command-center
claude
```

Ask Claude Code to read `README.md`, `DEPLOYMENT_GUIDE.md`, `todo.md`, and the relevant files under `client/`, `server/`, `shared/`, and `drizzle/` before changing anything. Run `pnpm check && pnpm test` after each meaningful change. Preserve the existing approval boundary: there is no unrestricted direct pane-messaging action in the dashboard.

## Vercel considerations

Vercel can host JavaScript/TypeScript web workloads, but this project was scaffolded as a single full-stack Node application with Express, tRPC, Drizzle, OAuth, and a durable SQL database. Moving it to Vercel may require converting server entry points to Vercel-compatible functions, configuring the database for a hosted environment, and adapting long-lived or push-event behavior. Treat the supplied project as the source to adapt rather than assuming that `vercel deploy` will reproduce the current server runtime unchanged.

For the first deployment, keep the existing Manus-hosted version as the known-good reference. If you choose Vercel, create a separate branch, configure environment variables in the Vercel project settings, connect the production database with SSL where required, and verify OAuth callback URLs, tRPC requests, snapshot loading, and approval mutations before switching traffic.

The official [Vercel documentation](https://vercel.com/docs) covers deployment and framework/runtime configuration. The [Claude Code documentation](https://code.claude.com/docs/en/overview) covers project-based coding workflows.

## Recommended first Vercel checklist

| Area | Verify |
|---|---|
| Build | `pnpm check`, `pnpm test`, and the production build succeed in CI. |
| Database | `DATABASE_URL` points to the intended hosted database and migrations have been applied safely. |
| Authentication | OAuth callback and portal URLs use the Vercel domain. |
| Backend | `/api/trpc` routes work in production and do not depend on a local long-running process. |
| Operations | Herdr push-event ingestion and any watchdog process have a deployment model compatible with the selected host. |
| Safety | Approval actions remain authenticated, confirmation-based, and target-validated. |

## Current limitations

The included dashboard is adapter-ready and populated with transparent demo data. It does not yet connect to your live Herdr panes, consume your production `pane.agent_status_changed` stream, or execute real pane operations. Those integrations should be added only after their authentication, event schema, retry behavior, and approval policy are specified.

## References

[1]: https://vercel.com/docs "Vercel Documentation"
[2]: https://code.claude.com/docs/en/overview "Claude Code Documentation"
