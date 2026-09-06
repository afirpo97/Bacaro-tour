# BRIEF OPERATIVO — Menu Assistant V0

Documento di specifica per l'implementazione. Destinatario: agente di sviluppo.
Versione 1.1 — 6 settembre 2026.

---

## 1. OBIETTIVO

Costruire una applicazione web locale che, a partire da un inventario alimentare gestito dall'utente, generi **tre proposte di pasto** utilizzando un modello linguistico, **validi deterministicamente** l'output prima di mostrarlo, e presenti le proposte in una interfaccia leggibile.

Il valore del sistema non è la generazione: è la **catena di controlli deterministici attorno alla generazione**. La parte generativa è un solo passo di un flusso in cui tutto il resto è codice verificabile.

Principio guida, valido per ogni decisione implementativa:

> Il modello propone. Il codice verifica. L'utente decide.

---

## 2. SCOPE

La V0 deve fare esattamente questo:

1. Visualizzare l'inventario degli alimenti.
2. Aggiungere un alimento (nome, stato, scadenza opzionale).
3. Modificare stato e scadenza di un alimento esistente.
4. Eliminare un alimento.
5. Calcolare **in codice** giorni alla scadenza e flag `inScadenza` a partire dalla data.
6. Raccogliere i parametri della richiesta: pasto, persone presenti, tempo disponibile, acquisti consentiti.
7. Inviare al modello **soltanto dati strutturati**, mai testo libero dell'utente.
8. Generare tre proposte.
9. Validare l'output contro uno schema e contro regole di conformità semantica.
10. Mostrare le proposte in una interfaccia responsive, leggibile anche a larghezza 380px.
11. Mostrare errori distinti e comprensibili per: input non valido, timeout, quota, output non conforme strutturalmente, output non conforme semanticamente, errore del provider.
12. Non modificare l'inventario in conseguenza della generazione.
13. Funzionare integralmente, compresa la generazione, **senza chiave API**, tramite un provider finto deterministico.

## 3. FUORI SCOPE — NON IMPLEMENTARE

Nessuna di queste voci va aggiunta, nemmeno "predisposta", nemmeno commentata come TODO nel codice:

- autenticazione, login, gestione utenti, sessioni;
- inventari personali, proprietari degli alimenti, permessi, visibilità differenziata;
- aggiornamento automatico dell'inventario dopo la selezione di una proposta;
- lista della spesa;
- preferenze alimentari, storico dei pasti, allergie o intolleranze;
- notifiche, scheduling, esecuzioni ricorrenti;
- database (SQL, ORM, Prisma, SQLite);
- deploy, Docker, CI, hosting;
- gestione dello stato globale (Redux, Zustand, Jotai);
- librerie di componenti (shadcn/ui, MUI, Chakra, Radix);
- retry automatici, circuit breaker, code, backoff;
- lock distribuiti, gestione della concorrenza fra processi;
- streaming della risposta del modello;
- internazionalizzazione, tema scuro, animazioni;
- rigenerazione automatica in caso di qualità insufficiente;
- iterazione conversazionale sulle proposte.

L'interfaccia utente è **in italiano**. Il codice, i nomi di variabili e i commenti sono in italiano o inglese, purché coerenti.

---

## 4. STACK

- Next.js (App Router), TypeScript in modalità strict
- Tailwind CSS
- Zod per la validazione degli schemi
- Vitest per i test
- Persistenza: **un singolo file JSON** su filesystem locale (vincoli in §6.4)
- Provider del modello: **FakeProvider** (predefinito) e **GeminiProvider** via API HTTP, entrambi dietro una interfaccia astratta

Nessuna dipendenza oltre a queste senza una ragione dichiarata.

---

## 5. STRUTTURA DELLE CARTELLE

```
/app
  page.tsx                    Inventario (schermata principale)
  genera/page.tsx             Form richiesta + risultati
  api/
    inventario/route.ts       GET, POST
    inventario/[id]/route.ts  PATCH, DELETE
    proposte/route.ts         POST
/lib
  tipi.ts                     Tipi e schemi Zod
  inventario.ts               Lettura/scrittura atomica del file JSON
  seedDemo.ts                 Generazione deterministica dei dati demo
  scadenze.ts                 Calcolo giorniAllaScadenza e inScadenza
  validazione.ts              Validazione semantica dell'output
  ordinamento.ts              Ordinamento deterministico delle proposte
  prompt.ts                   System prompt e composizione del payload
  config.ts                   Soglie e costanti configurabili
  providers/
    tipi.ts                   Interfaccia ModelProvider ed errori tipizzati
    fake.ts                   FakeProvider deterministico
    gemini.ts                 Implementazione Gemini
    index.ts                  getModelProvider()
/scripts
  reset-demo.ts               Rigenera /dati/inventario.json
/dati
  inventario.json             Generato, non versionato
/test
  scadenze.test.ts
  seedDemo.test.ts
  validazione.test.ts
  ordinamento.test.ts
  schema.test.ts
```

