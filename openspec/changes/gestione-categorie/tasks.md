## 1. Configurazione Progetto

- [ ] 1.1 Creare `config/keycloak-client-categorie.json` (client categorie-spa, public, PKCE S256)
- [ ] 1.2 Creare `config/setup-keycloak-categorie.sh` (setup client + ruoli viewer, admin)
- [ ] 1.3 Creare `config/apisix-route-categorie-frontend.json` (route /categorie/app/*, NO regex_uri, no auth)
- [ ] 1.4 Creare `config/apisix-route-categorie-api.json` (route /categorie/api/*, JWT validation, CORS, proxy-rewrite)
- [ ] 1.5 Creare `config/upload-routes-categorie.sh` (upload route su APISIX Admin API)
- [ ] 1.6 Creare `config/deploy-flows-categorie.sh` (deploy flows Node-RED con backup + merge)

## 2. Backend Node-RED + FerretDB

- [ ] 2.1 Copiare template `nodered-ferretdb/nodered/flows.json` in `nodered-categorie/flows.json`
- [ ] 2.2 Personalizzare Set Config (KEYCLOAK_CLIENT_ID: categorie-spa, COLLECTION: categorie)
- [ ] 2.3 Rinominare endpoint da /items a /categorie in tutti gli http-in nodes
- [ ] 2.4 Personalizzare Build Document per campi Categoria (nome, parentId, path, livello, ordine, attivo)
- [ ] 2.5 Implementare calcolo automatico path e livello: su POST, risolvere padre e costruire path = parentPath + "/" + slug(nome), livello = parent.livello + 1
- [ ] 2.6 Implementare validazione unicita nome sotto stesso padre su POST e PUT
- [ ] 2.7 Implementare validazione parentId esistente su POST e PUT (se parentId != null)
- [ ] 2.8 Implementare riscrittura path e livello su PUT con cambio nome o parentId: trovare discendenti con prefix match su vecchio path, aggiornare tutti
- [ ] 2.9 Implementare check anti-circolarita su PUT con cambio parentId: verificare che il nuovo padre non sia un discendente del nodo
- [ ] 2.10 Implementare check DELETE: blocca se ha figli diretti (query parentId = id)
- [ ] 2.11 Implementare check DELETE: blocca se ha articoli referenziati (query cross-collection su `articoli` con categoriaId = id)
- [ ] 2.12 Aggiungere filtro query per attivo su GET lista
- [ ] 2.13 Ordinare risultati GET per path e ordine
- [ ] 2.14 Rimuovere filtro owner privacy (dati condivisi, no createdBy filter)
- [ ] 2.15 Configurare REQUIRED_ROLES nel JWT Auth Subflow (GET: viewer,admin; POST/PUT/DELETE: admin)

## 3. Frontend Setup

- [ ] 3.1 Inizializzare progetto Vite + React 19 + TypeScript in frontend-categorie/
- [ ] 3.2 Installare dipendenze (@ui5/webcomponents-react, @tanstack/react-query, zustand, ky, keycloak-js)
- [ ] 3.3 Configurare vite.config.ts (base: /categorie/app/, envDir: .., allowedHosts)
- [ ] 3.4 Creare index.css (CSS reset Bishop con variabili SAP)
- [ ] 3.5 Creare index.html con title "Gestione Categorie"
- [ ] 3.6 Creare file dichiarazioni tipo (vite-env.d.ts, global.d.ts per UI5 Web Components)

## 4. Frontend Core

- [ ] 4.1 Creare src/config/index.ts (API URL, Keycloak config, app config da variabili ambiente)
- [ ] 4.2 Creare src/services/auth.ts (Keycloak singleton init, PKCE, token management, role check, canCreate/canEdit/canDelete basati su viewer/admin)
- [ ] 4.3 Creare src/services/api.ts (CategorieApiService con ky, CRUD methods, interceptor Bearer token, error handling 401/403/409)
- [ ] 4.4 Creare src/types/index.ts (Categoria, CategoriaFormData, CategoriaTreeNode, risposte API)
- [ ] 4.5 Creare src/stores/uiStore.ts (Zustand: dialog state, toast, authorizationError)
- [ ] 4.6 Creare src/hooks/useCategorie.ts (React Query: categorieKeys factory, useCategorie, useCreateCategoria, useUpdateCategoria, useDeleteCategoria)
- [ ] 4.7 Creare src/utils/treeUtils.ts (buildTree: flat list → nested tree, findNode, getAncestors, getDescendants — logica pura, no side effects)

## 5. Frontend Components

- [ ] 5.1 Creare src/components/layout/AppShell.tsx (ShellBar con title "Gestione Categorie", Avatar, profile popover, logout)
- [ ] 5.2 Creare src/components/pages/CategoriePage.tsx (TreeTable con colonne: Nome, Livello, Stato badge, Articoli conteggio, Azioni; toolbar con pulsante "Nuova Categoria" e filtro stato)
- [ ] 5.3 Creare src/components/dialogs/CategoriaDialog.tsx (dialog create/edit con form: Nome Input, Padre Select con categorie esistenti + opzione "Nessuno - radice", Ordine Input number, Attivo Switch, footer Bar)
- [ ] 5.4 Creare src/components/dialogs/DeleteCategoriaDialog.tsx (dialog conferma eliminazione con MessageBox, gestione errore 409 con messaggio dal backend)
- [ ] 5.5 Creare src/components/common/Toast.tsx (toast notification)
- [ ] 5.6 Creare src/components/common/AuthorizationErrorDialog.tsx (dialog bloccante per errori 401/403)
- [ ] 5.7 Creare src/App.tsx (Keycloak init, QueryClient, ThemeProvider, AppShell + pagina + dialogs + toast + auth error dialog)
- [ ] 5.8 Creare src/main.tsx (entry point con BrowserRouter basename=/categorie/app/)

## 6. Test API e Documentazione

- [ ] 6.1 Creare test/api/get-token-categorie.sh (ottiene JWT da Keycloak con ruolo admin, salva in .token)
- [ ] 6.2 Creare test/api/categorie-api.http (tutti gli endpoint CRUD con body di esempio, inclusi scenari di errore: cancellazione con figli, con articoli, riferimento circolare)
- [ ] 6.3 Aggiornare README.md con sezione app categorie (overview, entity model, API endpoints, matrice ruoli, relazione con app articoli)
