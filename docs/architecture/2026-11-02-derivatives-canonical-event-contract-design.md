# Design Spec — Contratto universale degli eventi derivati (V0)

**Data:** 2026-11-02  
**Stato:** BOZZA ARCHITETTURALE — non approvata, nessuna modifica del runtime  
**Repository/branch:** `dimkadiver-cpu/GiacenzeCrypto` / `feature/derivatives-integration-design`  
**Documenti correlati:** [Audit architetturale](2026-11-02-derivatives-end-to-end-audit.md); [PRD requisiti](../product/2026-11-02-derivatives-integration-prd.md); [precedente bozza PRD](../product/2026-11-02-derivatives-integration-master-prd.md).  
**Ambito:** ingresso e normalizzazione degli eventi derivati, senza anticipare le scelte H2, Swing o le regole fiscali definitive.

> **Decisione di fondo:** i JSON/adapter interpretano i tracciati degli exchange; il kernel riceve SOLO un modello canonico con semantica esplicita. La stessa informazione può avere più fonti, ma deve produrre un solo effetto patrimoniale. Il tipo di asset (USDT/USDC/EMT), la famiglia di contratto e la qualificazione fiscale sono assi separati.

---

## 1. Obiettivo e confini

Definire il contratto tra **importer/JSON**, **riconciliazione**, **ledger patrimoniale**, **motori di costo fiscale/LIFO**, **motore fiscale dei derivati**, **UI**, **report T/RT**, **persistenza**.

Problema di origine: nei formati reali una riga può essere (a) un differenziale regolato, (b) una fee, (c) una descrizione di ordine senza movimento, (d) una fotografia di posizione, (e) un record composito contenente differenziale, funding, fee e variazione netta del saldo. Il kernel non deve sapere se tale riga proveniva da Bybit, Binance, BingX, OKX o Bitget.

Fuori da questa specifica:
- formule fiscali definitive c-quater e trattamento giuridico di ogni asset di settlement;
- implementazione di H2/sidecar o modifica dei 46 campi legacy;
- UI finale, modelli PDF e routing dei quadri;
- nuovo kernel di pricing delle posizioni, ricostruzione mark-to-market e ordini;
- supporto fiscale ai contratti inversi, strutturati, opzioni, Dual Investment, portfolio margin non convalidati;
- scelta di un linguaggio arbitrario di espressioni per i JSON.

**Perimetro operativo candidato V1:** perpetual/futures lineari con settlement in USDT/USDC, saldi e componenti disponibili in report verificabili. Il contratto deve comunque RICONOSCERE e segnalare altri prodotti senza interpretarli col motore lineare.

## 2. Verifiche della codebase da preservare

| Elemento reale | Implicazione |
|---|---|
| `Importazioni.ColonneTabella=46` | Nessun campo legacy libero da occupare senza ADR e migrazione |
| `[3]` exchange, `[4]` sotto-wallet/account | Distinguere account operativo da gruppo fiscale esistente |
| `[41]` `DocumentiFonte`, `[42]` lignaggio, `[43]` `CommissioniCollegate`, `[44]` `Derivati`, `[45]` `GruppoOperazione` | Provenienza e legami esistenti da riusare; campo 44 è un marcatore, NON evento completo |
| `ImportazioneGenerica.ConfigurazioneImport.causaliDerivati` | Oggi: causale CSV → tipo breve `PNL`, `FUNDING`, `COMMISSIONE`, ecc. |
| `consolidaCausaliPerGiorno` | Attuale aggregazione anticipata: può perdere granularità, timestamp, matching e valore EUR evento per evento |
| Bybit UTA: `TRADE|OPEN → PNL` | Il marcatore corrente può associare una riga che contiene soltanto fee a `PNL`; non è una prova di reddito |
| `EmoneyToken`, `EMONEY`, `Funzioni.RitornaTipoCrypto`, `Prezzi` | Il trattamento dell'asset dipende da identità e decorrenza; USDC nell'elenco standard attuale, USDT non nell'elenco standard |
| `OperazioniCalcolate` | Il gruppo UI derivato da hash/ID non garantisce identità economica fiscale persistente |
| `Calcoli_PlusvalenzeNew`, `Calcoli_RT`, `Principale_GiacenzeaData` | Più percorsi dei lotti: richiedono una semantica comune dei settlement |
| `Backup_Restore.Tabelle` | Ogni eventuale nuovo stato persistente deve entrare esplicitamente nei backup |

