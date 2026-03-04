## ADDED Requirements

### Requirement: File .env.example con tutte le variabili
Il progetto SHALL avere un file `.env.example` nella root che elenca tutte le variabili d'ambiente necessarie con commenti descrittivi e valori di esempio.

#### Scenario: Nuovo sviluppatore clona il progetto
- **WHEN** uno sviluppatore clona il repository
- **THEN** trova `.env.example` nella root con tutte le variabili documentate
- **THEN** puo copiarlo in `.env` e compilare i valori per il proprio ambiente

#### Scenario: Variabili obbligatorie documentate
- **WHEN** lo sviluppatore apre `.env.example`
- **THEN** le variabili obbligatorie (`APP_DOMAIN`, `FRONTEND_UPSTREAM`) sono marcate come tali
- **THEN** le variabili opzionali hanno valori di default documentati

### Requirement: Template JSON per configurazione APISIX e Keycloak
I file JSON di configurazione APISIX e Keycloak SHALL essere convertiti in template con placeholder `${VAR_NAME}` e i file `.json` finali SHALL essere generati tramite script.

#### Scenario: Generazione config APISIX frontend da template
- **WHEN** si esegue `generate-config.sh` con `FRONTEND_UPSTREAM=10.11.5.160:5173` e `APP_DOMAIN=cloud-ale.dev`
- **THEN** `apisix-route-categorie-frontend.json` contiene `"10.11.5.160:5173": 1` nell'upstream
- **THEN** il file NON contiene placeholder `${...}` residui

#### Scenario: Generazione config APISIX backend da template
- **WHEN** si esegue `generate-config.sh` con `APP_DOMAIN=cloud-ale.dev` e `APP_SCHEME=https`
- **THEN** `apisix-route-categorie-api.json` contiene `https://cloud-ale.dev` nei CORS e nell'issuer JWT
- **THEN** il file NON contiene placeholder `${...}` residui

#### Scenario: Generazione config Keycloak da template
- **WHEN** si esegue `generate-config.sh` con `APP_DOMAIN=mio-server.com` e `APP_SCHEME=https`
- **THEN** `keycloak-client-categorie.json` contiene `https://mio-server.com` in rootUrl, redirectUris, webOrigins
- **THEN** il file NON contiene placeholder `${...}` residui

#### Scenario: Variabile obbligatoria mancante
- **WHEN** si esegue `generate-config.sh` senza `APP_DOMAIN` definito
- **THEN** lo script esce con errore e messaggio che indica quale variabile manca

### Requirement: Script di deploy leggono variabili da environment
Gli script shell (`deploy-flows-categorie.sh`, `setup-keycloak-categorie.sh`, `upload-routes-categorie.sh`) SHALL leggere le URL base da variabili d'ambiente e `.env`, senza fallback a valori hardcoded di specifici server.

#### Scenario: deploy-flows usa NODERED_BASE_URL da env
- **WHEN** `NODERED_BASE_URL=https://mio-server.com/nodered/customer` e si esegue `deploy-flows-categorie.sh`
- **THEN** lo script usa `https://mio-server.com/nodered/customer` come URL Node-RED

#### Scenario: setup-keycloak usa KEYCLOAK_URL da env
- **WHEN** `.env` contiene `APP_DOMAIN=mio-server.com`
- **THEN** `setup-keycloak-categorie.sh` usa `https://mio-server.com/auth` come URL Keycloak

#### Scenario: upload-routes mostra URL corretti
- **WHEN** si esegue `upload-routes-categorie.sh` con `APP_DOMAIN=mio-server.com`
- **THEN** i messaggi di output mostrano `https://mio-server.com/categorie/app/` e `https://mio-server.com/categorie/api/`

### Requirement: Frontend senza fallback hardcoded
Il file `frontend-categorie/src/config/index.ts` SHALL usare solo variabili `VITE_*` senza fallback a URL specifici di un server. Il file `vite.config.ts` SHALL leggere `allowedHosts` da environment.

#### Scenario: Variabili VITE_* obbligatorie
- **WHEN** il frontend viene avviato senza `VITE_API_BASE_URL` definito
- **THEN** il valore risultante e una stringa vuota o un errore, NON un URL hardcoded a un server specifico

#### Scenario: allowedHosts configurabile
- **WHEN** `VITE_ALLOWED_HOST=mio-server.com` e definito nel `.env`
- **THEN** `vite.config.ts` usa `mio-server.com` come host consentito

### Requirement: CLAUDE.md aggiornato con placeholder
La sezione "Ambiente" di `CLAUDE.md` SHALL mostrare le variabili con placeholder generici invece di URL specifici.

#### Scenario: Sezione ambiente usa placeholder
- **WHEN** uno sviluppatore legge CLAUDE.md
- **THEN** la sezione Ambiente mostra variabili con formato `https://<APP_DOMAIN>/...` invece di `https://cloud-ale.dev/...`