Non creare altre cartelle. Non creare una cartella `components` finché non esistono almeno due componenti realmente riutilizzati.

---

## 6. MODELLO DEI DATI

### 6.1 Alimento (come persistito)

| Campo | Tipo | Note |
|---|---|---|
| `id` | string | generato, stabile |
| `nome` | string | non vuoto |
| `stato` | `"pieno" \| "a metà" \| "quasi finito"` | esattamente questi valori |
| `scadenza` | string ISO `YYYY-MM-DD` oppure `null` | `null` = non deperibile |

Non esiste lo stato "finito": un alimento finito viene eliminato.

### 6.2 AlimentoArricchito (calcolato, mai persistito)

Aggiunge:

| Campo | Tipo | Calcolo |
|---|---|---|
| `giorniAllaScadenza` | number \| null | differenza in giorni fra `scadenza` e oggi; `null` se `scadenza` è `null` |
| `inScadenza` | boolean | `true` se `giorniAllaScadenza !== null && giorniAllaScadenza <= SOGLIA_SCADENZA` |

`SOGLIA_SCADENZA` sta in `config.ts`, valore iniziale **3**. Deve essere modificabile in un punto solo.

**Vincolo architetturale:** l'arricchimento avviene sempre lato server, prima della composizione del payload. Il modello riceve `inScadenza` già calcolato e **non deve mai interpretare le date**.

### 6.3 Dati demo generati a runtime

**Non salvare date assolute nel codice sorgente.** Un dataset con date fisse invecchia: dopo pochi giorni tutti gli alimenti risultano scaduti e i test smettono di verificare ciò per cui erano stati scritti.

In `/lib/seedDemo.ts`:

```ts
export function inizializzaInventarioDemo(dataRiferimento: Date = new Date()): Alimento[]
```

La funzione deve essere **pura e deterministica rispetto a `dataRiferimento`**: nessuna lettura dell'orologio al suo interno se il parametro è fornito. È il requisito che la rende testabile.

Offset in giorni rispetto a `dataRiferimento`:

| id | Nome | Stato | Offset |
|---|---|---|---|
| a1 | Yogurt bianco | quasi finito | +1 |
| a2 | Zucchine | a metà | +2 |
| a3 | Pane | pieno | +2 |
| a4 | Insalata | pieno | +3 |
| a5 | Uova | pieno | +10 |
| a6 | Cipolla | pieno | +15 |
| a7 | Parmigiano | quasi finito | +25 |
| a8 | Riso | pieno | `null` |
| a9 | Pasta | pieno | `null` |
| a10 | Pomodori pelati | pieno | `null` |

Il dataset è progettato, non casuale: quattro alimenti entro la soglia di 3 giorni, tre basi diverse disponibili (riso, pasta, pane/uova) perché tre proposte differenti siano effettivamente possibili, e uno yogurt in scadenza fra un giorno difficile da collocare in una cena — serve a verificare che il modello lo segnali come non utilizzato invece di forzarlo.

**Quando viene invocata:**
1. all'avvio, se `/dati/inventario.json` non esiste;
2. dal comando `npm run reset-demo`, che rigenera il file sovrascrivendolo.

Aggiungere in `package.json`:

```json
"reset-demo": "tsx scripts/reset-demo.ts"
```

Aggiungere `/dati/inventario.json` a `.gitignore`: è un file generato.

### 6.4 Persistenza — vincoli espliciti

La persistenza su file JSON è accettata **esclusivamente per esecuzione locale a processo singolo**.

1. **Validare in lettura.** Il contenuto del file viene analizzato con Zod prima dell'uso. Se non è valido, non tentare riparazioni: fallire con un errore chiaro che indica il percorso del file e suggerisce `npm run reset-demo`.
2. **Scrittura atomica.** Scrivere prima in un file temporaneo **nella stessa cartella** (`inventario.json.tmp`), poi rinominarlo su `inventario.json`. Il rename nella stessa partizione è atomico e impedisce di lasciare un file troncato in caso di interruzione.
3. **Nessun lock distribuito, nessuna gestione della concorrenza.** Fuori scope.
4. **Documentare nel README**, in una sezione dedicata: questa persistenza non è adatta a deployment serverless né multiistanza, perché ogni istanza avrebbe un filesystem proprio e le scritture concorrenti si sovrascriverebbero. La sostituzione della persistenza è il primo intervento richiesto da un eventuale deploy.