**Regola di compatibilità:** la vista legacy può continuare a mostrare `[44]=PNL/FUNDING/...` per gli eventi in cui la categoria è univoca, ma i motori non devono ricavare tutti i valori fiscali o tutti i componenti da quel solo campo. Nessuna migrazione automatica distruttiva dei movimenti storici.

## 3. Quattro concetti indipendenti

**3.1 Asset e classificazione.** L'asset che passa di mano è USDT, USDC, BTC, ecc.; la sua qualificazione `Crypto/EMoney/...` viene dal registro di token/data e dalle regole fiscali esistenti. Non inferire `EMoney` dal suffisso `USD`, né dare per scontato che 1 USDT valga 1 EUR.

**3.2 Contratto.** `instrumentId`, `productKind` (perpetual/future), `payoffKind` (linear/inverse/other/unknown), `underlyingAsset`, `quoteAsset`, `settlementAsset`, `marginAsset`, `contractMultiplier`, `expiry` se noti. La valuta di regolamento **non** si deduce automaticamente dal nome simbolo e non coincide obbligatoriamente con il collaterale; un conto unificato può condividere collaterale fra prodotti.

**3.3 Evento economico.** Un fatto identificabile: fee realmente addebitata, funding, PnL regolato, saldo trasferito, ecc. Una singola riga può descrivere più componenti economiche. Più righe possono essere evidenze dello stesso fatto.

**3.4 Rappresentazione del saldo.** Il movimento di quantità posseduta `deltaBalance` è distinto da esposizione nozionale, `available/locked`, margine riservato, equity e PnL solo latente. Una fotografia di stato non crea, di per sé, alcun flusso.

### 3.5 Differenza fra *regolamento* e *chiusura*

Un PnL regolato può essere registrato prima della chiusura definitiva di un contratto (es. settlement di sessione). Viceversa, un record di esecuzione/open o close può non rappresentare un accredito di PnL nel medesimo formato. Il parser deve seguire i campi effettivi della fonte, non inferire `realized` dalla parola `CLOSE` soltanto.

---

## 4. Layer del contratto canonico

La sequenza concettuale proposta è:

```text
DocumentiFonte / API source (immutabile)
     |
SourceRecord (identità della riga, ruolo del report, fuso, versione parser)
     |
Normalizer JSON + transformazioni riutilizzabili (non fiscale)
     |
ObservedEvent [tipo, strumenti, account, balanceDelta osservato,
               components economici firmati, metadata posizione]
     |
Reconciliation [identità economica canonica, fonti, conflitti, override]
     |
AcceptedEconomicEvent (osservazioni con autorità definita)
     |                           |
     +--> PatrimonialPosting(s)  +--> EconomicComponent(s)
             |                         |
        saldi e lotti              engine fiscale versionato
             |                         |
        W/RW e giacenze            T/RT c-quater (solo policy validata)
```

**Non è un nuovo database imposto.** Il modello può essere costruito come DTO temporaneo, come informazioni persistenti o come combinazione delle due. La persistenza e l'atomicità dovranno essere oggetto di ADR.

### 4.1 Un evento non equivale a una posizione

Una posizione può produrre molti eventi, e un evento (per esempio un funding o una fee) può non essere collegabile a un'unica posizione. `positionId`, `orderId` e `tradeId` sono riferimenti facoltativi e **non** diventano la chiave univoca universale.

### 4.2 Un'operazione può aggregare eventi distinti

Esempio Binance: `REALIZED_PNL`, `FUNDING_FEE`, `COMMISSION` possono essere record separati; non è necessario forzarli in un unico record canonico. Si possono raggruppare per *operazione* quando il legame è dimostrabile, ma ciascun movimento mantiene la propria identità, tempo e importo. `GruppoOperazione` può supportare la navigazione, non sostituire gli identificativi certi.

## 5. Campi richiesti e opzionali

**5.1 SourceRecord — obbligatori se provenienti da file/API**

| Campo | Semantica |
|---|---|
| `sourceDocumentId` o equivalente durevole | Riferimento a `DocumentiFonte` / sessione API |
| `sourceRecordKey` | Identità DENTRO il documento; stabile anche con righe identiche (hash file + posizione originale se manca ID provider) |
| `provider`, `sourceFormat`, `schemaVersion`, `parserVersion` | Regole e origine delle osservazioni |
| `nativeEventId`, `orderId`, `tradeId`, `positionId` | Quando disponibili: non inventare gli ID mancanti |
| `rawTimestamp`, `sourceTimezone`, `occurredAtUtc` | Data originale e normalizzazione deterministica; preservare precisione e offset |
| `recordRole` | `LEDGER` / `EXECUTION` / `POSITION_SNAPSHOT` / `BALANCE_SNAPSHOT` / `TRANSFER_HISTORY` / `UNKNOWN` |
| `rawCause`, `rawPayloadRef` | Causale originale e collegamento verificabile ai dati originali |

