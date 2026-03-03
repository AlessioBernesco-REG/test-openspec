# Anagrafica Articoli - Bishop App

Progetto didattico per testare l'integrazione OpenSpec/Bishop/Claude Code. CRUD anagrafica articoli con stack Bishop completo.

## Stack

- **Frontend:** React 19 + TypeScript 5.9 + Vite 7 + UI5 Web Components v2 + Zustand 5 (solo UI) + TanStack Query 5 (stato server) + Ky + keycloak-js 26
- **Backend:** Node-RED con FerretDB (MongoDB-compatible)
- **Auth:** Keycloak (OIDC/OAuth2) via APISIX gateway
- **Spec-Driven Development:** OpenSpec

## Struttura progetto

```
test-openspec/
  frontend/               React SPA
    src/
      components/
        layout/            AppShell
        pages/             ArticoliPage (AnalyticalTable)
        dialogs/           ArticoloDialog, DeleteArticoloDialog
        common/            Toast, AuthorizationErrorDialog
      hooks/               useArticoli (React Query)
      services/            api.ts (ky), auth.ts (Keycloak)
      stores/              uiStore.ts (Zustand)
      types/               Articolo, FormData, QueryParams
      config/              Env vars, costanti
  nodered/                 Flow Node-RED (FerretDB CRUD)
    flows.json
  config/                  Deploy & infrastruttura
    keycloak-client.json
    setup-keycloak.sh
    apisix-route-frontend.json
    apisix-route-api.json
    upload-routes.sh
    deploy-flows.sh
  test/api/                Test manuali API
    get-token.sh
    articoli-api.http
  openspec/                Artefatti OpenSpec
    changes/
    specs/
```

## Entita principale

**Articolo:**
- id: string (UUID)
- codice: string (obbligatorio, univoco)
- descrizione: string (obbligatorio)
- categoria: string (materia-prima | semilavorato | prodotto-finito | consumabile | altro)
- unitaMisura: string (PZ | KG | LT | MT | M2 | M3)
- prezzoUnitario: number (>= 0)
- attivo: boolean (default: true)
- note: string (opzionale)
- createdAt, updatedAt, createdBy, updatedBy: audit fields

## Path applicazione

- Frontend: `/articoli/app/`
- API Backend: `/articoli/api/`
- Keycloak client: `articoli-spa`

## Ruoli

- **viewer:** solo lettura (GET)
- **user:** lettura + creazione + modifica (GET, POST, PUT)
- **admin:** tutto + eliminazione (GET, POST, PUT, DELETE)

## Comandi principali

```bash
# Frontend
cd frontend && npm install && npm run dev

# Deploy backend
./config/deploy-flows.sh

# Setup infrastruttura
./config/setup-keycloak.sh
./config/upload-routes.sh

# Test API
cd test/api && ./get-token.sh
```

## Documentazione di riferimento (READ-ONLY)

PRIMA di generare codice, consultare questi documenti nell'ordine indicato:

| Priorita | File | Contenuto |
|----------|------|-----------|
| 1 | `/home/ubuntu/workspace/regesta.ultrafab.bishop.guidance/ai-docs/AI-MASTER-CONTEXT.md` | Stack, architettura, struttura |
| 2 | `/home/ubuntu/workspace/regesta.ultrafab.bishop.guidance/ai-docs/GENERATION-RULES.md` | Regole generazione, pattern obbligatori |
| 3 | `/home/ubuntu/workspace/regesta.ultrafab.bishop.guidance/ai-docs/CRITICAL-PATTERNS.md` | Pattern UI5/React critici |
| 4 | `/home/ubuntu/workspace/regesta.ultrafab.bishop.guidance/ai-docs/BACKEND-STORAGE.md` | Pattern persistenza Node-RED |
| 5 | `/home/ubuntu/workspace/regesta.ultrafab.bishop.guidance/ai-docs/ERROR-HANDLING.md` | Pattern error handling |

## Template e riferimenti

| Risorsa | Path |
|---------|------|
| **Backend FerretDB template** | `/home/ubuntu/workspace/regesta.ultrafab.bishop.guidance/templates/nodered-ferretdb/` |
| **Config scripts template** | `/home/ubuntu/workspace/regesta.ultrafab.bishop.guidance/templates/config-scripts/` |
| **App CRUD di riferimento** | `/home/ubuntu/workspace/regesta.ultrafab.bishop.bishop-app-contacts/` |
| **Guida creazione app** | `/home/ubuntu/workspace/regesta.ultrafab.bishop.guidance/prompts/CREATE-BISHOP-APP.md` |

## Convenzioni

- TypeScript `strict: true`, no `any`, `import type` per type-only imports
- Solo variabili CSS `--sap*` per colori (mai hardcoded)
- UI5 Web Components per tutti i controlli UI (mai `window.confirm()`)
- `ky` per HTTP client (mai axios)
- `keycloak-js` diretto (mai @react-keycloak/web)
- Zustand per stato UI, TanStack Query per stato server (mai mischiare)
- ASCII art con soli caratteri ASCII puri per diagrammi

## Ambiente

```bash
VITE_KEYCLOAK_URL=https://cloud-ale.dev/keycloak
VITE_KEYCLOAK_REALM=bishop
VITE_KEYCLOAK_CLIENT_ID=articoli-spa
VITE_API_BASE_URL=https://cloud-ale.dev/articoli/api
NODERED_URL=https://cloud-ale.dev/nodered/customer/
```