### 6.5 Ingredienti base

In `config.ts`:

```ts
export const INGREDIENTI_BASE = ["acqua", "olio", "sale", "pepe"];
```

Sempre disponibili senza comparire nell'inventario. Servono alla validazione semantica (§9). L'elenco è configurabile e **non deve essere deciso dal modello**.

---

## 7. SCHEMI ZOD

In `/lib/tipi.ts`. Gli schemi sono la fonte di verità: i tipi TypeScript si derivano con `z.infer`, non si scrivono a mano.

### Input della richiesta

```ts
export const RichiestaSchema = z.object({
  pasto: z.enum(["colazione", "pranzo", "cena", "spuntino"]),
  personePresenti: z.number().int().min(1).max(10),
  tempoDisponibile: z.number().int().min(5).max(180),
  acquistiAggiuntivi: z.enum(["nessuno", "minimi", "consentiti"]),
});
```

Nessun campo di testo libero. È una decisione deliberata: impedisce che richieste soggettive o fuori perimetro raggiungano il modello.

### Output del modello

```ts
export const PropostaSchema = z.object({
  idProposta: z.string(),
  titolo: z.string().min(1),
  descrizioneBreve: z.string().min(1),
  basePrincipale: z.string().min(1),
  ingredientiUtilizzati: z.array(z.string()).min(1),
  alimentiInScadenzaUtilizzati: z.array(z.string()),
  ingredientiMancanti: z.array(z.string()),
  tempoPreparazioneStimato: z.number().int().positive(),
  motivazione: z.string().min(1),
});

export const AlimentoNonUtilizzatoSchema = z.object({
  nome: z.string(),
  motivo: z.string(),
});

export const RispostaModelloSchema = z.object({
  stato: z.enum(["proposte_generate", "proposte_complete_non_disponibili"]),
  proposte: z.array(PropostaSchema).max(3),
  alimentiInScadenzaNonUtilizzati: z.array(AlimentoNonUtilizzatoSchema),
  motivoStatoDegradato: z.string().nullable(),
  ingredientiMinimiDaAcquistare: z.array(z.string()),
});
```

**Regola di coerenza da verificare in codice, non nello schema:** se `stato === "proposte_generate"` allora `proposte.length === 3`.

---

## 8. PROVIDER DEL MODELLO

### 8.1 Interfaccia

In `/lib/providers/tipi.ts`:

```ts
export interface RisultatoModello {
  testoGrezzo: string;      // il JSON come stringa, già estratto
  modello: string;
  latenzaMs: number;
}

export interface ModelProvider {
  nome: string;             // "fake" | "gemini"
  etichetta: string;        // "Modalità demo" | "Gemini API"
  genera(params: {
    systemPrompt: string;
    payload: unknown;
    timeoutMs: number;
  }): Promise<RisultatoModello>;
}
```

**Vincolo fondamentale, non negoziabile.** Tutto ciò che è specifico di Gemini — URL, forma del corpo della richiesta, intestazioni, **e soprattutto la navigazione della struttura di risposta per estrarre il testo** — vive esclusivamente in `gemini.ts`. Il resto dell'applicazione conosce solo `testoGrezzo`.

Questo è il punto in cui, in una implementazione ingenua, si annida l'accoppiamento al fornitore. Un provider alternativo deve poter essere aggiunto scrivendo un solo file, senza toccare nient'altro.

### 8.2 Selezione del provider

In `/lib/providers/index.ts`, **un solo punto**, lato server:

```ts
export function getModelProvider(): ModelProvider
```

Regole:

- variabile d'ambiente `MODEL_PROVIDER` con valori ammessi `fake` | `gemini`;
- **valore predefinito in assenza della variabile: `fake`**;
- `gemini` viene usato **soltanto** se `MODEL_PROVIDER=gemini` **e** `GEMINI_API_KEY` è presente e non vuota;
- se `MODEL_PROVIDER=gemini` ma la chiave manca, non ricadere silenziosamente su `fake`: fallire con `ERRORE_PROVIDER` e messaggio esplicito. Un ripiego silenzioso su dati finti sarebbe indistinguibile dal funzionamento reale, ed è la modalità di guasto peggiore possibile.

Nessuna factory, nessuna dependency injection, nessun registro di provider. Una funzione con un `if`.

### 8.3 FakeProvider

In `/lib/providers/fake.ts`. Utilizzabile solo in locale e nei test.