La chiave documento+posizione deve distinguere due record originali uguali; la deduplica *economica* tra documenti è un secondo passaggio. Il numero di riga non è sufficiente come identificativo trasversale tra export con ordinamento diverso.

**5.2 ObservedEvent — elementi minimi per un movimento di saldo**

| Campo | Semantica |
|---|---|
| `eventKind` | `BALANCE_CHANGE`, `TRANSFER`, `EXECUTION_INFO`, `MARGIN_STATE`, `SNAPSHOT`, `UNKNOWN` |
| `ownerKey`, `provider`, `accountKey` | Proprietario e account operativo risolto o stato di mancanza |
| `occurredAtUtc`, `sourceRefs[]` | Provenienza e tempo |
| `instrument` | Descriptor del prodotto se disponibile; `UNKNOWN` esplicito se non determinabile |
| `balanceDeltaObserved[]` | Variazioni **firmate** di quantità per asset e account, solo se la fonte le conferma |
| `economicComponents[]` | Scomposizione in categorie e quantità firmate; può essere parziale o non presente |
| `completeness` | `COMPLETE`, `PARTIAL`, `UNDETERMINED` per la scomposizione del saldo |
| `sourceAssertions[]` | Identità, saldo dopo operazione, regole del provider, eventuale checksum |
| `positionMetadata` | Informazioni di fill, side, size, leverage, PnL latente, NON contabilizzate come flussi |

Un `EXECUTION_INFO` senza differenziale può esistere come record informativo; un `BALANCE_CHANGE` deve avere almeno una variazione in asset supportata dalla fonte o una ricostruzione esplicita e verificata. `null`/campo assente non equivale a zero. L'asset della singola componente è obbligatorio se la quantità è nota.

**5.3 AcceptedEconomicEvent — risultato riconciliato**

| Campo | Semantica |
|---|---|
| `canonicalEventId` | Identità persistente stabile e non dipendente dall'ID legacy della riga |
| `sourceRefs[]` | Tutti i record testimoni dello stesso evento |
| `authoritativeFields` | Fonte scelta per ciascun attributo, motivazione e versione regola |
| `postingPlan[]` | Unico insieme di variazioni patrimoniali accettate |
| `components[]` | Componenti economiche accettate, con stato di provenienza |
| `reconciliationStatus` | `RECONCILED`, `MATCHED`, `UNMATCHED`, `AMBIGUOUS`, `CONFLICT`, `INCOMPLETE`, `OVERRIDDEN`, `UNSUPPORTED` |
| `fiscalEligibility` | Valore `PENDING` finché policy/regole/prezzi non sono verificati |

Non confondere `reconciliationStatus` con `fiscalEligibility`. `RECONCILED` indica che le componenti di una sola fonte sono coerenti con il saldo osservato; `MATCHED` indica inoltre corrispondenze tra fonti riconosciute. Un saldo può essere affidabile mentre la classificazione fiscale resta in sospeso.

---

## 6. Catalogo dei componenti ed eventi

**Componenti monetarie di un evento** — valori firmati nella valuta di regolamento o della fee:

| Tipo universale | Descrizione | Note |
|---|---|---|
| `REALIZED_PNL` | Differenziale regolato, positivo o negativo | Non dedurre dal notional; può essere settlement periodico |
| `FUNDING` | Funding pagato/ricevuto | Segno contabile normalizzato |
| `TRADING_FEE` | Commissione effettivamente addebitata | Valuta fee indipendente; negativo per costo |
| `FEE_REBATE` | Rimborso di commissioni | Positivo se effettivamente accreditato |
| `LIQUIDATION_ADJUSTMENT` | Penale/onere di liquidazione separato dal PnL | Non replicare un PnL già riconosciuto |
| `BONUS_OR_CREDIT` | Bonus, voucher, clawback e crediti | Stato fiscal/accounting distinto; non presumere disponibilità piena |
| `OTHER_SETTLEMENT` | Altro regolamento identificato non mappabile | Necessita policy o stato non supportato |
| `UNEXPLAINED_BALANCE_DELTA` | Delta osservato senza scomposizione adeguata | Richiede diagnosi, non reddito automaticamente imponibile |

