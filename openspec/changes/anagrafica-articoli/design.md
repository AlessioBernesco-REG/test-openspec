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
- Gestione CRUD delle categorie (app separata `gestione-categorie`, sviluppata in parallelo)
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

### D6: categoriaId come riferimento (non enum)

**Decisione:** il campo `categoriaId` e un UUID che referenzia l'entita Categoria gestita dall'app `gestione-categorie`. Non viene usato un enum hardcoded.

**Motivazione:** le due app si sviluppano assieme. Partire subito con il riferimento evita una migrazione futura (enum → UUID). Il frontend mostra un TreeSelect picker alimentato da `GET /categorie/api/categorie`.

**Alternative considerate:**
- Enum fisso con migrazione successiva → debito tecnico certo, doppio lavoro su frontend (Select → TreeSelect) e backend (stringa → UUID)
- Enum + categoriaId ibrido → complessita senza beneficio

### D7: Trust su categoriaId (nessuna validazione backend)

**Decisione:** il backend articoli NON valida che il categoriaId esista nella collection categorie. Si affida al frontend che presenta solo ID validi tramite il picker.

**Motivazione:** la regola "blocca cancellazione categoria se ha articoli sotto" (implementata nell'app categorie) previene la creazione di orfani. Gli unici orfani possibili sarebbero da chiamate API manuali con ID inventati — rischio accettabile. Validare richiederebbe una chiamata cross-app dal backend articoli al backend categorie, aggiungendo accoppiamento e latenza.

**Alternative considerate:**
- Validazione sincrona cross-app → accoppiamento backend-to-backend, single point of failure, latenza aggiuntiva su ogni POST/PUT

### D8: Picker categorie — albero completo, solo foglie selezionabili

**Decisione:** il TreeSelect picker mostra l'intero albero delle categorie per dare contesto gerarchico, ma solo le foglie (categorie senza figli) sono selezionabili. Le categorie intermedie sono visibili ma disabilitate (grigie, non cliccabili).

**Motivazione:** coerente con il vincolo "articoli solo su foglie" stabilito nella change `gestione-categorie` (D8). Mostrare l'albero completo aiuta l'utente a navigare la gerarchia e capire dove si colloca la foglia scelta. La distinzione foglia/non-foglia si calcola lato frontend dal tree (una categoria e foglia se nessun'altra categoria ha il suo id come parentId).

**Alternative considerate:**
- Mostrare solo foglie → perde il contesto gerarchico, l'utente vede una lista piatta di nomi senza capire la struttura
- Mostrare tutto e permettere selezione ovunque → viola il vincolo "solo foglie"

## Risks / Trade-offs

- **[FerretDB maturity]** FerretDB e meno maturo di MongoDB nativo → Mitigazione: usiamo solo operazioni CRUD semplici (find, insertOne, updateOne, deleteOne), nessuna aggregation pipeline
- **[Nessun test automatizzato]** L'assenza di unit/e2e test rende il refactoring piu rischioso → Mitigazione: accettabile per un progetto didattico, i test manuali via file `.http` coprono il backend
- **[Hard delete vs soft delete]** Si e scelto hard delete (DELETE fisico) invece di soft delete → Mitigazione: per un'anagrafica semplice il soft delete aggiunge complessita non necessaria. Puo essere aggiunto in un'evoluzione futura
- **[Accoppiamento runtime con app Categorie]** Il frontend articoli dipende dall'API categorie per il picker → Mitigazione: se l'API categorie non e raggiungibile, il picker non funziona ma il resto dell'app rimane operativo. Per il contesto didattico il rischio e accettabile
- **[Trust su categoriaId]** Nessuna validazione backend dell'esistenza della categoria → Mitigazione: la regola "blocca cancellazione categoria con articoli sotto" previene orfani. Solo chiamate API manuali con ID inventati potrebbero creare inconsistenze
