# PRD master — Gestione nativa dei derivati crypto in GiacenzeCrypto

> **Bozza precedente conservata per tracciabilità, non documento di riferimento.** Il PRD master da usare per la progettazione e per gli ID dei requisiti è [`2026-11-02-derivatives-integration-prd.md`](2026-11-02-derivatives-integration-prd.md). In particolare, la gestione della classificazione manuale, delle componenti economiche, degli override, dello storico, della reimportazione e della compatibilità con `GUI_ClassificazioneMovimento` è specificata nei requisiti `REQ-EDIT-08…16`, in `AC-15…19` e nella [Design Spec canonica §10.6–10.11](../architecture/2026-11-02-derivatives-canonical-event-contract-design.md). Non introdurre nuovi requisiti in questa bozza per evitare due fonti di verità.

**Data:** 2026-11-02  
**Stato:** DRAFT — requisiti di prodotto da approvare; **non** costituisce una specifica tecnica o una validazione fiscale.  
**Repository:** `dimkadiver-cpu/GiacenzeCrypto`  
**Branch:** `feature/derivatives-integration-design`  
**Documento sorgente:** [Audit architetturale](../architecture/2026-11-02-derivatives-end-to-end-audit.md)  
**Destinazione:** PRD master → decisioni normative e ADR → System Architecture Spec → specifiche tecniche → piani di implementazione → verifica di rilascio.

> **Principio guida:** una funzione sui derivati è completa soltanto quando ogni prodotto *dichiarato supportato* funziona, senza ambiguità o duplicazioni, attraverso importazione, patrimonio, contabilità, fiscalità, interfaccia, modifiche, salvataggio, backup, report e ripristino. Il documento non promette supporto indiscriminato a tutti i prodotti derivati esistenti.

---

## 1. Contesto, problema, risultato atteso

### 1.1 Stato del prodotto

GiacenzeCrypto gestisce un registro unico di movimenti crypto, costo fiscale/LIFO, wallet e gruppi fiscali, report W/RW e T/RT, prezzi, documenti origine, modifiche e backup. I derivati sono già **marcati**, ma **non gestiti fiscalmente**: `Derivati.CAMPO=44` identifica `PNL`, `FUNDING`, `COMMISSIONE` ecc.; il calcolo fiscale crypto esistente non interpreta ancora questi marcatori. Conseguenza: il risultato derivato e il regolamento in crypto possono essere classificati in modo improprio nei report e nei lotti.

Il codice contiene 46 campi per movimento (`Importazioni.ColonneTabella=46`): 41=documento di origine, 42=storico/lignaggio, 43=commissioni collegate, 44=tipo derivato, 45=identità operazione. Il modello `v[3]` (exchange/gruppo fiscale) e `v[4]` (sottowallet) è già distinto. Non riservare o sovrascrivere tali campi.

### 1.2 Problema dell'utente

Un contribuente con attività Spot e Futures sullo stesso exchange deve poter:
- ricostruire **il patrimonio reale**, anche se sposta la stessa crypto fra Fund, Spot e Futures;
- distinguere **saldo, margine, sottostante nozionale, PnL latente e differenziali regolati**;
- importare diverse fonti (Transaction History e Order History) senza duplicare PnL/fee;
- determinare i risultati dei derivati in EUR secondo una policy fiscale validata e **separata** dalle plusvalenze Spot;
- conservare i lotti/costi fiscali delle crypto di regolamento, senza duplicare redditi;
- correggere dati, rifare i calcoli, ripristinare archivi e riprodurre una dichiarazione.

### 1.3 Outcome osservabili

**O1 — Quadratura:** i saldi degli asset per account si riconciliano con gli estratti e il totale patrimonio non cambia per un giroconto senza fee.

**O2 — Unicità:** lo stesso evento economico presente in più export genera un solo effetto patrimoniale e un solo effetto fiscale.

**O3 — Separazione:** tutti i redditi derivati riconosciuti sono esclusi dalle somme crypto c-sexies) e calcolati in un prospetto c-quater) distinto, **soltanto quando la policy è confermata**.

**O4 — Conservazione:** i lotti delle crypto ricevute/usate per regolamenti sono coerenti con la relativa policy, il saldo e gli anni successivi.

**O5 — Operatività completa:** modifiche, cancellazioni, reimport, UI, filtri, report, documenti, backup e ricalcoli danno lo stesso stato valido e riproducibile.

**O6 — Retrocompatibilità:** un archivio senza derivati produce gli stessi risultati pre-esistenti, salvo modifiche autorizzate e documentate.

**O7 — Trasparenza:** nessun calcolo fiscale viene presentato come definitivo in presenza di eventi ambigui, documenti incompleti, cambi mancanti o policy non confermate.

---

## 2. Attori e principali casi d'uso