**Eventi non monetari/informativi:**
- `EXECUTION_OPEN`, `EXECUTION_INCREASE`, `EXECUTION_REDUCE`, `EXECUTION_CLOSE`, `EXECUTION_UNKNOWN`: descrivono operazioni su contratti, non la compravendita dello spot sottostante;
- `MARGIN_LOCK`, `MARGIN_RELEASE`, `MARGIN_STATE`: quantità disponibili/vincolate, salvo trasferimenti effettivi;
- `POSITION_SNAPSHOT`, `BALANCE_SNAPSHOT`: controllo, non cash flow;
- `LIQUIDATION_TRIGGER`, `ADL`, `DELIVERY`: causali di business da collegare ai movimenti effettivi; eventuale regolamento fisico fuori scope iniziale;
- `INTERNAL_TRANSFER`: movimento fra account dello stesso titolare; anche **due postings** (da/account) per una sola operazione; gestione dei lati realmente osservati.

Questa lista è tassonomia del dominio, non nuovi tipi interni `MovimentiCrypto` già implementati. Un evento può avere `eventKind=EXECUTION_INFO` e `TRADING_FEE` senza `REALIZED_PNL`.

---

## 7. Regole contabili: le somme non vanno contate due volte

### 7.1 Significato delle quantità

- `amount > 0`: incremento patrimoniale per l'asset/account indicato.
- `amount < 0`: decremento; conservare il segno provider originale e la regola di normalizzazione.
- Se un provider riporta `fee > 0` per indicare *costo*, la componente normalizzata è `-fee`. Se `fee < 0` indica rebate, la componente può risultare positiva; la regola va documentata per tracciato.
- Vietate conversioni implicite asset→EUR e interi in virgola mobile per riconciliazione.
- Le somme si confrontano **per asset**, non sommando USDT, USDC e BTC tra loro.

### 7.2 Delta osservato vs scomposizione

Per un account, asset e istante contabile:

```text
observedBalanceDelta(asset) = somma componenti monetarie firmate(asset)
                                + differenza esplicitamente riconosciuta(asset)
```

L'uguaglianza è esigibile solo quando la fonte dichiara una scomposizione **completa**. Non introdurre una voce fittizia `OTHER` per chiudere silenziosamente la differenza.

Un evento ha **uno** `postingPlan` patrimoniale; `economicComponents` lo spiegano, non sono ulteriori posting da sommare all'asset già registrato.

### 7.3 Fixture di riferimento Bybit — composito

La documentazione ufficiale Bybit V5 dichiara `change = cashFlow + funding - fee`; `fee` positiva è costo e `funding` positivo è incasso.

**Record sintetico a titolo di esempio:**

```json
{
  "contractVersion": "derivatives-canonical/1",
  "canonicalEventId": "SYNTHETIC-BYBIT-001",
  "sourceRefs": [{"documentId": "DOC_SYNTHETIC_001", "recordKey": "ROW_001", "sourceFormat": "BYBIT_V5"}],
  "recordRole": "LEDGER",
  "eventKind": "BALANCE_CHANGE",
  "occurredAtUtc": "2025-07-10T08:00:00Z",
  "account": {"provider": "Bybit", "operationalId": "UNIFIED", "ownerKey": "OWNER_TEST"},
  "instrument": {"nativeId": "BTCUSDC", "productKind": "PERPETUAL", "payoffKind": "LINEAR",
                  "settlementAsset": "USDC", "marginAsset": "USDC"},
  "balanceDeltaObserved": [{"asset": "USDC", "amount": "95"}],
  "economicComponents": [
    {"type": "REALIZED_PNL", "asset": "USDC", "signedAmount": "100"},
    {"type": "FUNDING", "asset": "USDC", "signedAmount": "-3"},
    {"type": "TRADING_FEE", "asset": "USDC", "signedAmount": "-2"}
  ],
  "completeness": "COMPLETE",
  "reconciliationStatus": "RECONCILED",
  "fiscalEligibility": "PENDING"
}
```

**Controllo:** `100 + (-3) + (-2) = +95 USDC`. **Ledger: +95 USDC una sola volta.** Il totale economico è anch'esso +95 nella valuta originale, non automaticamente +95 EUR fiscali.

**Attenzione:** è una fixture **sintetica del contratto proposto**, non una struttura oggi accettata dal parser, né una risposta ufficiale Bybit.

### 7.4 Fixture equivalente Binance — tre record