- Restituisce una risposta **deterministica e valida** con tre proposte costruite sui dati demo, conforme a `RispostaModelloSchema` e alle regole semantiche di §9.
- Non chiama nulla all'esterno. Latenza simulata: 300 ms circa, per rendere visibile lo stato "in corso" dell'interfaccia.
- Le tre proposte devono usare basi principali diverse e valorizzare almeno un alimento con `inScadenza === true`, così che il percorso felice sia collaudabile fino in fondo senza chiave API.
- Deve rispettare il vincolo di tempo ricevuto: se `tempoDisponibile` è troppo basso perché le tre proposte siano compatibili, restituire `proposte_complete_non_disponibili` con il motivo. Serve a collaudare lo stato degradato dell'interfaccia.

### 8.4 GeminiProvider

Errori tipizzati da distinguere e propagare: `TIMEOUT`, `QUOTA` (429), `ERRORE_PROVIDER` (altri status non-2xx), `RISPOSTA_ILLEGGIBILE` (struttura di risposta inattesa).

Nessun retry automatico. Timeout iniziale: **60000 ms**, in `config.ts`.

---

## 9. VALIDAZIONE

Tre livelli, in quest'ordine, tutti lato server, tutti prima di mostrare qualcosa all'utente. **Si applicano identicamente a entrambi i provider**: il provider finto non è esente dalla validazione, altrimenti il percorso di controllo non verrebbe mai esercitato in sviluppo.

### Livello 1 — Strutturale

Il testo grezzo viene ripulito da eventuali delimitatori di blocco di codice, analizzato come JSON, e validato con `RispostaModelloSchema`. Fallimento → `OUTPUT_NON_CONFORME`.

### Livello 2 — Semantica

In `/lib/validazione.ts`. Regole bloccanti:

1. **Sottoinsieme degli ingredienti.** Ogni elemento di `ingredientiUtilizzati` deve corrispondere a un alimento dell'inventario oppure a un elemento di `INGREDIENTI_BASE`. Confronto normalizzato: minuscolo, spazi esterni rimossi, accenti conservati. Violazione → `INGREDIENTE_INVENTATO`, con l'elenco degli ingredienti offensivi.
2. **Rispetto del tempo.** `tempoPreparazioneStimato <= tempoDisponibile` per ogni proposta. Violazione → `TEMPO_SUPERATO`.
3. **Rispetto degli acquisti.** Se `acquistiAggiuntivi === "nessuno"`, ogni `ingredientiMancanti` deve essere vuoto. Se `"minimi"`, al massimo 2 elementi per proposta. Violazione → `ACQUISTI_NON_CONSENTITI`.
4. **Coerenza del conteggio.** `stato === "proposte_generate"` implica esattamente 3 proposte. Violazione → `CONTEGGIO_INCOERENTE`.

Nessun retry, nessuna rigenerazione: la V0 mostra l'errore all'utente.

### Livello 3 — Qualità, non bloccante

Calcolare e **registrare in console lato server**, senza bloccare né mostrare:

- `cq01`: almeno una proposta usa l'alimento con `inScadenza === true` e il minor `giorniAllaScadenza`;
- `cq04`: le tre proposte hanno `basePrincipale` diverse a coppie;
- `cq06`: numero di `basePrincipale` distinte.

Metriche di osservazione, non criteri di accettazione. Non devono influenzare il comportamento del sistema.

---

## 10. ORDINAMENTO

In `/lib/ordinamento.ts`, applicato **dopo** la validazione, in codice, mai dal modello:

1. numero di alimenti in scadenza utilizzati, decrescente;
2. numero di ingredienti mancanti, crescente;
3. tempo di preparazione stimato, crescente.

A parità completa, si conserva l'ordine di arrivo. La funzione deve essere pura e deterministica.

---

## 11. SYSTEM PROMPT

In `/lib/prompt.ts`, come costante esportata, **in un solo punto**, versionata con una costante `VERSIONE_PROMPT`.