- **Contribuente:** importa documenti, consulta il saldo di ogni account, distingue risultato Futures da Spot, vede i problemi e ottiene il rendiconto fiscale.
- **Revisore/consulente:** esamina riconciliazioni, scarti, base normativa, fonti di cambio, rettifiche e report riproducibili.
- **Manutentore:** aggiunge un importer senza cambiare le regole fiscali, introduce un nuovo prodotto sotto feature flag e verifica test/golden master.

| ID caso | Scenario | Esito richiesto |
|---|---|---|
| UC-01 | Trasferire USDT da Fund a Futures e viceversa | Due saldi account aggiornati, stessa proprietà e costo fiscale; zero reddito per il semplice giroconto |
| UC-02 | Comprare BTC Spot e operare su BTC-USDT perpetual | L'acquisto BTC resta Spot; il nominale del perpetual non crea BTC in giacenza |
| UC-03 | Aprire/chiudere posizioni con commissioni e PnL | Ogni settlement e fee è registrato **una volta**; PnL tecnico e giacenza quadrano |
| UC-04 | Ricevere/pagare funding | Variazione asset registrata, voce derivata classificata e valorizzata |
| UC-05 | Subire una liquidazione | Risultato negativo e fee pertinenti; nessun raddoppio con l'ordine di chiusura |
| UC-06 | Importare due storici dello stesso exchange | Riconciliazione e stato di abbinamento spiegabili, senza duplicazioni |
| UC-07 | Avere un PnL solo latente al 31 dicembre | Nessuna creazione artificiale di cash flow; rappresentazione separata se disponibile |
| UC-08 | Correggere una fee o un cambio e salvare | Ricostruzione coerente delle proiezioni dipendenti e storico della correzione |
| UC-09 | Cancellare/reimportare documento o ripristinare backup | Nessun evento orfano, nessuna doppia importazione, stessa fotografia fiscale |
| UC-10 | Preparare dichiarazione annuale | Prospetti distinti, stato di validazione, provenienza e traccia dei parametri usati |

---

## 3. Ambito di prodotto

### 3.1 Target iniziale proposto (decisione di prodotto ancora da formalizzare)

**P1 candidata:** Futures/perpetual **lineari e con settlement effettivo in USDT o USDC**, exchange centralizzati, storico esportabile di PnL realizzato, funding, commissioni, liquidazioni e trasferimenti. BingX funge da fixture di riferimento; il dominio e il modello devono restare **exchange-agnostic**, con supporto successivo a provider per i quali i tracciati siano verificabili.

Devono essere coperte tutte le operazioni e i flussi di lifecycle della P1, non solo l'import o il report.

### 3.2 Capacità previste nell'architettura, non necessariamente nella prima release

Contratti inversi/coin-margined, margine in multi-asset, settlement fisico, scadenze con consegna, opzioni, cross/portfolio margin, eventi di conversione automatica del collaterale e bonus con restrizioni. L'architettura deve consentirne l'estensione, **ma il software non li deve interpretare con una logica lineare USDT per approssimazione**.

### 3.3 Non-obiettivi

- Esecuzione di ordini, gestione del rischio trading, segnali o trading bot.
- Ricostruzione in tempo reale di tutto l'order book o mark-to-market se la fonte fornisce soltanto eventi di saldo.
- Ricalcolo automatico del PnL dell'exchange da OHLCV quando esiste un differenziale affidabile e riconciliato.
- Secondo database patrimoniale indipendente o secondo LIFO fiscale concorrente.
- Modifica retroattiva di tutti gli archivi senza migrazione esplicita, backup e consenso dell'utente.
- Produzione di dichiarazioni fiscali definitive su una regola giuridica non validata.

---

## 4. Requisiti invarianti e confini di responsabilità

| ID | Requisito non negoziabile | Verifica |
|---|---|---|
| INV-01 | Un evento economico ammesso genera **una** variazione patrimoniale canonica; visualizzazioni/proiezioni non ne duplicano il saldo | Confronto per ID evento e saldo |
| INV-02 | Un trasferimento fra account dello stesso titolare, senza oneri, è a somma quantitativa zero e non produce PnL derivato | Test di proprietà per asset/account |
| INV-03 | Nominale, leva e margine vincolato non sono acquisti/vendite del sottostante | Fixture UC-02 |
| INV-04 | Il risultato dei derivati e quello Spot non si sommano come un'unica categoria fiscale | Prospetti e test di esclusione |
| INV-05 | Un risultato pagato in crypto ha una variazione di giacenza tracciabile e una regola di lotto identificata | Test round-trip costo/quantità |
| INV-06 | Un evento non supportato o ambiguo **non è interpretato in silenzio**; si arresta la finalizzazione fiscale coinvolta | Report di blocco |
| INV-07 | Ogni evento è ricostruibile da fonti, mapping, correzioni e criterio di valorizzazione | Audit trail ripercorribile |
| INV-08 | Cambiare data, classificazione, importo, prezzo o policy invalida tutte le dipendenze, anche di anni successivi | Ricalcolo completo = incrementale |
| INV-09 | Un reimport della stessa fonte e un import di fonti sovrapposte sono idempotenti | Ripetizione N volte |
| INV-10 | Dati e risultati precedenti senza derivati non subiscono regressioni non approvate | Golden master |
| INV-11 | Valori monetari/quantità usano decimali esatti e timestamp con fuso sorgente noto | Test precisione/DST |
| INV-12 | Una dichiarazione definitiva conserva la fotografia degli input e può essere riprodotta o verificata | Backup + report manifest |

