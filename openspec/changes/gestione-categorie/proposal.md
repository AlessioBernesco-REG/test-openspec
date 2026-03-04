## Why

Il progetto necessita di una gestione strutturata delle categorie articoli. L'app `anagrafica-articoli` referenzia le categorie tramite `categoriaId` (UUID) e dipende da questa app per il picker e la risoluzione dei nomi. Le categorie devono supportare una gerarchia ad N livelli (es. Elettronica > Componenti > Resistenze) gestita dagli admin tramite una UI dedicata con TreeTable.

## What Changes

- Nuova entita **Categoria** con struttura ad albero: nome, parentId, path materializzato, livello, ordine, stato attivo/inattivo
- Backend Node-RED con FerretDB, approccio ibrido adjacency list + materialized path per navigazione e filtro subtree
- Frontend React SPA con TreeTable per visualizzazione e gestione albero, dialog CRUD per nodi
- Autenticazione Keycloak con client `categorie-spa` e due ruoli (viewer, admin)
- Gateway APISIX con route frontend e API
- Regole di cancellazione: blocca se il nodo ha figli o articoli referenziati; soft-hide tramite flag `attivo`
- Endpoint GET consumato dall'app `anagrafica-articoli` per il picker categorie

## Capabilities

### New Capabilities
- `categorie-crud`: gestione CRUD dell'entita Categoria con struttura ad albero N livelli, visualizzazione TreeTable, controllo integrita referenziale con collection articoli, e controllo accessi basato su ruoli

### Modified Capabilities

## Impact

- **Infrastruttura**: nuovo client Keycloak `categorie-spa`, nuove route APISIX per `/categorie/app/*` e `/categorie/api/*`
- **Node-RED**: nuovi flow CRUD sulla collection `categorie` in FerretDB (istanza bishop-nodered-customer), con query cross-collection su `articoli` per check integrita
- **Frontend**: nuova SPA React accessibile su `/categorie/app/`
- **Dipendenza inversa**: l'app `anagrafica-articoli` dipende da questa app (GET /categorie/api/categorie) per il picker e la risoluzione nomi
- **Evoluzione futura**: supporto drag-and-drop per riordinamento nodi, icone per categoria, import/export albero
