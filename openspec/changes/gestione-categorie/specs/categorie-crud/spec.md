## ADDED Requirements

### Requirement: Entita Categoria
Il sistema DEVE gestire un'entita "Categoria" che rappresenta un nodo in una struttura ad albero gerarchico di profondita arbitraria. Ogni categoria DEVE avere: id (UUID), nome, parentId (UUID o null per radici), path materializzato, livello (calcolato), ordine di visualizzazione, stato attivo/inattivo e campi di audit.

#### Scenario: Creazione categoria radice
- **WHEN** l'admin crea una categoria con nome "Elettronica", parentId null
- **THEN** il sistema persiste la categoria con id UUID, path "/elettronica", livello 0, ordine 0, attivo = true, campi audit valorizzati

#### Scenario: Creazione categoria figlia
- **WHEN** l'admin crea una categoria con nome "Componenti", parentId = id di "Elettronica"
- **THEN** il sistema persiste la categoria con path "/elettronica/componenti", livello 1, parentId valorizzato

#### Scenario: Creazione categoria con nome duplicato sotto stesso padre
- **WHEN** l'admin tenta di creare una categoria "Componenti" sotto "Elettronica" quando esiste gia
- **THEN** il sistema rifiuta con errore 409 Conflict e messaggio "Categoria con questo nome esiste gia sotto lo stesso padre"

#### Scenario: Valori predefiniti alla creazione
- **WHEN** l'admin crea una categoria senza specificare ordine e attivo
- **THEN** il sistema imposta ordine = 0 e attivo = true come valori predefiniti

### Requirement: Validazione campi Categoria
Il sistema DEVE validare i campi della categoria prima della persistenza.

#### Scenario: Nome mancante
- **WHEN** l'admin invia una richiesta POST/PUT senza nome
- **THEN** il sistema risponde 400 Bad Request con indicazione del campo mancante

#### Scenario: parentId non valido
- **WHEN** l'admin invia una categoria con parentId che non corrisponde a nessuna categoria esistente
- **THEN** il sistema risponde 400 Bad Request con messaggio "Categoria padre non trovata"

#### Scenario: Riferimento circolare
- **WHEN** l'admin tenta di spostare una categoria sotto uno dei suoi discendenti
- **THEN** il sistema risponde 400 Bad Request con messaggio "Spostamento non valido: riferimento circolare"

### Requirement: API REST CRUD categorie
Il sistema DEVE esporre endpoint REST per la gestione CRUD delle categorie, accessibili tramite APISIX gateway con autenticazione JWT.

#### Scenario: Lista completa categorie (flat)
- **WHEN** il client richiede `GET /categorie`
- **THEN** il sistema risponde 200 con `{ data: Categoria[] }` contenente tutte le categorie ordinate per path e ordine. Il frontend ricostruisce l'albero lato client

#### Scenario: Filtro per stato attivo
- **WHEN** il client richiede `GET /categorie?attivo=true`
- **THEN** il sistema restituisce solo le categorie attive

#### Scenario: Dettaglio categoria
- **WHEN** il client richiede `GET /categorie/:id`
- **THEN** il sistema risponde 200 con la categoria completa

#### Scenario: Categoria non trovata
- **WHEN** il client richiede una categoria con ID inesistente
- **THEN** il sistema risponde 404 Not Found

#### Scenario: Creazione categoria
- **WHEN** il client invia `POST /categorie` con body valido
- **THEN** il sistema valida, calcola path e livello dal padre, persiste su FerretDB, e risponde 201 Created

#### Scenario: Creazione categoria sotto nodo con articoli (auto-move)
- **WHEN** il client invia `POST /categorie` con parentId di una categoria che ha articoli assegnati
- **THEN** il sistema crea la nuova categoria E sposta automaticamente tutti gli articoli dal padre alla nuova categoria (UPDATE articoli SET categoriaId = nuovo-id WHERE categoriaId = padre-id)
- **AND** risponde 201 Created con campo aggiuntivo `movedArticlesCount: N`

#### Scenario: Conteggio articoli per categoria
- **WHEN** il client richiede `GET /categorie`
- **THEN** ogni categoria nella risposta include il campo `articlesCount` (numero di articoli con quel categoriaId)
- _Nota: serve al frontend per il dialog di conferma auto-move e per la colonna "Articoli" nella TreeTable_

#### Scenario: Aggiornamento categoria (rename)
- **WHEN** il client invia `PUT /categorie/:id` con nome modificato
- **THEN** il sistema aggiorna nome, riscrive il path del nodo e di tutti i discendenti, e risponde 200

#### Scenario: Spostamento categoria (cambio padre)
- **WHEN** il client invia `PUT /categorie/:id` con parentId modificato
- **THEN** il sistema verifica assenza di riferimenti circolari, riscrive path e livello del nodo e di tutti i discendenti, e risponde 200