Un export `income` con `REALIZED_PNL +100 USDT`, `FUNDING_FEE -3 USDT`, `COMMISSION -2 USDT` può generare **tre eventi canonici di ledger**, ciascuno con una componente, e in totale `+95 USDT`. Non richiede un'unica riga composita. Se gli ordini offrono un dettaglio duplicato degli stessi importi, il dettaglio si collega come prova, **non** come nuovo posting.

### 7.5 Commissione su altro asset

Per una posizione con profitto `+100 USDT` e fee `-0.001 BTC`, i posting restano distinti per asset. La riconciliazione EUR valorizza le componenti secondo fonte, istante e regola fiscale, ma il saldo non può usare `100 - 0.001` come se fossero la stessa valuta.

### 7.6 Trasferimento interno

```text
Fund:    -1.000 USDT
Futures: +1.000 USDT
Totale:       0 USDT   (stesso titolare, nessuna fee)
```

Se le due righe sono presenti in file diversi, riconciliarle senza creare gambe sintetiche in più. Se è presente solo un lato, conservarlo come osservazione e dichiarare `UNMATCHED`; l'eventuale controparte inferita resta identificata come inferenza, non come prova. Il trasferimento interno non è un reddito derivato e non cambia di per sé il costo fiscale.

### 7.7 Equity, saldo, margin, PnL latente

Le grandezze `walletBalance`, `equity`, `availableBalance`, `lockedMargin` e `unrealizedPnl` sono **misure diverse**, non righe da sommare indistintamente. La policy di mapping deve specificare a quale grandezza si riferisce lo snapshot; il controllo di saldo usa soltanto snapshot comparabili con il tipo di posting. Nei conti UTA il collateral può essere condiviso: non creare automaticamente un saldo Futures separato se la fonte ha un solo conto unificato.

---

## 8. Requisiti per il parser JSON estensibile

### 8.1 Cosa deve fare il mapper specifico

- scegliere `recordRole` per report (income ledger, account bill, fill, order history, position snapshot, transfer);
- estrarre `timestamp` con unità e timezone, fonte e versione;
- mappare causale originale + sottotipo/prodotto a categorie canoniche, con regole esplicite;
- distinguere `deltaBalance` già netto, risultato lordo, fee separate o incluse, funding, rebate;
- riconoscere `asset` per ogni componente e `accountKey` senza dedurre dall'asset stesso;
- supportare una riga con N componenti e zero/uno/più asset di posting (es. conversione reale);
- preservare ID origine, ordine, esecuzione, contratto e reference tra report;
- validare requisiti di ciascun formato (versione/API/export, intervallo, colonne obbligatorie, natura del PnL);
- dichiarare cosa non è interpretabile, senza convertirlo a `PNL` o `REWARD` per default.

### 8.2 Limiti accettabili del sistema JSON

Preferire dichiarazioni finite e verificabili: selezione colonne, trasformazioni aritmetiche esatte, convenzioni del segno, condizioni sul tipo, campi netti/lordi, selezione account, chiavi di matching, specifica di struttura del settlement. NON introdurre script eseguibili arbitrari dentro i JSON. Quando una trasformazione richiede logica complessa, usare un adapter generico **versionato e testato**, non codice specifico in `DerivativesTaxEngine`.

Il contratto *specifico* del nuovo JSON (`derivatiV2`, nomi campi e schema), così come il fallback alle vecchie configurazioni, è una decisione di System Architecture Spec: **non è implementato qui**.

### 8.3 Mappatura esempi: provider diversi, stesso modello

| Fonte | Campi/causali documentati | Normalizzazione |
|---|---|---|
| **Bybit V5 Transaction Log** | `cashFlow`, `funding`, `fee`, `change`, `cashBalance`, `category`, `currency`, `transactionTime`, `id`, `orderId`, `tradeId` | Ledger già netto + componenti dettagliate; non sottrarre fee/funding un'altra volta |
| **Binance USDⓈ-M Income History** | `incomeType`: `REALIZED_PNL`, `FUNDING_FEE`, `COMMISSION`, `TRANSFER` ecc.; `income`, `asset`, `time`, ID | Ogni riga reddituale come componente firmata con relativo posting, non spot del sottostante |
| **OKX Bills** | `billId`, `type`/`subType`, `balChg`, `ccy`, `pnl`, `fee`, `instType`, `ts` | Usare `balChg` come delta osservato; interpretare gli altri campi con la regola esatta del sottotipo |
| **Bitget Futures Account Bills** | `productType` (`USDT-FUTURES`, `USDC-FUTURES`, `COIN-FUTURES`), `billId`, `amount`, `fee`, `businessType` | Mappare business type, costo e asset secondo prodotto; per la V1 `COIN-FUTURES` fuori scope |
| **BingX** | Transaction History e Order History: source type e semantica da verificare per versioni di esportazione | Separare ledger (realized PnL/funding/fee) da ordini; non importare gli stessi fatti due volte |

