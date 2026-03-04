## Context

Nuova Bishop App per la gestione delle categorie articoli in struttura gerarchica ad N livelli. Sviluppata in parallelo con `anagrafica-articoli` che la consuma come dipendenza per il picker categorie. Lo stack e completamente prescritto dalle convenzioni Bishop.

FerretDB non supporta `$graphLookup` ne aggregation pipeline complesse, quindi la struttura ad albero richiede un approccio ibrido che permetta sia navigazione padre/figlio che query su subtree senza ricorsione.

L'app di riferimento principale e bishop-app-contacts per i pattern generali, e bishop-app-hierarchy (se disponibile) per i pattern di TreeTable.

Vincoli:
- Tutte le convenzioni Bishop come definite in `guidance/ai-docs/GENERATION-RULES.md`
- Backend DEVE partire dal template normativo `nodered-ferretdb`
- Solo variabili CSS `--sap*`, mai colori hardcoded
- `ky` come HTTP client, `keycloak-js` diretto, TypeScript strict

## Goals / Non-Goals

**Goals:**
- App CRUD per categorie con visualizzazione ad albero (TreeTable)
- Struttura ad N livelli di profondita arbitraria
- Integrita referenziale: blocco cancellazione se la categoria ha articoli o figli
- API consumabile dall'app articoli per picker e risoluzione nomi
- Pattern Bishop replicabili

**Non-Goals:**
- Drag-and-drop per riordinamento nodi (evoluzione futura)
- Icone o immagini per categoria (evoluzione futura)
- Import/export albero da file (evoluzione futura)
- Test automatizzati (unit test, e2e) — fuori scope
- Internazionalizzazione (i18n) — l'app sara in italiano

## Decisions

### D1: Approccio ibrido adjacency list + materialized path

**Decisione:** ogni documento Categoria ha sia `parentId` (adjacency list) per navigazione padre/figlio, sia `path` materializzato (es. `/elettronica/componenti/resistenze`) per query su subtree con prefix match.

**Motivazione:** FerretDB non supporta `$graphLookup` per attraversamento ricorsivo dell'albero. L'adjacency list da sola richiederebbe query ricorsive lente. Il materialized path permette di trovare tutti i discendenti con una singola regex query (`^/elettronica/componenti`). Il campo `livello` e calcolato dalla profondita del path.

**Alternative considerate:**
- Solo adjacency list → query subtree ricorsive, non praticabile con FerretDB
- Nested sets → insert e move costosi, implementazione complessa, non giustificata per questo volume di dati
- Solo materialized path → perdita della navigazione diretta padre/figlio, piu complesso per CRUD singolo nodo

### D2: TreeTable con dialog per CRUD

**Decisione:** layout a pagina singola con TreeTable UI5 per visualizzazione albero, dialog modali per creazione/modifica nodi.

**Motivazione:** la TreeTable mostra naturalmente la gerarchia con expand/collapse. Il dialog permette di editare un nodo alla volta senza cambiare contesto. Pattern coerente con l'app articoli (tabella + dialog).

### D3: Due ruoli (viewer, admin)

**Decisione:** client `categorie-spa` con ruoli viewer (sola lettura) e admin (CRUD completo).

**Motivazione:** le categorie sono master data gestito da pochi admin. Non serve un ruolo intermedio "user" — o consulti (viewer) o gestisci tutto (admin). L'app articoli usa il token dell'utente corrente per leggere le categorie: basta che abbia almeno viewer su categorie-spa.

**Matrice permessi:**
```
             GET    POST   PUT    DELETE
viewer        x
admin         x      x      x      x
```

### D4: Check integrita cross-collection su DELETE

**Decisione:** il flow DELETE di categorie esegue una query diretta sulla collection `articoli` in FerretDB per verificare che nessun articolo referenzi il `categoriaId`. Se trovati, la cancellazione e bloccata con errore 409 Conflict.

**Motivazione:** entrambe le collection (`categorie` e `articoli`) risiedono nella stessa istanza FerretDB (bishop-nodered-customer). Una query diretta sulla collection e piu semplice e affidabile di una chiamata API cross-app. Non crea dipendenza runtime tra i backend.

**Alternative considerate:**
- Chiamata API al backend articoli → accoppiamento runtime, single point of failure se l'API articoli e giu
- Nessun check (trust) → rischio di categorie cancellate con articoli orfani, inaccettabile perche la regola "blocca cancellazione" e la garanzia che rende sicuro l'approccio trust lato articoli

### D5: Path applicazione /categorie/

**Decisione:** frontend su `/categorie/app/`, API su `/categorie/api/`.