```
Sei il componente che formula proposte alternative per il pasto richiesto.

Gli alimenti ricevuti sono già stati filtrati e verificati dal sistema. Considerali gli unici
alimenti disponibili e utilizzabili. Non eseguire ulteriori controlli su sicurezza alimentare
o autorizzazioni. La classificazione inScadenza è già stata calcolata: non interpretare le date.

Oltre agli alimenti ricevuti puoi utilizzare esclusivamente questi ingredienti base, sempre
disponibili: acqua, olio, sale, pepe.

1. Genera tre proposte realizzabili per il pasto richiesto, tenendo conto del numero di persone
   e del tempo disponibile.
2. Utilizza prioritariamente gli alimenti con inScadenza true, quando sono compatibili con il
   pasto e con gli altri dati ricevuti. Non forzarne l'utilizzo in combinazioni incoerenti.
   Segnala gli alimenti in scadenza che non compaiono in nessuna proposta, indicando il motivo.
3. Produci proposte sostanzialmente differenti per base del pasto, tecnica di preparazione o
   combinazione prevalente degli ingredienti. Non considerare differenti due proposte che
   cambiano soltanto titolo, condimento o guarnizione.
4. Non presentare come disponibile alcun ingrediente assente dall'elenco ricevuto e dagli
   ingredienti base. Se un ingrediente assente può essere acquistato, indicalo esclusivamente
   fra gli ingredienti mancanti.
5. Rispetta il vincolo ricevuto sugli acquisti aggiuntivi. Se gli acquisti non sono consentiti,
   nessuna proposta può richiedere ingredienti mancanti.
6. Proponi esclusivamente alternative il cui tempo stimato non superi il tempo disponibile.
7. Non inventare né completare silenziosamente dati mancanti.
8. Se non puoi produrre tre alternative sostanzialmente differenti e realizzabili, produci
   soltanto quelle valide, imposta lo stato a proposte_complete_non_disponibili e indica il
   motivo e, se gli acquisti sono consentiti, i soli ingredienti minimi necessari.

Rispondi esclusivamente con un oggetto JSON conforme allo schema fornito. Nessun testo prima
o dopo il JSON.
```

**Nota per l'implementatore:** questo prompt non contiene istruzioni su allergie, autorizzazioni o preferenze. Non è una dimenticanza: quei vincoli sono fuori scope nella V0 o garantiti a monte. **Non aggiungerli.**

Il payload inviato contiene solo: alimenti arricchiti (nome, stato, giorniAllaScadenza, inScadenza), pasto, personePresenti, tempoDisponibile, acquistiAggiuntivi, ingredientiBase.

**Non inviare mai al provider** la configurazione dell'applicazione: nome del provider, variabili d'ambiente, percorsi di file, versione del prompt.

---

## 12. API ROUTES

| Metodo | Percorso | Corpo | Risposta |
|---|---|---|---|
| GET | `/api/inventario` | — | `AlimentoArricchito[]` |
| POST | `/api/inventario` | `{nome, stato, scadenza}` | alimento creato |
| PATCH | `/api/inventario/[id]` | `{stato?, scadenza?}` | alimento aggiornato |
| DELETE | `/api/inventario/[id]` | — | `{ok: true}` |
| POST | `/api/proposte` | `Richiesta` | vedi sotto |

Ogni route valida l'input con Zod prima di fare qualunque cosa.

### 12.1 Corpo della risposta

Il corpo mantiene **sempre** una struttura discriminata, qualunque sia lo status HTTP.

Successo, e anche stato degradato funzionale:

```json
{ "esito": "ok", "stato": "proposte_generate", "proposte": [...],
  "alimentiInScadenzaNonUtilizzati": [...], "motivoStatoDegradato": null,
  "provider": "fake", "latenzaMs": 312 }
```

Errore:

```json
{ "esito": "errore", "codice": "TIMEOUT",
  "messaggioUtente": "...", "dettaglioTecnico": "..." }
```

### 12.2 Status HTTP

| Status | Codici |
|---|---|
| **200** | successo **e stato degradato funzionale** — il degradato non è un errore |
| **400** | `INPUT_NON_VALIDO` |
| **422** | `OUTPUT_NON_CONFORME`, `INGREDIENTE_INVENTATO`, `TEMPO_SUPERATO`, `ACQUISTI_NON_CONSENTITI`, `CONTEGGIO_INCOERENTE` |
| **429** | `QUOTA` |
| **502** | `ERRORE_PROVIDER`, `RISPOSTA_ILLEGGIBILE` |
| **504** | `TIMEOUT` |
| **500** | errori interni non previsti |

`INVENTARIO_VUOTO` è un caso di input non valido: **400**.

**Nota per la diagnosi.** Gli status `429` e `504` coincidono con quelli che possono essere generati da un proxy o dall'infrastruttura. Lo status serve al client; il campo `codice` nel corpo resta la fonte di verità per i log e per la diagnosi.

### 12.3 Lettura lato client

L'interfaccia deve leggere **sia lo status HTTP sia il corpo JSON**:

- status 2xx → distinguere `stato` per scegliere fra la vista "completato" e quella "degradato";
- status non-2xx → leggere `codice` dal corpo e mostrare il messaggio previsto in §15;
- corpo assente o non analizzabile su uno status non-2xx → messaggio generico di errore, mai una schermata bianca né una eccezione non gestita.