Le implementazioni V1 devono possedere fixture per ogni formato supportato. Le righe per le quali manca una interpretazione certa vengono conservate con `UNKNOWN`/blocco.

### 8.4 Consolidamento per giorno

`ImportazioneGenerica.consolidaCausaliPerGiorno` oggi raggruppa funding/trade in configurazioni Bybit. Per la nuova pipeline: **normalizzare e riconciliare prima**, poi eventualmente aggregare solo la visualizzazione o i report. Un aggregato ammesso come unica fonte deve restare marcato `SOURCE_AGGREGATE`, con granularità e intervallo originari dichiarati e limiti alla valorizzazione per evento: non inventare timestamp intraday.

### 8.5 Documenti storici e retrofit

Il campo 44 degli archivi pregressi prova soltanto che un movimento è marcato come derivato, non prova il dettaglio lordo/netto. La migrazione deve:
1. individuare il documento fonte [41] e recuperare i raw record se disponibili;
2. proporre una **preview** di ricostruzione; associare vecchio movimento e nuovo evento senza doppia giacenza;
3. conservare il movimento originale e l'override utente finché la conversione non è confermata;
4. indicare `INSUFFICIENT_SOURCE` se non c'è dettaglio utilizzabile;
5. non usare il marcatore `PNL` di un `TRADE|OPEN` come prova di PnL diverso da zero.

---

## 9. Identità, deduplica, matching e autorità

### 9.1 Tre identità da NON confondere

1. **SourceRecordKey** = record fisico in un documento/API, identificabile anche senza ID provider.
2. **CanonicalEventId** = un fatto economico riconciliato tra documenti e reimport (identity persistente).
3. **OperationKey** = raggruppamento facoltativo di eventi correlati, es. fill+fee+PnL di una posizione.

L'ID legacy del movimento e la chiave `OP-<ID>` di `OperazioniCalcolate` non devono diventare la chiave fiscale canonica.

### 9.2 Priorità di matching

1. ID native affidabili con *scope* dichiarato (`provider / owner / account / eventKind / nativeId`; gli ID possono essere riutilizzati tra famiglie causali);
2. riferimenti incrociati espliciti (tradeId, transactionId, orderId, transferId), quando documentatamente univoci per quella relazione;
3. attributi coincidenti con vincoli forti e verificati (asset, segno, quantità, istante, contratto) come **candidati**, non automatismo se ambigui;
4. conferma/override umano motivato se rimangono alternative.

Matching tra Transaction History e Order History dipende dal ruolo del report: l'Order History non è automaticamente un secondo ledger. Non scegliere una fonte unica per tutte le colonne; l'autorità è **per campo/ruolo**.

### 9.3 Invarianti di idempotenza

- Importare `A` poi `B`, oppure `B` poi `A`, dà lo stesso insieme di posting canonici.
- Importare N volte lo stesso documento non cambia il ledger.
- Eliminare/annullare un documento rimuove o riassegna le fonti coerentemente, senza perdere override non rimossi e senza lasciare un evento supportato da evidenza inesistente.
- Una modifica documentata cambia l'evento/proiezione e invalida i ricalcoli interessati; i precedenti snapshot dichiarativi restano archiviati e confrontabili.
- I match ambigui non sono forzati da regole di arrotondamento o da tolleranze troppo larghe.

### 9.4 Quadratura per account

Con saldo iniziale **noto e valido**:

```text
Balance(account, asset, t1)
  = Balance(account, asset, t0)
  + sum(accepted postings for account, asset, (t0,t1])
```

Confrontare con un saldo ufficiale di **medesima metrica** (wallet/cash, non equity contenente PnL non realizzato). Un dato storico iniziale mancante è `UNKNOWN`, non zero. La quadratura non richiede che tutte le componenti fiscali siano state approvate.

---

## 10. Contratti verso i motori esistenti

### 10.1 Ledger / patrimonio

Il ledger riceve solo `postingPlan` accettati, legati a asset e account; non prende a input `economicComponents` come una seconda lista da sommare. Il wallet totale e i sotto-account devono essere viste della medesima proprietà.

### 10.2 EMT, costo fiscale e LIFO

