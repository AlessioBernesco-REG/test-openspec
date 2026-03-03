## 1. Configurazione Progetto

- [ ] 1.1 Creare `.env.example` con tutte le variabili ambiente (Keycloak, API, APISIX, Node-RED, App)
- [ ] 1.2 Creare `config/keycloak-client.json` (client articoli-spa, public, PKCE S256)
- [ ] 1.3 Creare `config/setup-keycloak.sh` (setup client + ruoli viewer, user, admin)
- [ ] 1.4 Creare `config/apisix-route-frontend.json` (route /articoli/app/*, NO regex_uri, no auth)
- [ ] 1.5 Creare `config/apisix-route-api.json` (route /articoli/api/*, JWT validation, CORS, proxy-rewrite)
- [ ] 1.6 Creare `config/upload-routes.sh` (upload route su APISIX Admin API)
- [ ] 1.7 Creare `config/deploy-flows.sh` (deploy flows Node-RED con backup + merge)

## 2. Backend Node-RED + FerretDB

- [ ] 2.1 Copiare template `nodered-ferretdb/nodered/flows.json` in `nodered/flows.json`
- [ ] 2.2 Personalizzare Set Config (KEYCLOAK_CLIENT_ID: articoli-spa, COLLECTION: articoli)
- [ ] 2.3 Rinominare endpoint da /items a /articoli in tutti gli http-in nodes
- [ ] 2.4 Personalizzare Build Document per campi Articolo (codice, descrizione, categoriaId, unitaMisura, prezzoUnitario, attivo, note)
- [ ] 2.5 Aggiungere validazione unicita codice su POST e PUT (categoriaId obbligatorio ma senza validazione esistenza — approccio trust)
- [ ] 2.6 Aggiungere filtri query per categoriaId e attivo su GET lista
- [ ] 2.7 Rimuovere filtro owner privacy (dati condivisi, no createdBy filter)
- [ ] 2.8 Configurare REQUIRED_ROLES nel JWT Auth Subflow (GET: viewer,user,admin; POST/PUT: user,admin; DELETE: admin)

## 3. Frontend Setup

- [ ] 3.1 Inizializzare progetto Vite + React 19 + TypeScript in frontend/
- [ ] 3.2 Installare dipendenze (@ui5/webcomponents-react, @tanstack/react-query, zustand, ky, keycloak-js)
- [ ] 3.3 Configurare vite.config.ts (base: /articoli/app/, envDir: .., allowedHosts)
- [ ] 3.4 Creare index.css (CSS reset Bishop con variabili SAP)
- [ ] 3.5 Creare index.html con title "Anagrafica Articoli"
- [ ] 3.6 Creare file dichiarazioni tipo (vite-env.d.ts, global.d.ts per UI5 Web Components)

## 4. Frontend Core

- [ ] 4.1 Creare src/config/index.ts (API URL, Keycloak config, app config da variabili ambiente)
- [ ] 4.2 Creare src/services/auth.ts (Keycloak singleton init, PKCE, token management, role check, canCreate/canEdit/canDelete)
- [ ] 4.3 Creare src/services/api.ts (ArticoliApiService con ky, CRUD methods, interceptor Bearer token, error handling 401/403)
- [ ] 4.4 Creare src/services/categorieApi.ts (CategorieApiService con ky, GET albero categorie da /categorie/api/categorie, helper risoluzione nome da ID)
- [ ] 4.5 Creare src/types/index.ts (Articolo con categoriaId UUID, ArticoloFormData, ArticoliQueryParams, Pagination, risposte, Categoria tree type, label maps per unitaMisura)
- [ ] 4.6 Creare src/stores/uiStore.ts (Zustand: dialog state, queryParams, toast, authorizationError)
- [ ] 4.7 Creare src/hooks/useArticoli.ts (React Query: articoloKeys factory, useArticoli, useArticolo, useCreateArticolo, useUpdateArticolo, useDeleteArticolo)
- [ ] 4.8 Creare src/hooks/useCategorie.ts (React Query: categorieKeys factory, useCategorie per fetch albero, helper per lookup nome da ID, staleTime lungo)

## 5. Frontend Components

- [ ] 5.1 Creare src/components/layout/AppShell.tsx (ShellBar con title, Avatar, profile popover, logout)
- [ ] 5.2 Creare src/components/pages/ArticoliPage.tsx (AnalyticalTable, toolbar con SearchField + filtro categoria Select (alimentato da API categorie) + filtro stato SegmentedButton, pulsante "Nuovo Articolo", Pagination, badge stato, colonna Categoria con nome risolto da categoriaId)
- [ ] 5.3 Creare src/components/dialogs/ArticoloDialog.tsx (dialog create/edit con form: Codice Input, Descrizione Input, Categoria TreeSelect picker (da API categorie, selezione solo foglie), U.M. Select, Prezzo Input number, Attivo Switch, Note TextArea, footer Bar)
- [ ] 5.4 Creare src/components/dialogs/DeleteArticoloDialog.tsx (dialog conferma eliminazione con MessageBox, mai window.confirm)
- [ ] 5.5 Creare src/components/common/Toast.tsx (toast notification)
- [ ] 5.6 Creare src/components/common/AuthorizationErrorDialog.tsx (dialog bloccante per errori 401/403)
- [ ] 5.7 Creare src/App.tsx (Keycloak init, QueryClient, ThemeProvider, AppShell + pagina + dialogs + toast + auth error dialog, prefetch categorie)
- [ ] 5.8 Creare src/main.tsx (entry point con BrowserRouter basename=/articoli/app/)
- [ ] 5.9 Creare src/config/index.ts: aggiungere CATEGORIE_API_BASE_URL (da env var VITE_CATEGORIE_API_BASE_URL)

## 6. Test API e Documentazione

- [ ] 6.1 Creare test/api/get-token.sh (ottiene JWT da Keycloak, salva in .token, aggiorna .http)
- [ ] 6.2 Creare test/api/articoli-api.http (tutti gli endpoint CRUD con body di esempio)
- [ ] 6.3 Creare README.md (overview progetto, prerequisiti, setup, entity model, API endpoints, matrice ruoli)
