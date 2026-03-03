## Why

Il progetto necessita di una Bishop App per la gestione dell'anagrafica articoli. Attualmente non esiste un sistema strutturato per il censimento dei prodotti (materie prime, semilavorati, prodotti finiti). L'anagrafica articoli e il primo master data necessario per qualsiasi processo di gestione magazzino, produzione o vendita. Questo progetto serve anche come caso didattico per validare il workflow OpenSpec integrato con Bishop e Claude Code.

## What Changes

- Nuova entita **Articolo** con campi: codice univoco, descrizione, **categoriaId** (riferimento a entita Categoria dell'app `gestione-categorie`), unita di misura, prezzo unitario, stato attivo/inattivo, note
- Frontend React SPA con AnalyticalTable per lista articoli, filtri per categoria e stato, ricerca testuale, dialog CRUD con **TreeSelect picker** per selezione categoria (dati da API categorie)
- Backend Node-RED con FerretDB per persistenza documenti JSON, endpoint REST CRUD completo
- Autenticazione Keycloak con client `articoli-spa` e tre ruoli (viewer, user, admin)
- Gateway APISIX con route frontend (no auth) e route API (JWT validation)
- Validazione: codice articolo univoco, prezzo >= 0, unita di misura da lista valori predefiniti
- **Dipendenza runtime** da app `gestione-categorie` per il picker categorie e la risoluzione dei nomi categoria

## Capabilities

### New Capabilities
- `articoli-crud`: gestione CRUD completa dell'entita Articolo con ricerca, filtri per categoria/stato, paginazione, validazione unicita codice e controllo accessi basato su ruoli

### Modified Capabilities

## Impact

- **Infrastruttura**: nuovo client Keycloak `articoli-spa`, nuove route APISIX per `/articoli/app/*` e `/articoli/api/*`
- **Node-RED**: nuovi flow CRUD sulla collection `articoli` in FerretDB (istanza bishop-nodered-customer)
- **Frontend**: nuova SPA React accessibile su `/articoli/app/`
- **Dipendenza**: richiede che il backend dell'app `gestione-categorie` sia disponibile per il funzionamento del picker categorie nel frontend
- **Evoluzione futura**: l'anagrafica articoli sara la base per un catalogo prodotti con immagini