#### Scenario: Eliminazione categoria senza figli e senza articoli
- **WHEN** il client invia `DELETE /categorie/:id` per una categoria senza figli e senza articoli referenziati
- **THEN** il sistema elimina la categoria e risponde 204 No Content

#### Scenario: Eliminazione categoria con figli
- **WHEN** il client invia `DELETE /categorie/:id` per una categoria che ha figli diretti
- **THEN** il sistema risponde 409 Conflict con messaggio "Impossibile eliminare: la categoria ha sotto-categorie"

#### Scenario: Eliminazione categoria con articoli referenziati
- **WHEN** il client invia `DELETE /categorie/:id` per una categoria referenziata da articoli nella collection `articoli`
- **THEN** il sistema risponde 409 Conflict con messaggio "Impossibile eliminare: la categoria e utilizzata da N articoli"

### Requirement: Controllo accessi basato su ruoli
Il sistema DEVE limitare le operazioni in base al ruolo Keycloak dell'utente. I ruoli sono estratti da `resource_access[categorie-spa].roles` del JWT.

#### Scenario: Viewer accede in sola lettura
- **WHEN** un utente con ruolo "viewer" richiede `GET /categorie`
- **THEN** il sistema risponde 200 con la lista

#### Scenario: Viewer tenta creazione
- **WHEN** un utente con ruolo "viewer" invia `POST /categorie`
- **THEN** il sistema risponde 403 Forbidden

#### Scenario: Admin ha accesso completo
- **WHEN** un utente con ruolo "admin" invia qualsiasi richiesta CRUD
- **THEN** il sistema esegue l'operazione con successo

### Requirement: Frontend con TreeTable
Il frontend DEVE presentare le categorie in una TreeTable UI5 con visualizzazione gerarchica espandibile.

#### Scenario: Visualizzazione albero categorie
- **WHEN** l'utente accede alla pagina principale
- **THEN** il sistema mostra una TreeTable con colonne: Nome, Livello, Stato, Articoli (conteggio), Azioni
- **AND** le categorie radice sono visibili, i figli accessibili tramite expand

#### Scenario: Expand/collapse nodo
- **WHEN** l'utente clicca sull'icona expand di una categoria
- **THEN** la TreeTable mostra i figli diretti di quella categoria

#### Scenario: Badge stato categoria
- **WHEN** una categoria ha attivo = true
- **THEN** il sistema mostra un badge verde "Attiva"
- **WHEN** una categoria ha attivo = false
- **THEN** il sistema mostra un badge rosso "Inattiva"

#### Scenario: Pulsanti azione visibili per ruolo
- **WHEN** l'utente ha ruolo "viewer"
- **THEN** non sono visibili pulsanti "Nuova Categoria", "Modifica", "Elimina"
- **WHEN** l'utente ha ruolo "admin"
- **THEN** tutti i pulsanti sono visibili

### Requirement: Dialog CRUD per categorie
Il frontend DEVE usare dialog modali UI5 per creazione e modifica categorie.

#### Scenario: Apertura dialog creazione (radice)
- **WHEN** l'admin clicca "Nuova Categoria" dalla toolbar
- **THEN** si apre un dialog con campi: Nome (Input), Padre (Select con categorie esistenti, opzione "Nessuno - radice"), Ordine (Input number), Attivo (Switch)

#### Scenario: Apertura dialog creazione (sotto-categoria)
- **WHEN** l'admin clicca "Aggiungi figlia" su una categoria esistente
- **THEN** si apre il dialog di creazione con il campo Padre precompilato con la categoria selezionata

#### Scenario: Salvataggio categoria da dialog
- **WHEN** l'admin compila i campi e clicca "Salva"
- **THEN** il sistema crea la categoria, chiude il dialog, mostra toast "Categoria creata", aggiorna la TreeTable

#### Scenario: Apertura dialog modifica
- **WHEN** l'admin clicca "Modifica" su una categoria
- **THEN** si apre un dialog con i campi precompilati. Il campo Padre permette di spostare la categoria (con validazione anti-circolarita)

#### Scenario: Conferma creazione sotto-categoria con auto-move
- **WHEN** l'admin clicca "Aggiungi figlia" su una categoria che ha articoli (articlesCount > 0)
- **THEN** il sistema mostra un dialog di conferma: "La categoria 'X' ha N articoli assegnati. Creando una sotto-categoria, gli articoli verranno automaticamente spostati nella nuova categoria. Continuare?"
- **WHEN** l'admin conferma
- **THEN** si apre il dialog di creazione con parentId precompilato

#### Scenario: Conferma eliminazione
- **WHEN** l'admin clicca "Elimina" su una categoria
- **THEN** il sistema mostra un dialog di conferma con MessageBox (mai window.confirm)
- **WHEN** l'admin conferma e la categoria ha figli o articoli
- **THEN** il dialog mostra il messaggio di errore restituito dal backend
