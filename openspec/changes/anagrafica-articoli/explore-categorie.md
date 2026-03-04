# Esplorazione: Gestione Categorie Articoli

Data: 2026-03-03

## Contesto

Attualmente le categorie sono un enum fisso nella spec di `anagrafica-articoli`:

```
materia-prima | semilavorato | prodotto-finito | consumabile | altro
```

L'utente vuole passare a una gestione categorie strutturata.

## Decisioni emerse

| Domanda | Risposta |
|---|---|
| Chi gestisce le categorie? | Admin, da una UI dedicata |
| L'articolo punta a quale livello? | Solo alla foglia |
| Quanti livelli servono? | Profondita arbitraria (N livelli) |
| E' parte della change articoli? | No, e' una seconda app separata |

## Architettura a due app

```
  ┌─────────────────────┐         ┌─────────────────────┐
  │   App Categorie     │         │   App Articoli      │
  │   /categorie/       │         │   /articoli/        │
  ├─────────────────────┤         ├─────────────────────┤
  │                     │         │                     │
  │  Tree gerarchico    │         │  AnalyticalTable    │
  │  CRUD nodi          │   ref   │  categoriaId ───────┤
  │  N livelli          │◀────────│  filtro per categ.  │
  │                     │         │                     │
  └─────────────────────┘         └─────────────────────┘
           │                                 │
           ▼                                 ▼
  ┌─────────────────┐              ┌──────────────────┐
  │  categorie      │              │  articoli        │
  ├─────────────────┤              ├──────────────────┤
  │ id              │              │ id               │
  │ nome            │◀─────────────│ categoriaId      │
  │ parentId (null) │              │ codice           │
  │ ordine          │              │ descrizione      │
  │ livello         │              │ ...              │
  │ path            │              └──────────────────┘
  │ attivo          │
  │ audit fields    │
  └─────────────────┘
```

## Modello dati: albero a N livelli con FerretDB

FerretDB non supporta `$graphLookup` ne aggregation pipeline complesse. Approccio consigliato: **ibrido adjacency list + materialized path**.

| Campo | Scopo |
|---|---|
| `parentId` | Navigazione padre/figlio |
| `path` (es. `/elettronica/componenti/resistenze`) | Filtro subtree con prefix match |
| `livello` | Calcolato dalla profondita del path |

Confronto approcci:

| | Adjacency List | Materialized Path | Nested Sets |
|---|---|---|---|
| Pro | Semplice, CRUD facile | Query antenati veloci, filtro con prefix | Query subtree velocissime |
| Con | Query subtree ricorsive (lente) | Riscrittura path su spostamento | Insert/move costosi, complesso |
| FerretDB | Non praticabile da solo | Buon fit con prefix match | Troppo complesso |

## Decisioni aggiuntive (dalla sessione di explore)

| Domanda | Risposta |
|---|---|
| Sviluppo sequenziale o parallelo? | Assieme: Categorie backend first, poi Articoli completa + Categorie UI |
| Modello dati articoli: enum o riferimento? | Strada B: `categoriaId` (UUID ref) dal giorno 1, nessun enum, zero migrazione |
| Cancellazione categoria con articoli sotto? | Blocca (A) + Soft delete (C): non puoi cancellare se ha articoli o figli; usa flag `attivo` per nascondere |
| Cancellazione nodo intermedio con figli? | Blocca: devi prima spostare o cancellare i figli |
| Validazione categoriaId lato backend articoli? | Trust: il frontend manda sempre ID validi dal picker; la regola "blocca cancellazione se ha articoli" previene orfani |
| Accoppiamento runtime inter-app? | Accettabile: Articoli chiama GET /categorie/api/ per il picker |

## Nodi chiusi

- ~~Cancellazione categoria con articoli sotto~~: **blocca + soft delete**
- ~~Relazione inter-app~~: **chiamata diretta API, accoppiamento accettabile**
- ~~Validazione categoriaId~~: **trust, nessuna verifica backend**

---

## Sessione 2026-03-04: Vincolo foglie e auto-move

### Problema emerso

Se una categoria foglia ha articoli e l'admin crea figli sotto di essa, quegli articoli non sono piu su una foglia. Il vincolo "articoli solo su foglie" viene violato silenziosamente.

```
PRIMA                              DOPO
Resistenze [Art-001, Art-002]      Resistenze        <- non piu foglia!
  (foglia)                         +-- SMD            <- nuova foglia
                                   +-- Through-Hole   <- nuova foglia
                                   Art-001, Art-002 dove vanno?
```

### Opzioni valutate

| Approccio | Come funziona | Pro | Contro |
|---|---|---|---|
| Blocca | Non puoi creare figli sotto una categoria con articoli | Semplice | UX rigida |
| Forza riassegnazione | Obbliga a riassegnare gli articoli ai nuovi figli | Coerente | UX complessa |
| Rilassa il vincolo | Articoli ovunque nell'albero | Semplice | Perde precisione classificazione |
| Tollera + segnala | Ovunque ma con warning | Flessibile | Dati temporaneamente inconsistenti |
| **Auto-move** | Articoli si spostano al primo figlio creato | Mantiene vincolo automaticamente | Write cross-collection su POST |

### Decisioni emerse (sessione 2)

| # | Domanda | Risposta |
|---|---|---|
| D8 | Articoli su nodi intermedi? | **No** — solo foglie, vincolo enforced |
| D9 | Cosa succede quando foglia diventa padre? | **Auto-move**: articoli si spostano automaticamente al primo figlio creato |
| D10 | Il frontend avvisa prima dell'auto-move? | **Si**, dialog di conferma con conteggio articoli |
| D11 | Picker categorie nell'app articoli | Albero completo, **solo foglie selezionabili** (intermedie disabilitate) |

### Flusso auto-move in dettaglio

```
1. POST /categorie { nome: "SMD", parentId: "resistenze-id" }
2. Backend: padre "Resistenze" ha articoli? -> query articoli WHERE categoriaId = parentId
3. Trova Art-001, Art-002 -> crea categoria "SMD"
4. UPDATE articoli SET categoriaId = "smd-id" WHERE categoriaId = "resistenze-id"
5. Risponde 201 con movedArticlesCount: 2

Risultato:
   Resistenze
   +-- SMD [Art-001, Art-002]   <- eredita articoli dal padre
```

### Nodi chiusi (sessione 2)

- ~~Articoli su nodi intermedi~~: **no, solo foglie**
- ~~Foglia diventa padre con articoli~~: **auto-move al primo figlio**
- ~~Conferma auto-move~~: **si, dialog con conteggio**
- ~~Picker articoli~~: **albero completo, solo foglie selezionabili**

## Prossimi passi

- [x] Aggiornare artifact change `anagrafica-articoli` (proposal, design, spec, tasks)
- [x] Creare nuova change proposal `gestione-categorie`
- [x] Definire entita Categoria, API albero, ruoli, UI TreeTable
- [ ] Aggiornare artifact `gestione-categorie` con decisioni D8-D10 (design, spec, tasks)
- [ ] Aggiornare artifact `anagrafica-articoli` con decisione D11 picker (design, spec, tasks)