---

## 13. SCHERMATE

### Inventario (`/`)

Elenco degli alimenti con nome, stato, e indicazione della scadenza. Gli alimenti con `inScadenza === true` sono visivamente distinti (bordo o etichetta, non solo colore del testo). Ordinamento: prima quelli in scadenza, per giorni crescenti; poi gli altri; infine i non deperibili.

Per ogni riga: modifica dello stato, modifica della scadenza, eliminazione con conferma.
In testa: form di aggiunta.
In fondo: pulsante primario "Genera menù" verso `/genera`.

### Genera (`/genera`)

Form con quattro campi, tutti con valore predefinito già selezionato:

- pasto: derivato dall'ora corrente (colazione < 11, pranzo < 16, cena < 22, altrimenti spuntino), modificabile;
- persone presenti: predefinito 2;
- tempo disponibile: predefinito 30 minuti;
- acquisti aggiuntivi: predefinito "nessuno" — valore prudenziale.

Un solo pulsante di invio. Sotto il form compare il risultato.

**Indicatore del provider.** In posizione discreta ma sempre visibile, una piccola etichetta: **"Modalità demo"** oppure **"Gemini API"**. È l'unico modo per distinguere a colpo d'occhio una proposta finta da una reale; senza di essa, un provider finto attivo per errore sarebbe invisibile.

Il valore va risolto **lato server** e passato alla pagina, oppure incluso nel campo `provider` della risposta di `/api/proposte`. **Non usare variabili con prefisso pubblico** e non esporre altre parti della configurazione.

### Risultato

Tre schede, nell'ordine deciso dal codice. Ogni scheda mostra: titolo, descrizione breve, ingredienti utilizzati (con evidenza di quelli in scadenza), tempo stimato, eventuali ingredienti mancanti, motivazione.

Sotto le proposte, se presente, la sezione degli alimenti in scadenza non utilizzati con il motivo.

**Nessun pulsante di selezione, approvazione o conferma.** La V0 non compie azioni: la presenza di un pulsante che non fa nulla sarebbe fuorviante.

Layout responsive, verificabile a 380px di larghezza. Nessun elemento interattivo sotto i 44px di lato.

---

## 14. STATI DELL'INTERFACCIA

La schermata di generazione ha esattamente cinque stati, tutti da implementare:

1. **inattivo** — form pronto, nessun risultato;
2. **in corso** — indicatore di attesa esplicito, form disabilitato, testo che dichiara che la generazione può richiedere fino a un minuto;
3. **completato** — tre proposte visibili;
4. **degradato** — meno di tre proposte, con il motivo dichiarato; **non è un errore e non va presentato come tale**;
5. **errore** — nessuna proposta, messaggio specifico per codice.

Lo stato 2 non è un dettaglio estetico: la chiamata può durare decine di secondi e un'interfaccia muta è indistinguibile da un guasto.

---

## 15. GESTIONE DEGLI ERRORI

| Codice | Status | Causa | Messaggio all'utente |
|---|---|---|---|
| `INPUT_NON_VALIDO` | 400 | validazione Zod dell'input | indicare il campo da correggere |
| `INVENTARIO_VUOTO` | 400 | nessun alimento | "Aggiungi almeno un alimento prima di generare un menù." |
| `TIMEOUT` | 504 | oltre 60 s | "Il servizio non ha risposto in tempo. Nessun dato è stato modificato. Puoi riprovare." |
| `QUOTA` | 429 | 429 dal provider | "Limite di richieste raggiunto. Riprova fra qualche minuto." |
| `ERRORE_PROVIDER` | 502 | altri errori HTTP, chiave assente con `MODEL_PROVIDER=gemini` | "Il servizio di generazione non è disponibile. Riprova più tardi." |
| `RISPOSTA_ILLEGGIBILE` | 502 | struttura di risposta inattesa | "La risposta del servizio non è stata interpretabile. Nessuna proposta è stata mostrata." |
| `OUTPUT_NON_CONFORME` | 422 | JSON non analizzabile o schema fallito | "La risposta ricevuta non ha il formato previsto. Nessuna proposta è stata mostrata." |
| `INGREDIENTE_INVENTATO` | 422 | validazione semantica | "Le proposte contenevano ingredienti non disponibili e non sono state mostrate." |
| `TEMPO_SUPERATO` | 422 | validazione semantica | messaggio specifico |
| `ACQUISTI_NON_CONSENTITI` | 422 | validazione semantica | messaggio specifico |
| `CONTEGGIO_INCOERENTE` | 422 | validazione semantica | messaggio specifico |

