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


### 10.6 Riuso della classificazione manuale già presente — audit verificato

**Non progettare un secondo gestore generalista dei movimenti derivati.** Il flusso UI esistente copre già una parte rilevante del ciclo:

| Punto reale | Comportamento verificato | Implicazione |
|---|---|---|
| [\`GUI_ClassificazioneMovimento\`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/GUI_ClassificazioneMovimento.java) | Classifica depositi/prelievi come reward, acquisto, donazione, trasferimento, commissione/cash-out, operazioni di collaterale; può classificare gruppi compatibili e annullare la classificazione | Estendere le scelte in base alla *natura economica* derivata; non riusare \`DAI\`/reward per un PnL di perpetual |
| \`ClassificaMovimenti(...)\` | Dopo \`PulisciMovimentiAssociati\`, modifica **in place** gli originali (\`[5]\`, \`[17]\`, \`[18]\`, \`[19]\`, \`[20]\`, \`[21]\`, eventuali \`[15]\` e \`[35]\`) | Il vecchio array non conserva una versione fiscale delle scelte per i derivati; la nuova operazione deve produrre uno storico di **decisione semantica** |
| \`RiportaTransazioniASituazioneIniziale(...)\` | Rimuove i movimenti generati \`[22]=AU\`, scioglie gli abbinamenti \`[20]\`, ripristina importi/prezzo temporanei e codici di classificazione | L'annullamento va esteso al nuovo override senza perdere source record o toccare altri eventi correlati |
| [\`GUI_ModificaMovimento\`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/GUI_ModificaMovimento.java) | Consente modifica dati, ma i generati o collegati (\`[20]\` non vuoto) sono limitati a **prezzo e note**, salvo il percorso che scioglie il gruppo | L'editor di componenti derivati non deve aggirare le protezioni dei movimenti collegati |
| [\`MovimentiStorico\`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/MovimentiStorico.java) | Registra snapshot delle modifiche esplicitamente accodate e li scrive solo al salvataggio dei movimenti | \`ClassificaMovimenti\` non chiama direttamente \`AccodaModifica\`; progettare copertura del cambio di classificazione senza assumere che sia già storicizzato |
| [\`Importazioni\`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Importazioni.java) | \`Sovrascrivi esistenti\` annulla associazioni/classificazioni, incrementa \`ClassificazioniAnnullate\` e mostra un avviso; lo storico del movimento può mantenere il lignaggio | L'avviso da solo **non** preserva un override economico; serve confronto, conservazione e scelta esplicita alla sovrascrittura |
| [\`DocumentiFonte\`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/DocumentiFonte.java) | Mantiene documenti originali e relativa identità | Le correzioni non devono mutare il file/API di origine |

**Limite di questa verifica:** si sono ispezionate le routine nominate e i loro contratti, non l'intero wiring di \`Principale.java\` o una sessione GUI. La classificazione economica dei derivati descritta sotto è **da implementare**, non è oggi disponibile nell'editor.

### 10.7 Tre operazioni che NON devono diventare un'unica modifica

| Tipo intervento | Oggetto | Effetto consentito | Invariante |
|---|---|---|---|
| **Classificazione economica** | Tipo di componente: per esempio \`TRADING_FEE\` → \`FUNDING\` | Cambia natura, eventuale trattamento fiscale, report e filtri | Importo, asset e posting **non** cambiano solo per riclassificazione |
| **Correzione contabile** | Quantità, asset, timestamp, account, componenti del netto e matching | Cambia i dati *effettivi* dopo approvazione, con override documentato | Il delta osservato dalla fonte resta consultabile; le somme per asset devono riconciliarsi |
| **Decisione fiscale** | Qualificazione/regime, costo attribuito al settlement, aliquota, ammissibilità onere | Si applica tramite versione di policy e perimetro fiscale, non tramite causale arbitraria nel JSON | Nessuna correzione di importo o categoria fiscale può sanare implicitamente una regola giuridica \`PENDING\` |

Un utente può effettuare queste tre operazioni attraverso **un unico dettaglio UI**, ma devono avere autorizzazioni, validazioni, storico e conseguenze di ricalcolo distinte. La scelta di una categoria manuale \`PNL\` non prova da sola che il contratto sia fiscalmente c-quater.

**Stato attuale del confronto:** la UI ordinaria classifica soprattutto i depositi/prelievi; non conosce la scomposizione canonica \`economicComponents[]\`. L'estensione non è quindi un mero nuovo valore nella combobox: è un adattamento del workflow esistente al nuovo modello.

### 10.8 Viste sorgente, automatica ed effettiva: niente perdita del dato originale

Richiesta funzionale (persistenza da definire in ADR):

1. **\`source\`** — \`SourceRecord\` e valori originari immutabili, inclusi importi, cause e id nativi.
2. **\`automatic\`** — normalizzazione riproducibile da sorgente e versione del parser/JSON.
3. **\`manualOverride\`** — solo i campi modificati, valore precedente, nuovo valore, autore/istante, motivo, riferimenti, stato attivo/annullato e *versione della base su cui è stato proposto*.
4. **\`effective\`** — proiezione deterministica dei dati automatici con override esplicitamente attivi e compatibili; è l'unica che produce il \`postingPlan\` accettato e l'input dei motori.

Non imporre in questa Design Spec una nuova tabella per ognuna delle quattro viste. La scelta DTO, H2 e proiezioni ricostruibili sarà l'ADR. **La fonte di verità patrimoniale rimane unica**; il layer di override non è un secondo ledger.

**Identità:** l'override deve riferirsi a un \`canonicalEventId\` durevole, con collegamento a \`SourceRecordKey\`. Il solo \`v[0]\` del movimento non basta: può cambiare per modifica, traslazione, scambio differito o reimport. \`[42]\` (lignaggio) resta utile per lo storico legacy, ma non sostituisce automaticamente l'identità economica canonica. Se un nuovo parsing divide o unisce eventi, il sistema deve proporre un remapping esplicito e verificato, non applicare l'override a un evento sbagliato.

### 10.9 Contratto di riconciliazione dopo modifica manuale

**Fixture di incoerenza controllata:** una fonte documenta \`+100 USDC PnL -3 USDC funding -2 USDC fee = +95 USDC change\`.

- **Riclassificazione sola:** \`TRADING_FEE → FUNDING\` per la componente di \`-2\`: la quantità netta resta \`+95\`. Reimputare categorie fiscali e mostrare l'avvenuta scelta.
- **Correzione componenti:** modificando il funding da \`-3\` a \`-4\`, le componenti sommano \`+94\`, ma la fonte dichiara \`+95\`. Il sistema **non** deve sottrarre 1 USDC dal saldo né registrare una nuova fee per chiudere automaticamente il disavanzo.
- **Risposta obbligatoria:** stato \`CONFLICT\` / \`PENDING_RESOLUTION\`; confronto \`observed=+95\` vs \`components=+94\`, differenza \`-1\` USDC; impedire l'output fiscale definitivo interessato. L'utente può correggere un'altra componente **con evidenza**, oppure proporre una rettifica del posting con motivazione e fonte di conferma; in entrambi i casi occorre una conferma esplicita e ricalcolo.
- **Inversione/undo:** annullare l'override deve ripristinare il dato \`automatic\` senza reimport obbligatorio, preservando le versioni storiche.

Per fee in una *valuta diversa* il controllo deve essere effettuato per asset; non richiedere una somma aritmetica tra USDT e BTC. Per una \`completeness=PARTIAL\` non dare per scontato che la differenza sia un errore: segnalare *scomposizione incompleta*, evitando di inventare redditi.

### 10.10 Reimportazione, sovrascrittura e annullamento: workflow comune

Il percorso attuale \`Sovrascrivi esistenti\` può sciogliere classificazioni manuali; per i **nuovi eventi derivati** la condotta richiesta è:

1. **Preview non distruttiva:** confrontare nuovo parsing/fonte e versione base con \`effective\` esistente, mostrando le decisioni utente attive e le differenze.
2. **Stessa semantica, stessa identità:** l'override manuale può rimanere attivo e deve essere verificato; nessun doppio posting, nessuna riscrittura arbitraria della cronologia.
3. **Dati sorgente modificati o ristrutturazione di eventi:** non riapplicare automaticamente override incompatibili; marcare \`CONFLICT\` e richiedere \`mantieni override / accetta nuovo parsing / modifica manualmente / rimanda\` secondo condizioni documentate.
4. **Utente annulla il reimport:** nessuna mutazione di saldo, override, legami o storico.
5. **Conferma e salvataggio:** aggiornare unitariamente gli stati necessari in coerenza col ciclo di salvataggio esistente; rollback di tutti gli effetti in caso di errore; il dettaglio conserva prima/dopo e relative fonti.
6. **Cancellazione documento:** rivalutare se altri documenti supportano gli eventi; non cancellare senza conferma una decisione utente o lasciare proiezioni fiscali orfane. Un override senza più una fonte valida richiede uno stato esplicito e l'esclusione dai report definitivi finché non risolto.
7. **Dichiarazione già definitiva:** mantenere inalterato lo snapshot archiviato, mostrare la differenza rispetto al ricalcolo corrente.

**Non assumere che il normale avviso \`ClassificazioniAnnullate\` soddisfi questi requisiti:** documenta una perdita di classificazione già avvenuta, non costituisce conservazione dell'override.

### 10.11 Validazione e invalidazione delle dipendenze

Distinguere \`sourceRevision\`, \`parserRevision\`, \`manualRevision\` e \`fiscalPolicyRevision\`, oppure un meccanismo semanticamente equivalente (struttura concreta da decidere). Il ricalcolo deve invalidare in modo causale:

| Modifica | Account/saldo | Lotti/costo | Componenti derivati | Report T/RT e W/RW |
|---|---|---|---|---|
| Solo tipo \`FEE → FUNDING\`, stessa valuta/quantità | **Invariato** | Invariato salvo impatto fiscale provato | **Ricalcolare** classificazione | Rifare report interessati |
| Cambia importo, asset, data, account | **Ricalcolare** | **Ricalcolare** | **Riconciliare** | Rifare anni interessati |
| Cambio manuale del prezzo EUR, quantità invariata | Invariato | Ricalcolare se determina costo fiscale | **Rivalutare** | Rifare totali fiscali interessati |
| Modifica di regola fiscale senza cambio quantità | Invariato | Ricalcolare se impatta lotti | **Ricalcolare** proiezione fiscale | Rifare i prospetti interessati |
| Rimozione fonte o modifica matching | Verificare posting accettati | Verificare lotti dipendenti | Rivalutare autorità/conflitti | Report potenzialmente bloccati |

Non introdurre un ricalcolo totale *per definizione*: usare i checkpoint e le impronte dove è corretto, ma garantire che lo stato completo e quello incrementale siano **equivalenti**. In particolare, quando il nuovo kernel leggerà \`[44]\` e gli override, le impronte/epoche dovranno riconoscerne la modifica; oggi il marcatore 44 è intenzionalmente fuori dall'impronta del kernel crypto.


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
| EVT-21 | +95 USDC, componenti +100/−3/−2; riclassifica sola FEE → FUNDING | Delta patrimoniale +95 invariato, classificazione e report aggiornati, scelta storicizzata |
| EVT-22 | +95 osservato e componente funding corretta −3 → −4 | Componenti +94, differenza −1 esplicita, nessun posting aggiuntivo, report fiscale bloccato |
| EVT-23 | Correzione dei componenti con riconciliazione dell'altra voce documentata | Stato da CONFLICT a riconciliato senza cambiare dato raw; versioni consultabili |
| EVT-24 | Annulla classificazione derivati e collega/scollega movimento generato AU | Identità evento/fonte preservata, niente movimenti o componenti orfani |
| EVT-25 | Reimport identico con override manuale attivo | Override conservato e verificato, stessi posting e report, nessun doppione |
| EVT-26 | Reimport cambia PnL, timestamp o scompone eventi: override non più compatibile | Preview CONFLICT, nessuna perdita silenziosa o remapping a evento sbagliato |
| EVT-27 | Modifica quantità di evento collegato o generato | UI impedisce bypass dei vincoli, propone percorso sicuro; undo ripristina tutti i legami |
| EVT-28 | Salva/annulla correzione; backup, restore e report definitivo preesistente | Commit coerente; backup mantiene decisioni, ripristino identico; snapshot storico invariato |

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
| ADR-EVT-10 | PRD canonico | **Individuato**: [PRD di riferimento](../product/2026-11-02-derivatives-integration-prd.md); la [prima bozza](../product/2026-11-02-derivatives-integration-master-prd.md) resta storica/non normativa per i nuovi requisiti |
| ADR-EVT-11 | Editor e decisioni manuali dei derivati | Riusare GUI di classificazione/modifica e storico esistenti; stabilire copertura delle versioni della classificazione (oggi non garantita da `ClassificaMovimenti`) |
| ADR-EVT-12 | Persistenza override e commit/undo | Legame tramite `canonicalEventId`, modifiche semantiche separate da importi/policy; rollback e backup atomici |
| ADR-EVT-13 | Reimport con override e matching modificati | Precedenza/manual conflict review, mai azzeramento silenzioso o reimport che crea due redditi |

### Gate di approvazione di questo contratto

1. Confermare che `recordRole`, `observedDelta`, `economicComponents`, `account`, `instrument` e `sourceRefs` rappresentino tutti i report P1 senza campi impliciti.
2. Confermare su fixture reali **anonimizzate** le semantiche net/gross e i segni per ogni export/provider dichiarato supportato.
3. Tracciare ogni campo nuovo a una capacità già esistente o a un ADR necessario; non duplicare il motore di saldo/LIFO.
4. Produrre specifica successiva della sintassi JSON, della persistenza e della migrazione, solo dopo questi test concettuali.
5. Mantenere separati gli stati `accounting-ready` e `tax-ready`; nessuna regola fiscale non confermata entra nel kernel generico.
6. Coprire l'intero lifecycle (edit/delete/undo/reimport/backup/restore/report) prima di dichiarare completata la feature.
7. Verificare con EVT-21…28 che la UI esistente supporti override semantici e quantitativi senza introdurre un secondo sistema di classificazione o perdere correzioni al reimport.

---

## 13. Fonti tecniche verificate (descrivono export/API, NON la fiscalità italiana)

- [Bybit V5 — Get Transaction Log](https://bybit-exchange.github.io/docs/v5/account/transaction-log): `change = cashFlow + funding - fee`, segni fee/funding e campi ordine/asset.
- [Binance — Get Income History](https://developers.binance.com/docs/derivatives/usds-margined-futures/account/rest-api/Get-Income-History): `incomeType`, `income`, `asset`, `time` e tipi dei redditi.
- [OKX — Get bills details](https://www.okx.com/docs-v5/en/): `billId`, `balChg`, `subType`, `pnl`, `fee`, `ccy`.
- [Bitget — Get Account Bills](https://www.bitget.com/docs/catalog/classic-contract-account/classic-contract-account): `productType` USDT/USDC/COIN futures, `billId`, `amount`, `fee`, `businessType`.
- [Bitget — Position History](https://www.bitget.com/legacy-docs/uta/trade/Get-Position-History): distingue PnL cumulato lordo, net profit, funding e fee; utile come fonte descrittiva, non come posting extra.
- [GiacenzeCrypto — classe Derivati](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Derivati.java), [config Bybit Derivati](../../config/import/Bybit%20Derivati.json), [config Bybit UTA](../../config/import/Bybit%20UTA.json), [config EMT](../../config/varie/EmoneyToken.json) (codice e configurazioni esistenti).

La fiscalità italiana viene determinata **a valle**, in una policy distinta e documentata, secondo normativa e prassi del periodo d'imposta. Questa specifica non afferma che tutti i tipi di evento sopra siano automaticamente c-quater imponibili/deducibili.