**Confini:** importatore = interpretazione tecnica della fonte; registro patrimoniale = proprietà, asset e movimenti; dominio derivati = natura economica di differenziali/oneri; fiscal policy = regole per anno/strumento/giurisdizione; presentazione = vista derivata. Non inserire regole fiscali italiane dentro i JSON di un singolo exchange.

---

## 5. Modello funzionale degli oggetti (non schema DB)

Il PRD richiede le seguenti informazioni concettuali; nomi, tabelle e classi verranno decisi dalla System Architecture Spec.

| Concetto | Dati richiesti | Vincolo |
|---|---|---|
| **SourceRecord** | documento/fonte, provider, versione parser, identificativo originale, riga, timestamp originale e fuso, hash | Conservare evidenza, senza modificare il sorgente |
| **EconomicEvent** | ID canonico stabile, tipo evento, account, istante UTC, asset, quantità firmata, correlazioni, stato | Identità indipendente dalla posizione del record in un CSV |
| **Posting** | account origine/destinazione, asset, variazione, motivo, riferimento evento | Somma conservativa per giroconti; eventi reali non duplicati |
| **DerivativeAttribute** | contratto/strumento, famiglia, settlement asset, side/ordine/posizione se disponibili | Assenza metadati ≠ invenzione dei dati |
| **FiscalComponent** | tipo di reddito/onere, importo EUR, status, base normativa/policy, fonte prezzo, periodo | Separato dal posting patrimoniale |
| **LotEffect** | asset, quantità, costo imputato/consumato, ID lotto, regime, policy applicata | Coerenza con storico e giacenza |
| **MatchEvidence** | record A/B, attributi di confronto, esito, motivazione, override, user action | Ambiguità non risolta non diventa matching automatico |
| **PolicyVersion** | regola/periodo, fonte, stato di approvazione, effetto sul ricalcolo | Tracciabilità temporale |
| **TaxSnapshot** | anno, totale per comparto, dettaglio, saldi, warnings, riferimenti e hash degli input | Riproducibile e non ambiguo |

Gli stati di validazione minimi sono: `VALID`, `INCOMPLETE`, `AMBIGUOUS`, `UNSUPPORTED`, `UNPRICED`, `POLICY_PENDING`, `OVERRIDDEN`. Sono **stati di dominio richiesti**, non nuovi codici già esistenti.

---

## 6. Requisiti funzionali (con ID tracciabili)

### 6.1 Importazioni, fonti e deduplica

| ID | Requisito | Priorità | Accettazione |
|---|---|---|---|
| ING-01 | Importare storici delle variazioni saldo e storici ordini tramite gli importatori esistenti o adattatori espliciti | P0 | I dati tecnici rimangono attribuiti alla fonte |
| ING-02 | Preservare record originali, documento, ID exchange, timezone, precisione e parser/versione | P0 | Dall'evento si risale al record |
| ING-03 | Riconciliare più documenti dello stesso provider senza contare due volte PnL/fee/funding | P0 | Due fonti sovrapposte → un effetto |
| ING-04 | Assegnare un'autorità motivata per attributo (saldo, ordine, PnL, liquidazione, fee) e **non** un unico file preferito per tutto | P0 | La provenienza di ciascun valore è leggibile |
| ING-05 | Non effettuare match incerti usando solo timestamp o importo; mostrare `AMBIGUOUS` | P0 | Collisioni intenzionali non producono abbinamenti falsi |
| ING-06 | Prevedere import incrementale, reimport, sovrascrittura, dati arrivati fuori ordine e righe tardive | P0 | Risultati equivalenti qualunque sia l'ordine delle fonti |
| ING-07 | Prevedere trattamento esplicito per righe non supportate e ordini annullati/non eseguiti | P0 | Nessun movimento economico inventato |
| ING-08 | Riutilizzare prova configurazione e mostrare *prima di scrivere* saldi, nuovi eventi, duplicati, conflitti e scarti | P1 | Simulazione non modifica archivio |
| ING-09 | Conservare il collegamento tra evento canonico e tutte le fonti che lo documentano | P0 | Audit e cancellazione fonte sicuri |

### 6.2 Account, patrimoni, trasferimenti, margini