Il settlement asset è indicato dall'evento, **non** dal fatto che sia EMT o meno. L'attribuzione di tipo di asset e decorrenza resta al sistema `EMONEY`/registro token; l'impatto sui lotti va esposto tramite un unico contratto semantico condiviso dai motori `Calcoli_PlusvalenzeNew`, `Calcoli_RT` e `Principale_GiacenzeaData`. Se il criterio fiscale del nuovo lotto o dell'asset consumato è in sospeso, mostrare il saldo ma non attribuire implicitamente costo zero e non certificare il report.

### 10.3 Motore fiscale derivati

Input fiscale: `economicComponents` **riconciliate** con quantità, asset, istante di competenza, provenienza, descrittore contratto, semantica netto/lordo. Il motore fiscale sceglie valorizzazione EUR, oneri ammissibili, classificazione c-quater ed esercizio fiscale soltanto attraverso regole normative **approvate e versionate**. Il kernel non contiene nomi exchange.

### 10.4 T/RT e W/RW

Il report specifico c-quater non è un nuovo "RT Futures" indipendente dal sistema dichiarazioni: deve integrarsi con `Dichiarazioni`, rispettando sezione/modello dell'anno, fonti, manifest, validazione, backup e confronto. Non sommare i risultati derivati in `v[19]` come plusvalenza Spot per comodità. W/RW va alimentato dagli asset/rapporti monitorabili secondo una policy distinta: margine e notional non si sommano per automatismo al valore degli asset.

### 10.5 GUI e modifiche

La UI espone in modo leggibile: evento originale, record collegati, asset/account, delta netto, componenti, PnL lordi/netti, stato di riconciliazione e stato fiscale. Modifiche a una classificazione, data, segno, prezzo, associazione o gruppo invalidano i consumatori; l'override non riscrive il documento sorgente.

---

## 11. Test contrattuali obbligatori

| ID | Fixture | Esito vincolante |
|---|---|---|
| EVT-01 | Bybit composito `cashFlow=100, funding=-3, fee=2, change=95` USDC | Un posting +95 e tre componenti `+100,-3,-2` |
| EVT-02 | Binance tre Income History `+100,-3,-2` USDT | Tre posting non duplicati; aggregato +95, EUR con tempi singoli |
| EVT-03 | OKX `balChg` con `pnl`/`fee` su stesso bill | Delta osservato una volta; confronto del sottotipo; mai doppio fee |
| EVT-04 | Bitget `USDT-FUTURES` vs `USDC-FUTURES` | Settlement asset corretto; `COIN-FUTURES` rifiutato dal primo kernel |
| EVT-05 | Trade OPEN a PnL nullo con fee | Nessun PnL fittizio; fee effettiva presente |
| EVT-06 | Trade CLOSE con più fill, documenti e record duplicati | Posting unico per fatto; reference preservate, attributi ordine informativi |
| EVT-07 | Fund -1000, Futures +1000 da due report | Totale invariato; un trasferimento e due lati; nessuna quarta gamba |
| EVT-08 | Solo lato Fund -1000 | Evento `UNMATCHED`, conto incompleto, nessun reddito artificiale |
| EVT-09 | PnL +100 USDT e fee -0.001 BTC | Due asset distinti; valori EUR separati; no somma quantità |
| EVT-10 | Funding dopo apertura, settlement di sessione, chiusura successiva | Soli differenziali regolati, nessuna riclassificazione automatica del latente |
| EVT-11 | Ora Londra 31/12 e Shanghai 01/01 per medesimo record | Un UTC, dedup cross-export, anno secondo policy fiscale validata |
| EVT-12 | CSV originario con consolidato giornaliero | Granularità dichiarata; no timestamp/pricing inventato |
| EVT-13 | USDC in `EmoneyToken` e USDT non presente; decorrenze diverse | Identità asset separata e classificazione EMT dinamica, stesso kernel derivati |
| EVT-14 | Correzione tipo/valore/data nel 2025 dopo report, poi reimport | Snapshot report conservato, ricalcolo e override coerenti |
| EVT-15 | Più import in ordine invertito e N reimport | Identico insieme dei posting e totali, zero effetti aggiuntivi |
| EVT-16 | Dataset senza derivati | LIFO, RT, RW, import/GUI invariati (golden master) |
| EVT-17 | Saldo iniziale ignoto o equity usata come snapshot wallet | Nessuna quadratura falsa; stato incompleto/metric mismatch |
| EVT-18 | Report senza policy costo asset settlement / prezzo EUR | Ledger diagnostico possibile, report dichiarativo NON definitivo |
| EVT-19 | Fee già inclusa in PnL netto e presente in Order History | Un addebito economico, nessun doppio fee |
| EVT-20 | Bonus, liquidazione, consegna fisica o inverso non supportati | Provenienza mantenuta, nessun posting inventato, stato di blocco pertinente |