Ogni errore registra lato server: codice, timestamp, versione del prompt, provider, modello, latenza, e in caso di errore semantico l'elenco delle violazioni. Nessun dato dell'utente nei log oltre a quanto necessario.

**Regola generale:** ogni messaggio di errore deve dire cosa è successo e che nulla è stato modificato. Nessun messaggio tecnico grezzo nell'interfaccia.

---

## 16. TEST MINIMI

Solo logica deterministica. Nessun test che chiami un modello reale. `FakeProvider` può essere usato nei test.

**`scadenze.test.ts`** — `giorniAllaScadenza` per data futura, oggi, passata, `null`; `inScadenza` a 0, 3, 4 giorni (verifica del confine della soglia); indipendenza dal fuso orario.

**`seedDemo.test.ts`** — con una `dataRiferimento` fissa, il seed produce esattamente gli offset attesi; due invocazioni con la stessa data producono lo stesso risultato; Riso, Pasta e Pomodori pelati hanno `scadenza === null`; con la soglia a 3, esattamente quattro alimenti risultano `inScadenza`.

**`validazione.test.ts`** — ingrediente inventato rilevato; ingrediente base accettato; confronto insensibile a maiuscole e spazi; tempo superato rilevato; acquisti non consentiti rilevati; conteggio incoerente rilevato; **caso completamente valido che passa** (un validatore che rifiuta tutto supererebbe gli altri test); **l'output di `FakeProvider` supera tutta la validazione**.

**`ordinamento.test.ts`** — ordine per criterio 1; parità risolta dal criterio 2; parità doppia risolta dal criterio 3; parità totale conserva l'ordine; funzione pura, l'array in ingresso non viene mutato.

**`schema.test.ts`** — output valido accettato; campo mancante rifiutato; tipo errato rifiutato; quarta proposta rifiutata; JSON avvolto in delimitatori di blocco di codice ripulito e accettato.

Obiettivo: tutti i test verdi con `npm test`, e nessun errore con `tsc --noEmit`.

---

## 17. CRITERI DI ACCETTAZIONE

La V0 è completa quando **tutti** questi punti sono verificati:

1. `npm run dev` avvia l'applicazione senza errori **e senza alcuna variabile d'ambiente impostata**.
2. `tsc --noEmit` non produce errori.
3. `npm test` passa integralmente.
4. Al primo avvio l'inventario mostra i dieci alimenti demo, con i quattro in scadenza evidenziati, **con date coerenti con il giorno di esecuzione**.
5. `npm run reset-demo` rigenera il file e le date risultano ricalcolate sulla data corrente.
6. Aggiunta, modifica ed eliminazione di un alimento persistono nel file JSON e sopravvivono al riavvio.
7. Con `MODEL_PROVIDER` non impostato, una generazione con cena, 2 persone, 30 minuti, acquisti "nessuno" produce tre proposte visibili e l'etichetta "Modalità demo".
8. Le proposte del provider finto superano l'intera validazione, strutturale e semantica.
9. Nessuna proposta contiene ingredienti fuori dall'inventario e dagli ingredienti base.
10. Nessuna proposta supera il tempo indicato.
11. Con `tempoDisponibile` a 5 minuti il sistema risponde in modo pulito: stato degradato o errore comprensibile, **mai** proposte implausibili presentate come valide.
12. Con `MODEL_PROVIDER=gemini` e chiave assente o errata compare `ERRORE_PROVIDER` con status 502, non una schermata bianca, non una eccezione non gestita, **e non un ripiego silenzioso sul provider finto**.
13. Gli status HTTP corrispondono alla tabella di §12.2, verificabili dal pannello di rete del browser.
14. Interrompendo il processo durante una scrittura, `inventario.json` resta valido e analizzabile.
15. L'interfaccia è utilizzabile a 380px di larghezza.
16. La chiave API non compare in nessun file versionato né in nessuna risposta inviata al browser.

---

## 18. ORDINE DI IMPLEMENTAZIONE

Rigorosamente sequenziale. Ogni passo si conclude con test verdi e controllo dei tipi pulito prima di iniziare il successivo.

