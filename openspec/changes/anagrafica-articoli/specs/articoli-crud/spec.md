## ADDED Requirements

### Requirement: Entita Articolo
Il sistema DEVE gestire un'entita "Articolo" che rappresenta un prodotto nel catalogo aziendale. Ogni articolo DEVE avere: codice univoco, descrizione, categoriaId (riferimento UUID a entita Categoria dell'app gestione-categorie), unita di misura, prezzo unitario, stato attivo/inattivo, note opzionali e campi di audit (createdAt, updatedAt, createdBy, updatedBy).

#### Scenario: Creazione articolo con campi obbligatori
- **WHEN** l'utente crea un articolo con codice "ART-001", descrizione "Bullone M8", categoriaId "uuid-categoria-materia-prima", unitaMisura "PZ", prezzoUnitario 0.50
- **THEN** il sistema persiste l'articolo con id UUID generato, attivo = true, e campi audit valorizzati

#### Scenario: Creazione articolo con codice duplicato
- **WHEN** l'utente tenta di creare un articolo con codice "ART-001" gia esistente
- **THEN** il sistema rifiuta con errore 409 Conflict e messaggio "Codice articolo gia esistente"

#### Scenario: Valori predefiniti alla creazione
- **WHEN** l'utente crea un articolo senza specificare il campo attivo
- **THEN** il sistema imposta attivo = true come valore predefinito

### Requirement: Validazione campi Articolo
Il sistema DEVE validare i campi dell'articolo prima della persistenza.

#### Scenario: Campi obbligatori mancanti
- **WHEN** l'utente invia una richiesta POST/PUT senza codice o descrizione
- **THEN** il sistema risponde 400 Bad Request con indicazione dei campi mancanti

#### Scenario: Prezzo unitario negativo
- **WHEN** l'utente invia un articolo con prezzoUnitario = -5
- **THEN** il sistema risponde 400 Bad Request con messaggio "prezzoUnitario deve essere >= 0"

#### Scenario: Unita di misura non valida
- **WHEN** l'utente invia un articolo con unitaMisura = "GALLONI"
- **THEN** il sistema risponde 400 Bad Request con messaggio "unitaMisura deve essere uno tra: PZ, KG, LT, MT, M2, M3"

#### Scenario: categoriaId mancante
- **WHEN** l'utente invia un articolo senza categoriaId
- **THEN** il sistema risponde 400 Bad Request con messaggio "categoriaId e obbligatorio"

_Nota: il backend NON valida l'esistenza del categoriaId nella collection categorie (approccio trust). Il frontend garantisce ID validi tramite il TreeSelect picker. La regola "blocca cancellazione categoria con articoli sotto" nell'app categorie previene orfani._

#### Scenario: categoriaId punta a categoria non-foglia
- **WHEN** l'utente tenta di assegnare un articolo a una categoria intermedia (con figli)
- **THEN** il picker non lo permette (la categoria e disabilitata)
- _Nota: il backend NON valida che la categoria sia foglia (approccio trust). Il vincolo e enforced solo lato frontend dal picker._

### Requirement: API REST CRUD articoli
Il sistema DEVE esporre endpoint REST per la gestione CRUD degli articoli, accessibili tramite APISIX gateway con autenticazione JWT.

#### Scenario: Lista paginata
- **WHEN** il client richiede `GET /articoli?page=1&limit=20`
- **THEN** il sistema risponde 200 con `{ data: Articolo[], pagination: { page, limit, totalItems, totalPages, hasNextPage, hasPrevPage } }`

#### Scenario: Ricerca testuale
- **WHEN** il client richiede `GET /articoli?search=bullone`
- **THEN** il sistema filtra per corrispondenza parziale case-insensitive su codice e descrizione

#### Scenario: Filtro per categoria
- **WHEN** il client richiede `GET /articoli?categoriaId=uuid-categoria`
- **THEN** il sistema restituisce solo gli articoli con quel categoriaId

#### Scenario: Filtro per stato attivo
- **WHEN** il client richiede `GET /articoli?attivo=true`
- **THEN** il sistema restituisce solo gli articoli con attivo = true

#### Scenario: Combinazione filtri
- **WHEN** il client richiede `GET /articoli?search=bull&categoriaId=uuid-categoria&attivo=true&page=1&limit=10`
- **THEN** il sistema applica tutti i filtri contemporaneamente e restituisce la pagina richiesta

#### Scenario: Creazione articolo
- **WHEN** il client invia `POST /articoli` con body valido
- **THEN** il sistema valida, persiste su FerretDB, e risponde 201 Created con l'articolo creato

#### Scenario: Dettaglio articolo
- **WHEN** il client richiede `GET /articoli/:id`
- **THEN** il sistema risponde 200 con l'articolo completo

#### Scenario: Articolo non trovato
- **WHEN** il client richiede un articolo con ID inesistente
- **THEN** il sistema risponde 404 Not Found

#### Scenario: Aggiornamento articolo
- **WHEN** il client invia `PUT /articoli/:id` con body valido
- **THEN** il sistema valida (inclusa unicita codice), aggiorna su FerretDB, e risponde 200 con l'articolo aggiornato

#### Scenario: Eliminazione articolo
- **WHEN** il client invia `DELETE /articoli/:id`
- **THEN** il sistema elimina l'articolo da FerretDB e risponde 204 No Content

### Requirement: Controllo accessi basato su ruoli
Il sistema DEVE limitare le operazioni in base al ruolo Keycloak dell'utente. I ruoli sono estratti da `resource_access[articoli-spa].roles` del JWT.