| ID | Requisito | Priorità | Accettazione |
|---|---|---|---|
| ACC-01 | Distinguere identità exchange, account operativo (Fund/Spot/Futures), wallet fiscale e contribuente | P0 | Nessuna divisione automatica dei lotti per semplice sotto-account |
| ACC-02 | Registrare il giroconto con uscita+entrata collegate e saldo totale invariato, fee separata se presente | P0 | Invariante INV-02 |
| ACC-03 | Abbinare trasferimenti quando le due fonti descrivono lati opposti; non creare gambe aggiuntive già presenti | P0 | Quadratura per singolo account |
| ACC-04 | Calcolare saldo aggregato e per sotto-account a ogni istante utile, senza sommare il valore nozionale | P0 | Conto Futures coerente con fonte |
| ACC-05 | Distinguere saldo disponibile, margine vincolato e PnL latente **solo se la fonte lo consente**, senza trattarli come transazioni fiscali | P1 | Voci non sommate due volte al patrimonio |
| ACC-06 | Supportare account creati/rinominati/aggregati senza cambiare artificialmente identità economica o gruppo fiscale | P1 | Ricalcolo coerente dopo mapping account |

### 6.3 Eventi derivati

| ID | Requisito | Priorità | Accettazione |
|---|---|---|---|
| DER-01 | Classificare e distinguere `PNL_REALIZED`, `FUNDING`, `TRADING_FEE`, `LIQUIDATION`, `FEE_REBATE`, `BONUS`, `SETTLEMENT`, `POSITION_META`, `TRANSFER` | P0 | Tipi conservati senza mapping improprio |
| DER-02 | Trattare apertura/chiusura come metadati di posizione ove presenti; il nominale non è una variazione di asset | P0 | Nessuna falsa giacenza BTC |
| DER-03 | Riconoscere chiusure parziali e differenziali effettivamente regolati; non equiparare latente e realizzato | P0 | Due partial close danno i soli risultati regolati |
| DER-04 | Preservare segno e valuta effettiva del PnL/funding/fee, compresi importi piccolissimi | P0 | Quadratura con saldo provider |
| DER-05 | Registrare fee su aperture anche con PnL nullo, senza generare falsi depositi/prelievi | P0 | Evento economico limitato alla fee |
| DER-06 | Verificare se il PnL di una fonte è lordo o netto di funding/fee, esponendo l'incertezza | P0 | Nessun doppio onere |
| DER-07 | Gestire liquidazioni, rebate, bonus e anomalie con classificazioni non equivalenti a un normale profitto | P0 | Stato fiscale specifico o `POLICY_PENDING` |
| DER-08 | Marcare prodotti non coperti anziché applicare il modello dei perpetual lineari per difetto | P0 | `UNSUPPORTED` blocca report fiscale |

### 6.4 Regolamento in crypto e lotti fiscali

| ID | Requisito | Priorità | Accettazione |
|---|---|---|---|
| SET-01 | Per ogni accredito/addebito di settlement, mantenere **un solo posting patrimoniale** | P0 | Saldo derivante da un evento unico |
| SET-02 | Calcolare l'effetto sui lotti dell'asset ricevuto/usato secondo una policy **validata e versionata** | P0 | Lotto riconciliato e motivato |
| SET-03 | Non assegnare automaticamente costo zero a un profitto da derivato per mancanza di classificazione | P0 | `POLICY_PENDING` se base non definita |
| SET-04 | Aggiornare quantità e costo di lotti consumati da perdite/fee; tenere distinto l'eventuale secondo effetto fiscale sulle crypto | P0 | Una base non viene tassata due volte |
| SET-05 | Coordinare le tre vie esistenti di LIFO/costo (`Calcoli_PlusvalenzeNew`, `Calcoli_RT`, `Principale_GiacenzeaData`) | P0 | Stessi lotti su tutte le viste |
| SET-06 | Conservare il costo di carico nei trasferimenti interni, anche quando sono usate pile fiscali per gruppo | P0 | UC-01 invariato prima e dopo |

### 6.5 Motore fiscale dei derivati

| ID | Requisito | Priorità | Accettazione |
|---|---|---|---|
| TAX-01 | Produrre risultati annuali c-quater) separati da c-sexies), secondo regime e validità della regola | P0 | I due comparti non si compensano impropriamente |
| TAX-02 | Valorizzare ogni componente nella valuta fiscale tramite tasso storico, timestamp pertinente, fonte e override tracciato | P0 | Riconciliazione evento→EUR, niente tasso fisso 1:1 |
| TAX-03 | Gestire oneri pertinenti, differenziali positivi/negativi, effetti di netting delle fonti, eventuali rettifiche | P0 | Totale riconciliato per anno |
| TAX-04 | Tracciare perdite pregresse e possibilità di utilizzo soltanto in comparti/anni ammessi dalla policy normativa approvata | P0 | Report annuali coerenti anche dopo ricalcolo precedente |
| TAX-05 | Marcare esiti fiscali sospesi fino a conferma di instrument, timing, funding, asset settlement, monitoraggio estero | P0 | Nessun prospetto finale non validato |
| TAX-06 | Ricalcolare cambiando la policy fiscale, conservando versione precedente e spiegazione delle differenze | P1 | Confronto prima/dopo policy |
| TAX-07 | Mantenere separata la disciplina di monitoraggio W/RW e eventuali imposte patrimoniali sul rapporto/asset | P0 | Nessun nominale/margine sommato per automatismo |

