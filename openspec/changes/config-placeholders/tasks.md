## 1. File .env.example e infrastruttura

- [ ] 1.1 Creare `.env.example` nella root con tutte le variabili documentate (APP_DOMAIN, APP_SCHEME, KEYCLOAK_DOMAIN, KEYCLOAK_BASE_PATH, KEYCLOAK_REALM, NODERED_BASE_URL, FRONTEND_UPSTREAM, DEV_PORT, APISIX_ADMIN_URL)
- [ ] 1.2 Aggiungere `.env` al `.gitignore` (se non gia presente)
- [ ] 1.3 Aggiungere i file `*.json` generati della cartella config/ al `.gitignore` (verranno generati dai template)

## 2. Template JSON

- [ ] 2.1 Convertire `config/apisix-route-categorie-frontend.json` in `config/apisix-route-categorie-frontend.json.template` con placeholder `${FRONTEND_UPSTREAM}`
- [ ] 2.2 Convertire `config/apisix-route-categorie-api.json` in `config/apisix-route-categorie-api.json.template` con placeholder `${APP_SCHEME}`, `${APP_DOMAIN}`, `${KEYCLOAK_BASE_PATH}`, `${KEYCLOAK_REALM}`
- [ ] 2.3 Convertire `config/keycloak-client-categorie.json` in `config/keycloak-client-categorie.json.template` con placeholder `${APP_SCHEME}`, `${APP_DOMAIN}`, `${DEV_PORT}`

## 3. Script generate-config.sh

- [ ] 3.1 Creare `config/generate-config.sh` che carica `.env`, valida variabili obbligatorie, e genera i `.json` finali da ogni `.json.template` usando `envsubst`
- [ ] 3.2 Aggiungere validazione: errore chiaro se `APP_DOMAIN` o `FRONTEND_UPSTREAM` mancano
- [ ] 3.3 Aggiungere controllo: errore se restano placeholder `${...}` nei file generati

## 4. Aggiornamento script di deploy

- [ ] 4.1 Aggiornare `config/deploy-flows-categorie.sh`: rimuovere URL hardcoded, leggere `NODERED_BASE_URL` da `.env` o env, errore se non definito
- [ ] 4.2 Aggiornare `config/setup-keycloak-categorie.sh`: rimuovere fallback hardcoded a `cloud-ale.dev`, derivare URL da `APP_DOMAIN`/`APP_SCHEME`, rimuovere URL hardcoded nel messaggio finale
- [ ] 4.3 Aggiornare `config/upload-routes-categorie.sh`: URL nei messaggi finali derivati da variabili env

## 5. Frontend

- [ ] 5.1 Aggiornare `frontend-categorie/src/config/index.ts`: rimuovere fallback URL hardcoded, lasciare solo `import.meta.env.VITE_*`
- [ ] 5.2 Aggiornare `frontend-categorie/vite.config.ts`: leggere `allowedHosts` da `VITE_ALLOWED_HOST` env var

## 6. Documentazione

- [ ] 6.1 Aggiornare sezione "Ambiente" in `CLAUDE.md` con placeholder generici (`<APP_DOMAIN>`) invece di URL specifici