#### Scenario: Viewer accede in sola lettura
- **WHEN** un utente con ruolo "viewer" richiede `GET /articoli`
- **THEN** il sistema risponde 200 con la lista

#### Scenario: Viewer tenta creazione
- **WHEN** un utente con ruolo "viewer" invia `POST /articoli`
- **THEN** il sistema risponde 403 Forbidden

#### Scenario: User puo creare e modificare
- **WHEN** un utente con ruolo "user" invia `POST /articoli` o `PUT /articoli/:id`
- **THEN** il sistema esegue l'operazione con successo

#### Scenario: User tenta eliminazione
- **WHEN** un utente con ruolo "user" invia `DELETE /articoli/:id`
- **THEN** il sistema risponde 403 Forbidden

#### Scenario: Admin ha accesso completo
- **WHEN** un utente con ruolo "admin" invia qualsiasi richiesta CRUD
- **THEN** il sistema esegue l'operazione con successo

### Requirement: Integrazione con API Categorie
Il frontend DEVE consumare l'API dell'app `gestione-categorie` per ottenere l'albero delle categorie. Il backend articoli NON valida l'esistenza del categoriaId (approccio trust).

#### Scenario: Picker mostra albero completo con solo foglie selezionabili
- **WHEN** l'utente apre il dialog di creazione o modifica articolo
- **THEN** il frontend richiede `GET /categorie/api/categorie` e popola il TreeSelect picker con l'intero albero delle categorie attive
- **AND** solo le categorie foglia (senza figli) sono selezionabili
- **AND** le categorie intermedie (con figli) sono visibili ma disabilitate (grigie, non cliccabili)
- _Nota: la distinzione foglia/non-foglia si calcola lato frontend dal tree (una categoria e foglia se nessun'altra categoria ha il suo id come parentId)_

#### Scenario: Risoluzione nome categoria in tabella
- **WHEN** la tabella articoli viene renderizzata
- **THEN** il frontend risolve ogni `categoriaId` nel nome leggibile della categoria (ottenuto dall'albero categorie gia in cache)

#### Scenario: API categorie non raggiungibile
- **WHEN** l'API categorie non e raggiungibile
- **THEN** il picker categoria mostra un messaggio di errore, ma il resto dell'app articoli rimane funzionante. In tabella, il categoriaId viene mostrato come fallback al posto del nome

### Requirement: Dati condivisi senza filtro owner
L'anagrafica articoli e master data condiviso. Il sistema NON DEVE filtrare gli articoli per utente creatore. Tutti gli utenti con il ruolo appropriato vedono tutti gli articoli.

#### Scenario: Utenti diversi vedono gli stessi articoli
- **WHEN** utente A con ruolo "viewer" e utente B con ruolo "viewer" richiedono `GET /articoli`
- **THEN** entrambi ricevono la stessa lista completa di articoli

### Requirement: Frontend con AnalyticalTable
Il frontend DEVE presentare gli articoli in una AnalyticalTable UI5 con toolbar di ricerca e filtri.

#### Scenario: Visualizzazione lista articoli
- **WHEN** l'utente accede alla pagina principale
- **THEN** il sistema mostra una AnalyticalTable con colonne: Codice, Descrizione, Categoria (nome risolto da API categorie), U.M., Prezzo, Stato, Azioni

#### Scenario: Ricerca dalla toolbar
- **WHEN** l'utente digita "bullone" nel campo di ricerca
- **THEN** la tabella si aggiorna mostrando solo gli articoli che contengono "bullone" in codice o descrizione

#### Scenario: Filtro categoria dalla toolbar
- **WHEN** l'utente seleziona una categoria dal filtro (alimentato da API categorie)
- **THEN** la tabella mostra solo articoli con quel categoriaId

#### Scenario: Badge stato articolo
- **WHEN** un articolo ha attivo = true
- **THEN** il sistema mostra un badge verde "Attivo"
- **WHEN** un articolo ha attivo = false
- **THEN** il sistema mostra un badge rosso "Inattivo"

#### Scenario: Pulsanti azione visibili per ruolo
- **WHEN** l'utente ha ruolo "viewer"
- **THEN** non sono visibili pulsanti "Nuovo Articolo", "Modifica", "Elimina"
- **WHEN** l'utente ha ruolo "user"
- **THEN** sono visibili "Nuovo Articolo" e "Modifica" ma non "Elimina"
- **WHEN** l'utente ha ruolo "admin"
- **THEN** tutti i pulsanti sono visibili

### Requirement: Dialog CRUD per articoli
Il frontend DEVE usare dialog modali UI5 per creazione e modifica articoli.

#### Scenario: Apertura dialog creazione
- **WHEN** l'utente clicca "Nuovo Articolo"
- **THEN** si apre un dialog con campi vuoti: Codice, Descrizione, Categoria (TreeSelect picker alimentato da API categorie, selezione solo foglie), U.M. (Select), Prezzo (Input number), Attivo (Switch), Note (TextArea)

#### Scenario: Salvataggio articolo da dialog
- **WHEN** l'utente compila i campi e clicca "Salva"
- **THEN** il sistema crea l'articolo, chiude il dialog, mostra toast "Articolo creato", e aggiorna la tabella

#### Scenario: Apertura dialog modifica
- **WHEN** l'utente clicca "Modifica" su un articolo
- **THEN** si apre un dialog con i campi precompilati con i dati dell'articolo

#### Scenario: Conferma eliminazione
- **WHEN** l'utente clicca "Elimina" su un articolo
- **THEN** il sistema mostra un dialog di conferma con MessageBox (mai window.confirm)
- **WHEN** l'utente conferma
- **THEN** l'articolo viene eliminato, toast "Articolo eliminato", tabella aggiornata