**Nota:** aliquota, sezioni dichiarative, compensabilità e trattamento delle stablecoin NON devono essere hardcoded in questo PRD; l'implementazione deve seguire un registro delle regole normative per periodo con fonti verificabili.

### 6.6 Interfaccia e flussi utente

| ID | Requisito | Priorità | Accettazione |
|---|---|---|---|
| UX-01 | Vista derivati collegata ai movimenti esistenti con tipo, account, contratto se noto, stato, EUR e documento | P0 | Dallo stesso evento si naviga al movimento e al documento |
| UX-02 | Filtro per exchange, account, anno, tipo evento, stato e provenienza, senza alterare i risultati fiscali filtrando la vista | P0 | Filtro solo presentazionale |
| UX-03 | Dashboard di riconciliazione: quadratura per asset/account, match, duplicati, scarti, prezzi mancanti | P0 | Differenze spiegabili e azionabili |
| UX-04 | Rettifica manuale/override con motivo e fonte obbligatori per campi fiscali critici; conservare il record sorgente | P0 | Storico e rollback disponibili |
| UX-05 | Eliminare, annullare, separare, unire, traslare, cambiare wallet/account senza lasciare eventi e lotti orfani | P0 | Ricalcolo e controllo integrità |
| UX-06 | Mostrare avvisi contestuali e bloccare la conferma fiscale per `UNSUPPORTED/AMBIGUOUS/POLICY_PENDING/UNPRICED` pertinenti | P0 | Lo stato non è nascosto in un log |
| UX-07 | Il riepilogo di ogni anno collega i due comparti ma non ne somma i redditi | P1 | Prospetti distinti |
| UX-08 | Estendere, senza duplicare inutilmente, filtri, GUI dettagli, modifiche e prove importazione esistenti | P1 | Esperienza coerente con UI corrente |

### 6.7 Persistenza, export, backup e lifecycle

| ID | Requisito | Priorità | Accettazione |
|---|---|---|---|
| SYS-01 | Persistenza atomica/coerente fra evento, collegamenti, versioni, overrides e movimenti | P0 | Nessuna scrittura parziale silenziosa |
| SYS-02 | Migrazione idempotente e retrocompatibile per archivi precedenti, senza alterare i sorgenti | P0 | Upgrade ripetibile |
| SYS-03 | Backup/ripristino di tutti i dati derivati, fonti, policy, prezzi, audit trail e report; aggiornare manifest/compatibilità | P0 | Restore e confronto riproducibili |
| SYS-04 | Cancellazione documento/evento, undo e reimport non lasciano record derivati orfani | P0 | Invarianti database e registry |
| SYS-05 | L'importazione di un archivio senza derivati non richiede riclassificazione e non cambia il risultato storico | P0 | Golden master |
| SYS-06 | Ricalcolo incrementale e completo equivalenti; invalidazione su data, ammontare, account, link, policy e cambio | P0 | Snapshot identici |
| SYS-07 | Report e export contengono importi, categoria, metodologia, riferimenti, warnings, eventuale policy e confronto di rettifiche | P0 | Un revisore ricostruisce ogni totale |
| SYS-08 | Dati sensibili (API key, percorsi locali, note identificative) non entrano per errore in report e diagnostica condivisibili | P0 | Test di redazione |
| SYS-09 | Nessun aumento di complessità/performance non misurato; confronti prima/dopo su suite rappresentative | P1 | Budget prestazioni confermato prima del rilascio |

---

## 7. Semantica contabile — scenari di accettazione

In tutti gli esempi sotto, soltanto per semplicità, **1 USDT = 1 EUR**, costo storico degli USDT uguale al valore al momento dell'operazione e fee già incluse dove indicato. **Sono test contabili**, non l'implementazione delle regole fiscali controverse.

| Test | Situazione | Saldo per account e patrimonio | Reddito derivato tecnico | Controllo aggiuntivo |
|---|---|---|---|---|
| SC-01 | 2.000 USDT Fund; 1.000 a Spot; acquisto 1 BTC; 1.000 a Futures | Spot 1 BTC, Futures 1.000 USDT; Fund 0 | 0 | Nessun nominale Futures in saldo |
| SC-02 | SC-01 + Futures netti -400 | Futures 600 USDT, Spot 1 BTC | -400 EUR | Lotti consumati secondo policy approvata |
| SC-03 | SC-01 + Futures netti +400 | Futures 1.400 USDT, Spot 1 BTC | +400 EUR | Accredito lotto senza doppia tassazione |
| SC-04 | Margine vincolato 500 su Futures con saldo 1.000 | Totale patrimonio USDT invariato; disponibile 500, vincolato 500 *se fonte disponibile* | 0 | Nessuna spesa 500 |
| SC-05 | Chiusura +100 e fee -2, funding +1 | Variazione asset +99 USDT | +99 EUR, **solo se le componenti sono separate e riconosciute** | Se PnL è già netto evitare detrazione ulteriore |
| SC-06 | 2 record fonte A/B per stessa chiusura | Un solo posting | Un solo PnL | Due evidenze collegate |
| SC-07 | Transfer 100 Fund→Futures con due righe già presenti | -100 / +100, totale 0 | 0 | Non creare altre due gambe |
| SC-08 | Posizione aperta con PnL latente -250 e nessun regolamento | Saldo ledger non variato | 0 *salvo altri eventi regolati* | Mostrare latente soltanto informativo |
| SC-09 | Evento reale 31/12 UTC ma 01/01 locale fiscale | Attribuzione anno secondo regola timezone validata | Anno verificabile | Conservare timestamp e fuso originale |
| SC-10 | Reimport, modifica PnL e ripristino backup | Saldi/risultati uguali dopo ricostruzione | Idempotenti | Audit completo |
| SC-11 | Asset settlement USDT con cambio EUR variabile | Quantità USDT indipendente dal valore EUR | Conversione per evento | No conversione 1:1 implicita |
| SC-12 | Mancanza storico apertura, fee o riferimento PnL ambiguo | Saldo parziale evidenziato, non inventato | Blocco o warning esplicito | Nessuna dichiarazione pronta |

