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

## Nodi aperti

- **Cancellazione categoria con articoli sotto**: bloccare? spostare? lasciare orfani?
- **UI**: TreeTable da bishop-app-hierarchy come riferimento?
- **Relazione inter-app**: articoli deve chiamare endpoint categorie per il picker

## Prossimi passi

- Creare una nuova change proposal `gestione-categorie`
- Definire entita, API, ruoli, UI
- Chiarire la dipendenza con app articoli (modifica del campo `categoria` da enum a riferimento)
