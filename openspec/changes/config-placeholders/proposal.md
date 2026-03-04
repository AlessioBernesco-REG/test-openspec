## Why

Il codice generato contiene riferimenti hardcoded a server specifici (`cloud-ale.dev`, `dev-flaviocola.dev`) e indirizzi IP privati (`10.11.5.160`). Questo rende il progetto non portabile e richiede modifiche manuali per ogni ambiente di deploy.

## What Changes

- Introduzione di placeholder configurabili per hostname, IP e URL base in tutti i file di configurazione e script
- Sostituzione di tutti i riferimenti hardcoded a `cloud-ale.dev`, `dev-flaviocola.dev`, `10.11.5.160` con variabili/placeholder
- Creazione di un file `.env.example` centralizzato con tutti i placeholder documentati
- Aggiornamento degli script di deploy per leggere le variabili da environment o file `.env`
- Aggiornamento dei config JSON (APISIX, Keycloak) per usare template con sostituzione variabili

## Capabilities

### New Capabilities
- `env-placeholders`: Parametrizzazione di tutti i riferimenti server/IP con placeholder configurabili tramite variabili d'ambiente e file `.env.example`

### Modified Capabilities

Nessuna capability esistente modificata a livello di requisiti.

## Impact

- **Config scripts** (`config/*.sh`): Tutti gli script useranno variabili d'ambiente invece di URL hardcoded
- **Config JSON** (`config/*.json`): I file APISIX e Keycloak useranno un meccanismo di template/sostituzione
- **Frontend** (`frontend-categorie/src/config/index.ts`, `vite.config.ts`): Fallback URL parametrizzati
- **CLAUDE.md**: Sezione ambiente aggiornata con placeholder
- **Nessun impatto su API o logica applicativa**: cambia solo la configurazione