**Motivazione:** naming italiano coerente con il dominio. Pattern Bishop standard con separazione app/api.

### D6: Blocca cancellazione nodi con figli

**Decisione:** un nodo non puo essere cancellato se ha figli diretti. L'admin deve prima spostare o cancellare i figli. Per nascondere un nodo senza cancellare, usa il flag `attivo = false`.

**Motivazione:** semplice, prevedibile, nessuna magia. Evita situazioni complesse come reparenting automatico o cancellazione a cascata di sotto-alberi interi. Il flag `attivo` copre il caso "non voglio piu questa categoria ma ha dati storici".

**Alternative considerate:**
- Reparenting automatico dei figli al nonno → complessita, riscrittura path per tutto il subtree
- Cancellazione a cascata → pericolosa, un click cancella potenzialmente decine di nodi

### D7: Riscrittura path su spostamento nodo

**Decisione:** quando un nodo viene spostato (cambio parentId), il backend riscrive il `path` e il `livello` del nodo e di tutti i suoi discendenti.

**Motivazione:** il materialized path deve restare coerente. Lo spostamento e un'operazione rara (admin-only), quindi il costo della riscrittura e accettabile. La query per trovare i discendenti usa il prefix match sul vecchio path.

### D8: Vincolo solo foglie per articoli

**Decisione:** gli articoli possono referenziare solo categorie foglia (senza figli). Il vincolo e enforced sia dal picker nell'app articoli (solo foglie selezionabili) sia dal backend categorie (auto-move su creazione figlio).

**Motivazione:** garantisce classificazione precisa — ogni articolo appartiene alla categoria piu specifica disponibile. Senza questo vincolo, articoli potrebbero restare su nodi generici anche quando esistono sotto-categorie piu appropriate.

**Alternative considerate:**
- Articoli ovunque nell'albero → semplice ma perde la garanzia di classificazione precisa
- Articoli ovunque + warning → flessibile ma dati temporaneamente inconsistenti

### D9: Auto-move articoli su creazione figlio

**Decisione:** quando POST /categorie crea un figlio sotto un nodo che ha articoli assegnati, il backend sposta automaticamente tutti gli articoli dal padre al nuovo figlio (UPDATE cross-collection su `articoli` SET categoriaId = nuovoId WHERE categoriaId = padreId). La risposta 201 include `movedArticlesCount`.

**Motivazione:** mantiene il vincolo "solo foglie" senza bloccare l'evoluzione dell'albero. L'alternativa "blocca creazione figlio se il padre ha articoli" era troppo rigida: costringeva l'admin a riassegnare manualmente tutti gli articoli prima di poter ristrutturare l'albero.

**Trade-off:** il flow POST ora esegue una write cross-collection (prima solo il DELETE faceva una read cross-collection). L'operazione e rara (solo quando un nodo con articoli diventa padre) e il volume di update e contenuto.

**Alternative considerate:**
- Blocca creazione figlio → UX rigida, costringe a riassegnare manualmente prima
- Forza riassegnazione interattiva → UX complessa, dialog articolato, scope della change esplode

### D10: Dialog conferma auto-move

**Decisione:** il frontend mostra un dialog di conferma prima di creare un figlio sotto una categoria con articoli. Il dialog indica il numero di articoli che verranno spostati: "La categoria 'X' ha N articoli assegnati. Creando una sotto-categoria, gli articoli verranno automaticamente spostati nella nuova categoria. Continuare?"

**Motivazione:** l'auto-move e un side-effect non ovvio della creazione di un figlio. L'admin deve essere consapevole che gli articoli cambieranno categoria. Il conteggio viene dal campo `articlesCount` gia presente nella risposta GET.

## Risks / Trade-offs

- **[FerretDB maturity]** FerretDB e meno maturo di MongoDB nativo → Mitigazione: solo operazioni CRUD semplici, regex per prefix match su path
- **[Riscrittura path su move]** Spostare un nodo con molti discendenti richiede update multipli → Mitigazione: operazione rara, volume dati contenuto, batch update in singola transazione
- **[Check cross-collection]** Il flow DELETE accede direttamente alla collection `articoli` → Mitigazione: solo una query find di conteggio, read-only, nessun rischio di corruzione dati
- **[Nessun test automatizzato]** → Mitigazione: accettabile per progetto didattico, test manuali via `.http`
- **[Write cross-collection su POST]** Il flow POST accede in scrittura alla collection `articoli` per l'auto-move → Mitigazione: operazione rara (solo quando si aggiunge un figlio a un nodo con articoli), update batch in singola operazione
