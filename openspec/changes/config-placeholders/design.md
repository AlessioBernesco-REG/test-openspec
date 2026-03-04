## Context

Il progetto contiene riferimenti hardcoded a 3 host/IP diversi sparsi in 8 file:

| Valore | Tipo | File coinvolti |
|--------|------|----------------|
| `cloud-ale.dev` | Dominio produzione | config/*.sh, apisix-route-*.json, frontend config, CLAUDE.md |
| `dev-flaviocola.dev` | Dominio dev alternativo | keycloak-client-categorie.json |
| `10.11.5.160` | IP privato upstream | apisix-route-categorie-frontend.json |
| `localhost:5173` | Dev frontend | keycloak-client, apisix CORS |
| `localhost:9180` | APISIX admin | setup-keycloak, upload-routes |

Gli shell script gia usano variabili d'ambiente con fallback (`${VAR:-default}`), ma i default sono hardcoded. I file JSON non supportano variabili nativamente.

## Goals / Non-Goals

**Goals:**
- Ogni riferimento a server/IP diventa configurabile tramite variabili d'ambiente
- Un unico file `.env.example` documenta tutte le variabili necessarie
- Gli script di deploy funzionano senza modifiche manuali una volta configurato `.env`
- I file JSON usano template (`.template`) con sostituzione via `envsubst`

**Non-Goals:**
- Non si cambia la logica applicativa o le API
- Non si introduce un tool di templating esterno (Helm, Jinja, ecc.)
- Non si gestiscono ambienti multipli (staging, prod) - un solo set di variabili per ambiente
- Non si toccano i flow Node-RED (nodered-categorie/flows.json)

## Decisions

### 1. Meccanismo per file JSON: template + envsubst

I file JSON di configurazione (APISIX, Keycloak) diventano template `.json.template` con placeholder `${VAR_NAME}`. Uno script wrapper genera i `.json` finali tramite `envsubst`.

**Alternativa considerata:** sed inline. Scartata perche fragile con URL contenenti `/` e meno leggibile.

**Alternativa considerata:** jq per sostituzione. Scartata perche il Lua embedded nell'APISIX route non e JSON-friendly.

### 2. Variabili d'ambiente standardizzate

Tutte le variabili usano il prefisso che riflette il loro scopo:

| Variabile | Default | Uso |
|-----------|---------|-----|
| `APP_DOMAIN` | (obbligatorio) | Dominio principale (es. `cloud-ale.dev`) |
| `APP_SCHEME` | `https` | Schema URL (http/https) |
| `KEYCLOAK_DOMAIN` | `${APP_DOMAIN}` | Dominio Keycloak (se diverso) |
| `KEYCLOAK_BASE_PATH` | `/auth` | Path base Keycloak |
| `KEYCLOAK_REALM` | `bishop` | Realm Keycloak |
| `NODERED_BASE_URL` | `${APP_SCHEME}://${APP_DOMAIN}/nodered/customer` | URL Node-RED |
| `FRONTEND_UPSTREAM` | (obbligatorio) | IP:porta upstream frontend (es. `10.11.5.160:5173`) |
| `DEV_PORT` | `5173` | Porta dev locale |
| `APISIX_ADMIN_URL` | `http://localhost:9180` | URL admin APISIX |

### 3. File .env.example nella root del progetto

Un file `.env.example` con tutte le variabili documentate e valori di esempio. Il file `.env` reale e gia in `.gitignore`.

### 4. Script generate-config.sh

Nuovo script che legge `.env` (o variabili d'ambiente) e genera i file JSON finali dai template usando `envsubst`. Va eseguito prima di `upload-routes` e `setup-keycloak`.

### 5. Frontend: rimuovere fallback hardcoded

In `frontend-categorie/src/config/index.ts` i fallback URL hardcoded vengono rimossi - le variabili `VITE_*` diventano obbligatorie (errore a runtime se mancanti). In `vite.config.ts` il `allowedHosts` legge da env.

## Risks / Trade-offs

- **[Passo in piu nel deploy]** Bisogna eseguire `generate-config.sh` prima del deploy. Mitigazione: lo script setup-keycloak e upload-routes lo invocano automaticamente se i `.json` non esistono.
- **[envsubst non disponibile ovunque]** Raro ma possibile. Mitigazione: `envsubst` fa parte di `gettext` disponibile su tutti i sistemi Linux/macOS principali; in alternativa si puo usare `sed` come fallback nello script.
- **[File .env dimenticato]** Se manca `.env` e non ci sono env vars, lo script fallisce. Mitigazione: `generate-config.sh` valida le variabili obbligatorie e mostra errore chiaro.