La suite deve verificare anche invarianti di precisione, timestamp in secondi/millisecondi, identità hash per scope, `BigDecimal`, simulazione JSON senza scrittura e completo vs incrementale.

**Test da NON dichiarare svolti:** non sono stati eseguiti né Maven né GUI/benchmark in questa fase documentale.

---

## 12. Rischi, decisioni aperte e criteri per l'approvazione

| ADR | Questione | Scelta da validare |
|---|---|---|
| ADR-EVT-01 | Persistenza di `SourceRecord/CanonicalEvent` | Evento durevole in H2 oppure sidecar/proiezioni, senza doppio ledger |
| ADR-EVT-02 | Nuova DSL JSON | Sintassi minima dichiarativa, compatibilità `causaliDerivati`, validazione schema e versioni |
| ADR-EVT-03 | Autorità della fonte | Regole per provider/formato, incluse copie multiple e assenza ID univoci |
| ADR-EVT-04 | Caratterizzazione dell'account | Rappresentare UTA/unified senza saldi Futures fittizi |
| ADR-EVT-05 | Bridge legacy `String[46]` | Proiezione e legami persistenti, backup, editing, invalidazione, migrazione senza distruzione |
| ADR-EVT-06 | Requisiti dei record composti | Distingue delta osservato netto, componenti, fee separate, rettifiche |
| ADR-EVT-07 | Regole asset settlement e fiscale | Integrazione EMT, LIFO unico, policy legalmente validata |
| ADR-EVT-08 | Performance / storage | Granularità raw vs aggregazione reversibile, benchmark su funding massivi |
| ADR-EVT-09 | Eventi storici già importati | Recovery da `DocumentiFonte`, handling per missing raw |
| ADR-EVT-10 | PRD canonico | Nel branch sono presenti due file "PRD master": scegliere il documento di riferimento prima di approvare tutti i gate |

### Gate di approvazione di questo contratto

1. Confermare che `recordRole`, `observedDelta`, `economicComponents`, `account`, `instrument` e `sourceRefs` rappresentino tutti i report P1 senza campi impliciti.
2. Confermare su fixture reali **anonimizzate** le semantiche net/gross e i segni per ogni export/provider dichiarato supportato.
3. Tracciare ogni campo nuovo a una capacità già esistente o a un ADR necessario; non duplicare il motore di saldo/LIFO.
4. Produrre specifica successiva della sintassi JSON, della persistenza e della migrazione, solo dopo questi test concettuali.
5. Mantenere separati gli stati `accounting-ready` e `tax-ready`; nessuna regola fiscale non confermata entra nel kernel generico.
6. Coprire l'intero lifecycle (edit/delete/undo/reimport/backup/restore/report) prima di dichiarare completata la feature.

---

## 13. Fonti tecniche verificate (descrivono export/API, NON la fiscalità italiana)

- [Bybit V5 — Get Transaction Log](https://bybit-exchange.github.io/docs/v5/account/transaction-log): `change = cashFlow + funding - fee`, segni fee/funding e campi ordine/asset.
- [Binance — Get Income History](https://developers.binance.com/docs/derivatives/usds-margined-futures/account/rest-api/Get-Income-History): `incomeType`, `income`, `asset`, `time` e tipi dei redditi.
- [OKX — Get bills details](https://www.okx.com/docs-v5/en/): `billId`, `balChg`, `subType`, `pnl`, `fee`, `ccy`.
- [Bitget — Get Account Bills](https://www.bitget.com/docs/catalog/classic-contract-account/classic-contract-account): `productType` USDT/USDC/COIN futures, `billId`, `amount`, `fee`, `businessType`.
- [Bitget — Position History](https://www.bitget.com/legacy-docs/uta/trade/Get-Position-History): distingue PnL cumulato lordo, net profit, funding e fee; utile come fonte descrittiva, non come posting extra.
- [GiacenzeCrypto — classe Derivati](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Derivati.java), [config Bybit Derivati](../../config/import/Bybit%20Derivati.json), [config Bybit UTA](../../config/import/Bybit%20UTA.json), [config EMT](../../config/varie/EmoneyToken.json) (codice e configurazioni esistenti).

La fiscalità italiana viene determinata **a valle**, in una policy distinta e documentata, secondo normativa e prassi del periodo d'imposta. Questa specifica non afferma che tutti i tipi di evento sopra siano automaticamente c-quater imponibili/deducibili.