1. Inizializzazione del progetto, dipendenze, configurazione Tailwind e Vitest.
2. `tipi.ts`, `config.ts`, `scadenze.ts`, `seedDemo.ts`, script `reset-demo` + i relativi test.
3. `inventario.ts` con lettura validata e scrittura atomica, e le API route dell'inventario.
4. Schermata inventario completa e funzionante.
5. `providers/tipi.ts`, `providers/fake.ts`, `providers/index.ts` con `getModelProvider()`.
6. `prompt.ts`, `validazione.ts`, `ordinamento.ts` + i relativi test.
7. `api/proposte/route.ts` con status HTTP corretti, funzionante **end-to-end con il provider finto**.
8. Schermata di generazione con i cinque stati e l'indicatore del provider.
9. `providers/gemini.ts` con errori tipizzati. **È il primo passo che richiede una chiave API.**
10. Verifica dei sedici criteri di accettazione.

**Fino al passo 8 compreso l'applicazione è completa e collaudabile senza alcuna dipendenza esterna, senza chiave e senza consumo.** Se il passo 9 dovesse rivelarsi problematico, la V0 resta un artefatto funzionante e dimostrabile.

---

## 19. VINCOLI DI SICUREZZA

1. La chiave API sta esclusivamente in `.env.local`, sotto il nome `GEMINI_API_KEY`.
2. `.env.local` è in `.gitignore`. Creare `.env.example` con `MODEL_PROVIDER=fake` e `GEMINI_API_KEY=` vuota.
3. La chiave è usata **solo** in codice server. Mai in un componente client, mai in una variabile con prefisso pubblico, mai in una risposta inviata al browser.
4. La chiamata al modello avviene solo da API route lato server. Il browser non contatta mai direttamente il provider.
5. Al browser può essere esposto **soltanto** il nome del provider e la sua etichetta. Nessun'altra parte della configurazione.
6. Nessuna chiave, nessun segreto e nessun URL completo con credenziali nei log.
7. Le route validano ogni input con Zod prima di qualunque operazione.
8. Il payload inviato al provider non contiene configurazione dell'applicazione (§11).
9. I dati sono sintetici. Nessuna telemetria, nessun invio a terzi oltre alla chiamata al modello.

---

## 20. ISTRUZIONI OPERATIVE

Procedi in autonomia:

1. **Crea direttamente i file.** Non chiedere conferma per ogni file, non mostrare il codice in chat prima di scriverlo.
2. **Installa le dipendenze** necessarie.
3. **Esegui** `tsc --noEmit` e `npm test` dopo ogni passo dell'ordine di implementazione.
4. **Avvia l'applicazione** e verifica che risponda.
5. **Correggi autonomamente** errori di compilazione, di tipo, di test e di runtime. Non riportare all'utente un errore che puoi risolvere.
6. **Fermati e chiedi** soltanto davanti a una **decisione funzionale non presente in questa specifica** — cioè quando la specifica è ambigua o silente su un comportamento che l'utente deve vedere. Non fermarti per decisioni implementative: quelle prendile tu, scegliendo l'opzione più semplice.
7. **Non aggiungere funzionalità non richieste.** Se pensi che qualcosa manchi, elencalo a fine lavoro in una sezione "osservazioni", senza implementarlo.

---

## 21. ISTRUZIONI PER NON SOVRA-INGEGNERIZZARE

Questa applicazione serve a tre persone, per circa venti generazioni al mese, e sarà probabilmente sostituita. Ogni astrazione non richiesta è un costo senza contropartita.

- **Nessuna astrazione al primo utilizzo.** Un helper si estrae quando serve la terza volta, non la prima.
- **Nessun livello di servizio, repository, factory, dependency injection.** Funzioni esportate da moduli, importate dove servono. `getModelProvider()` è una funzione con un `if`, non un registro estensibile.
- **Una sola astrazione è obbligatoria: l'interfaccia del provider** (§8). È l'unica richiesta esplicitamente, perché protegge un requisito dichiarato di sostituibilità.
- **Nessun componente creato "perché forse servirà".** Scrivi il markup direttamente nella pagina finché non c'è duplicazione reale.
- **Nessuna gestione dei casi limite non elencata in §15.**
- **Nessun commento che ripeta il codice.** Commenta solo il perché di una scelta non ovvia.
- **Preferisci sempre la soluzione più corta che soddisfa il criterio di accettazione.**
- Se ti accorgi di scrivere codice che nessuno dei sedici criteri di accettazione verifica, fermati: probabilmente è fuori scope.

---

## 22. OSSERVAZIONI FINALI RICHIESTE

Al termine, produci una breve sezione con:

- comandi per avviare applicazione, test e `reset-demo`;
- come passare da provider finto a Gemini e come tornare indietro;
- cosa manca rispetto al sistema completo, senza implementarlo;
- eventuali punti in cui la specifica è risultata ambigua e quale interpretazione hai adottato;
- eventuali dipendenze aggiunte oltre allo stack dichiarato, con la ragione.