Il risultato **tecnico** non è automaticamente un importo dichiarabile. La sua qualificazione fiscale è subordinata alla policy §10.

---

## 8. Matrice obbligatoria di copertura end-to-end

La System Architecture Spec deve trasformare questa matrice in elenco di touchpoint e test. La colonna **completo** non può essere dichiarata se un flusso in-scope non attraversa coerentemente gli stadi applicabili.

| Classe evento/oggetto | Ingest & match | Saldo/account | Lotto/costo | Fisco | UI/modifica | Report/export | Backup/reload |
|---|---|---|---|---|---|---|---|
| Giroconto interno | Sì | Sì | Conservazione | Neutro salvo fee | Sì | Sì | Sì |
| Apertura/chiusura come metadata | Sì | No nominale | Non per sottostante | Solo settlement | Sì | Sì | Sì |
| PnL positivo | Sì | Accredito | Policy lotto entrata | Separato | Sì | Sì | Sì |
| PnL negativo | Sì | Addebito | Policy consumo | Separato | Sì | Sì | Sì |
| Funding +/- | Sì | +/- asset | Policy settlement | Policy derivati | Sì | Sì | Sì |
| Fee e rebate | Sì | +/- asset | Policy costo/consumo | Onere/provento policy | Sì | Sì | Sì |
| Liquidazione | Sì | Solo risultati effettivi | Policy | Separato | Sì | Sì | Sì |
| Margine/latente | Se disponibile | Solo stato, no posting fittizio | Invariato salvo regolamento | Policy | Sì | Informativo | Sì |
| Evento sconosciuto | Conservare/scartare con ragione | Non inventare | Non inventare | Blocco | Diagnostica | Warning | Sì |
| Override e cancellazione | Tracciabile | Ricostruito | Ricostruito | Ricostruito | Undo | Versioni | Sì |

**Ulteriore matrice azioni:** per ciascun evento in-scope verificare creare/importare, visualizzare, filtrare, modificare, classificare, associare/scollegare, eliminare, annullare, rimportare, traslare ora, unire/separare se ammesso, esportare, generare report, salvare, fare backup, ripristinare, ricalcolare dopo modifica anno precedente. Ogni azione deve essere **supportata**, **bloccata con motivazione**, oppure **non applicabile con criterio documentato**.

---

## 9. Requisiti non funzionali e verificabilità

### 9.1 Correttezza e integrità
- L'import non modifica l'archivio se la preview viene annullata.
- Elaborazione deterministica a parità di input, configurazione, fuso, prezzi e versione policy.
- Ogni nuovo effetto ha reference all'evento, e ogni evento alla fonte; le eccezioni sono esplicite.
- Nessun recupero silenzioso di PnL da un evento `UNSUPPORTED`.
- Le operazioni concorrenti, interrotte e fallite non lasciano stato persistente a metà.

### 9.2 Prestazioni
La baseline va misurata nella fase di audit runtime: tempo import, riconciliazione, ricalcolo completo/incrementale, RAM, refresh UI e report, su archivio solo Spot e misto con migliaia di funding. I checkpoint LIFO esistenti (blocchi di 5.000 movimenti) sono un vincolo di regressione da verificare. **Soglie numeriche solo dopo benchmark**, non soglie arbitrarie nel PRD.

### 9.3 Qualità del prodotto
- Errori operabili: ogni differenza di saldo o conflitto ha importo, account, asset, fonte, causa, azioni disponibili.
- Una schermata di risultato deve distinguere `tecnico`, `fiscalmente validato`, `incompleto`.
- Niente hardcode fiscale per provider; UI e diagnostica in italiano, formule e dettagli riproducibili.
- Accessibilità e comportamenti Swing coerenti con controlli esistenti; nessun blocco dell'EDT in import massivi.
- Nessuna esposizione in log/report di segreti, path sensibili o contenuto raw se non necessario.

