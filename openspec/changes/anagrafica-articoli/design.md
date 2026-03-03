## Context

Progetto greenfield: una nuova Bishop App standalone per la gestione dell'anagrafica articoli. Non ci sono vincoli su codice esistente. Lo stack e completamente prescritto dalle convenzioni Bishop (React 19 + UI5 WC + TypeScript + Vite + Zustand + React Query + Node-RED + FerretDB + Keycloak + APISIX).

L'app di riferimento principale e bishop-app-contacts, da cui si ereditano tutti i pattern frontend e backend. La differenza principale e nel layout: contacts usa Master-Detail, questa app usera AnalyticalTable con dialog.

Vincoli:
- Tutte le convenzioni Bishop come definite in `guidance/ai-docs/GENERATION-RULES.md`
- Backend DEVE partire dal template normativo `nodered-ferretdb`
- Solo variabili CSS `--sap*`, mai colori hardcoded
- `ky` come HTTP client, `keycloak-js` diretto, TypeScript strict

## Goals / Non-Goals

**Goals:**
- App CRUD completa funzionante end-to-end con stack Bishop
- Tutti gli artefatti di deployment (Keycloak, APISIX, Node-RED, frontend)
- Pattern Bishop replicabili per future app
- Documentazione per scopi didattici

**Non-Goals:**
- Categorie gerarchiche (sara evoluzione futura con OpenSpec)
- Upload immagini prodotto (sara evoluzione futura)
- Export/import CSV o integrazione con sistemi esterni
- Test automatizzati (unit test, e2e) — fuori scope per questa iterazione
- Internazionalizzazione (i18n) — l'app sara in italiano

## Decisions

### D1: FerretDB come storage

**Decisione:** FerretDB (MongoDB-compatible su PostgreSQL) per la persistenza.

**Motivazione:** camelCase nativo nei documenti JSON elimina la trasformazione snake_case -> camelCase richiesta da PostgreSQL diretto. Schema flessibile per iterare rapidamente sul modello entita. Allineato al template normativo `nodered-ferretdb`.

**Alternative considerate:**
- PostgreSQL diretto → richiede trasformazione camelCase in ogni response node, migration SQL per ogni cambio schema
- Flow Context Node-RED (in-memory) → nessuna persistenza al restart, inadatto anche per demo

### D2: AnalyticalTable con Dialog (non Master-Detail)

**Decisione:** layout a pagina singola con AnalyticalTable UI5 + dialog modali per create/edit.

**Motivazione:** l'articolo e un'entita piatta senza sotto-risorse (no indirizzi, no riferimenti come in contacts). Un layout Master-Detail sarebbe over-engineering. L'AnalyticalTable offre sorting, grouping e column resize nativi.

**Alternative considerate:**
- Master-Detail come contacts → pannello dettaglio inutilizzato, entita troppo semplice
- Table standard UI5 → meno funzionalita native (no sorting, no grouping)

### D3: Tre ruoli Keycloak (viewer, user, admin)

**Decisione:** client `articoli-spa` con ruoli viewer (sola lettura), user (CRUD senza delete), admin (CRUD completo).

**Motivazione:** pattern standard Bishop. La separazione user/admin protegge da cancellazioni accidentali. Viewer permette consultazione senza rischi.

**Matrice permessi:**
```
             GET    POST   PUT    DELETE
viewer        x
user          x      x      x
admin         x      x      x      x
```

### D4: Dati condivisi (no filtro owner)

**Decisione:** tutti gli utenti vedono tutti gli articoli. Nessun filtro `createdBy`.

**Motivazione:** l'anagrafica articoli e master data condiviso. Non ha senso che un utente veda solo gli articoli che ha creato lui. Il campo `createdBy` viene comunque salvato per audit, ma non usato come filtro.

**Alternative considerate:**
- Filtro owner come contacts → inadatto per master data, ogni utente vedrebbe solo i "suoi" articoli

### D5: Path applicazione /articoli/

**Decisione:** frontend su `/articoli/app/`, API su `/articoli/api/`.

**Motivazione:** naming italiano coerente con il dominio. Pattern Bishop standard con separazione app/api.

## Risks / Trade-offs

- **[FerretDB maturity]** FerretDB e meno maturo di MongoDB nativo → Mitigazione: usiamo solo operazioni CRUD semplici (find, insertOne, updateOne, deleteOne), nessuna aggregation pipeline
- **[Nessun test automatizzato]** L'assenza di unit/e2e test rende il refactoring piu rischioso → Mitigazione: accettabile per un progetto didattico, i test manuali via file `.http` coprono il backend
- **[Hard delete vs soft delete]** Si e scelto hard delete (DELETE fisico) invece di soft delete → Mitigazione: per un'anagrafica semplice il soft delete aggiunge complessita non necessaria. Puo essere aggiunto in un'evoluzione futura
