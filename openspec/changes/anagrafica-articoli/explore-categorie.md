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

## Prossimi passi

- [x] Aggiornare artifact change `anagrafica-articoli` (proposal, design, spec, tasks)
- [ ] Creare nuova change proposal `gestione-categorie`
- [ ] Definire entita Categoria, API albero, ruoli, UI TreeTable