---

## 10. Registro decisioni normative — gate prima della fiscalizzazione

Questa sezione è **backlog di validazione**, non interpretazione fiscale definitiva. Ogni decisione deve essere corredata da disposizione applicabile, prassi amministrativa, anno, condizioni del prodotto, validatore e test.

| ID | Questione | Rischio se lasciata implicita |
|---|---|---|
| LEG-01 | Classificazione fiscale per tipo contratto (lineare, inverso, physically settled, opzione, CFD, rapporto estero) | Comparto fiscale sbagliato |
| LEG-02 | Momento rilevante per PnL parziale/regolato, liquidazioni, settlement periodici e posizioni aperte | Anno fiscale sbagliato |
| LEG-03 | Funding positivo/negativo: qualificazione, competenza e inclusione negli oneri | Imponibile errato |
| LEG-04 | Commissioni e rebate: oneri ammissibili, PnL netto già riportato, commissioni in asset diversi | Doppio costo / costo non ammesso |
| LEG-05 | Attribuzione del costo fiscale alle crypto ricevute da risultato derivato | Tassazione duplicata negli anni successivi |
| LEG-06 | Consumo di crypto per perdita/fee: eventuale ulteriore evento crypto fiscalmente rilevante | Doppia o mancata tassazione |
| LEG-07 | USDt/USDC: qualificazione della stablecoin, prezzo EUR, cambi e regole sui trasferimenti | Base di calcolo errata |
| LEG-08 | Minusvalenze c-quater, compensazione e riporto in anni successivi; sezioni effettive del modello per anno | Riepilogo dichiarativo errato |
| LEG-09 | Monitoraggio W/RW, IVAFE/imposta sulle crypto, collaterale e rapporti su piattaforme estere | Omissione o doppio monitoraggio |
| LEG-10 | Operazioni composite, bonus e derivati non lineari | Erronea generalizzazione del modello |

Criteri di chiusura: `PROPOSTA` → `EVIDENZIATA` (fonte raccolta) → `VALIDATA` (responsabile/consulente, ambito e decorrenza) → `IMPLEMENTATA` → `TESTATA`. Le regole `PROPOSTA` o `EVIDENZIATA` possono servire a visualizzare simulazioni, **non** generare output dichiarativi definitivi.

Riferimenti da esaminare e citare in modo puntuale nei documenti normativi: TUIR articoli 9, 67 e 68, Circolare Agenzia delle Entrate n. 30/E del 27 ottobre 2023 e istruzioni del modello dichiarativo dell'anno applicabile. Non assumere che esempi di Koinly, Blockpit, CoinTracking o Divly definiscano il trattamento fiscale italiano.

---

## 11. Requisiti di validazione, test e rilascio

### 11.1 Suite funzionali richieste

| Gruppo | Verifiche bloccanti |
|---|---|
| Accounting | Conservazione per transfer, bilanci per account e aggregato, margini non consumati, prezzi e asset esatti |
| Matching | Duplicati multi-report, parziali, riordino degli import, fonte discordante, override |
| Derivati | PnL +/-, funding, fee su apertura con PnL 0, liquidazione, bonus e rebate, settlement incompleto |
| Crypto settlement | Lotti creati/consumati, LIFO condiviso, nuova origine del costo, coerenza anno N/N+1 |
| Motori | `Calcoli_PlusvalenzeNew` = `Calcoli_RT` = giacenze a data per gli effetti sui lotti; RW separato per monitoraggio |
| UI/workflow | Dettaglio, modifica, associazione, cancellazione documento, undo, reimport, filtri, stampa e export |
| Storage | Schema upgrade, backup/restore totale e parziale, persist/reload, checksum e identificativi |
| Regression | Archivio senza derivati: golden master; completo/incrementale equivalenti |
| Performance | Tempi/RAM pre-post in fixture versionate, nessuna regressione non motivata |
| Fiscal rules | Per ogni regola `VALIDATA`: fonte, anno, fixture, test nominale e controesempio |

### 11.2 Riutilizzo test già presenti

Estendere, non duplicare, `DerivatiTest`, `ImportazioneGenericaAvvisoDerivatiTest`, `ImportazioneGenericaBybitTest`, `CommissioniCollegateTest`, `GruppoOperazioneTest`, `CalcoliPlusvalenzeNewEquivalenzaIncrementaleTest`, `CalcoliPlusvalenzeNewGoldenMasterTest`, `Calcoli_RW_GoldenMasterTest`, `Calcoli_RW_GiacenzeTest`, `Principale_Movimenti_SeparaUnisciTest`, `GUI_ModificaMovimentoSoloPrezzoTest`, `MovimentiStoricoTest`, `DocumentiFonteTest`, `ScartiImportTest`, `Backup_RestoreTest`, `Backup_CompatibilitaTest` e `DichiarazioniTest`.

I golden master con dati reali utilizzano materiale locale `nocommit/GoldenMaster`; produrre inoltre fixture anonime versionate in repository per i percorsi principali. **L'analisi statica NON equivale ad aver eseguito i test.**

