# FRYTE API Docs

https://docs.fryte.com/ · Mintlify.

`openapi.json` is auto-synced from prod on every `fryte-api` release (`deploy-prod.yml` fetches it, commits it here, Mintlify deploys). Docs show what's live in prod.

**Add/change an endpoint:** expose it in the prod schema in `fryte-api` `api/main.py` + list it in the `docs.json` navigation. Ship to prod → auto-syncs.