### 11.3 Definition of Done della feature

Il rilascio di un prodotto/anno è ammissibile solo se:
1. scope instrument/provider coperto e dichiarato; altri tipi `UNSUPPORTED`;
2. ogni evento/azione applicabile della matrice §8 è implementato o correttamente bloccato;
3. saldo asset per account + totale, costo fiscale, reddito derivati e anno risultano riconciliati;
4. nessun PnL/fee duplicato fra documenti; prezzo e timezone verificabili;
5. dati vecchi senza derivati invariati, test suite e backup/restore superati;
6. report e prospetti senza policy aperte in ambito;
7. differenze rispetto alla versione precedente, performance e regressioni documentate;
8. audit trail e riproducibilità del report verificati da una seconda esecuzione.

**Non basta** che il nuovo motore fornisca un totale corretto: tutti i percorsi di modifica e il ciclo dei dati devono preservare gli invarianti.

---

## 12. Sequenza documentale e piano di lavoro (non implementation plan)

**G0 — Audit runtime e baseline (prerequisito):** completare l'ispezione degli handler della grande `Principale.java`, delle GUI `.form` e dei punti di salvataggio; build/test Maven, inventario delle tabelle, benchmark; mappa di dipendenza requisito→componenti e matrice impatto UX.

**G1 — PRD master:** approvare questo documento, perimetro della prima release, casi d'uso e definizione di completezza; assegnare le questioni LEG aperte.

**G2 — ADR e System Architecture Spec:** decidere identità del raw record/evento, modello conto, normalizzazione, storage H2/sidecar, policy engine, modalità di ricalcolo, ownership della riconciliazione, proiezioni, semantica transazionale, versioni.

**G3 — Design Spec modulari:** ingestion/reconciliation; accounts/transfers; settlement/LIFO; derivative tax; UI/editing; reporting/backup; validation/performance.

**G4 — Implementation Plan per vertical slice:** slice 1 account/transfer/import/preview; slice 2 PnL/funding/fee+riconciliazione e saldi; slice 3 settlement/lotti e motori; slice 4 report/accounting UI/backup; slice 5 fiscale validato/dichiarazioni. Ogni slice passa test end-to-end e regressioni, non solo test unitari.

**G5 — Release acceptance:** validazione con fixture di BingX opportunamente anonimizzate, più almeno un altro provider/tracciato per escludere dipendenze implicite; riconciliazione, prestazioni, migrazione, backup e confronto report per anni successivi.

> **Vincolo anti-overengineering:** il PRD stabilisce capacità e comportamento; non impone classi, tabelle, microservizi o nuove gerarchie di wallet. In ADR va preferito il minimo cambiamento sicuro ai componenti esistenti. Non creare un secondo registry patrimoniale, un secondo LIFO indipendente o un sistema di eventi parallelo soltanto per comodità di sviluppo.

---

## 13. Questioni ancora da decidere prima della System Architecture Spec

| ID | Decisione necessaria | Indicazione iniziale |
|---|---|---|
| DEC-01 | Definizione esatta della prima release | Linear USDT/USDC perpetual + futures se fonti complete |
| DEC-02 | Autorevolezza delle fonti per ogni tipo evento | Transaction History per posting, Order History per attributi, con controlli/eccezioni |
| DEC-03 | Identità persistente dell'evento canonico | Non usare il solo ID movimento, né hash/timestamp isolato |
| DEC-04 | Sidecar persistente o proiezione ricalcolabile | Decidere dopo audit schema/backup e benchmark |
| DEC-05 | API di accesso comune ai lotti | Evitare quarta copia delle regole |
| DEC-06 | Policy normativo-fiscali e loro validazione | Gate per report dichiarativo |
| DEC-07 | UX derivati integrata o scheda dedicata | Valutare rispetto ai flussi Swing attuali, filtri e modifica |
| DEC-08 | Migrazione vecchi movimenti già importati con campo 44 | Preview diff, ricalcolo deterministico, nessuna conversione distruttiva |
| DEC-09 | Monitoraggio rapporti esteri/collaterale | Regola distinta dal risultato derivati, su base giuridica verificata |
| DEC-10 | Trattamento rettifiche di periodo già dichiarato | Snapshot storico e avviso differenze, mai riscrittura silenziosa |

---

## 14. Limitazioni di questo PRD e criterio di revisione

Il documento è fondato sull'audit statico del branch corrente, **non** su un'esecuzione della codebase o su una validazione fiscale professionale. Tutte le sezioni tecniche indicano ciò che è necessario ottenere, non un'implementazione già approvata. La suite di regressione, le prestazioni e i risultati dei moduli fiscali sono da verificare prima del piano esecutivo.

Aggiornare il PRD mediante requisiti identificati, evidenziando decisione, motivazione, impatto sui test e migrazione. Ogni Design Spec deve riportare gli ID dei requisiti che soddisfa e ogni Implementation Plan i test che li dimostrano.
