# PRD Master — Gestione nativa dei derivati crypto in GiacenzeCrypto

**Versione documento:** 0.1 — bozza di requisiti, non approvata  
**Data:** 2026-11-02  
**Repository:** dimkadiver-cpu/GiacenzeCrypto  
**Branch di progettazione:** feature/derivatives-integration-design  
**Documento di ingresso:** [Audit architetturale 2026-11-02](../architecture/2026-11-02-derivatives-end-to-end-audit.md)  
**Ambito fiscale iniziale:** persona fisica residente in Italia, operazioni fuori dall'attività d'impresa.  
**Baseline di codice esaminata:** fork allineato all'upstream al commit 24c9b4ab3851caf4bceac7caf252bcb18df4f70d; audit statico, senza esecuzione dei test.

> **Stato e limiti.** Questo documento definisce il prodotto richiesto, NON la sua implementazione. Non autorizza a considerare verificato il trattamento fiscale di ogni perpetual o stablecoin. Le questioni normative aperte sono gate espliciti prima di qualsiasi export fiscale dichiarato definitivo. Nessun algoritmo, struttura H2 o modifica dell'array legacy viene imposto senza successiva Design Spec/ADR e test.

---

## 1. Visione e problema

GiacenzeCrypto tiene un archivio di movimenti di cripto-attività, calcola giacenze, lotti e plusvalenze crypto, ricostruisce monitoraggio W/RW e produce report e dichiarazioni. I derivati sono già riconoscibili tramite il marcatore del movimento al campo 44, ma tale marcatore non guida il kernel fiscale. Oggi le variazioni in USDT generate da Futures possono quindi essere interpretate erroneamente come depositi, prelievi, reward o movimenti Spot, alterando sia il costo fiscale sia le plusvalenze.

L'obiettivo è rendere i **Futures e Perpetual regolati in cripto-attività una funzionalità nativa, completa e verificabile**, senza duplicare il patrimonio né introdurre una contabilità scollegata da importazioni, modifiche, report, backup e GUI.

Il problema include i trasferimenti tra Fund/Spot/Futures, l'attribuzione di PnL, funding e commissioni, la riconciliazione di due report che descrivono lo stesso evento, l'effetto dell'asset di regolamento sui lotti crypto, la fiscalità distinta e il mantenimento della correttezza dopo modifiche e ricalcoli.

### 1.1 Risultato per l'utilizzatore

L'utente deve poter importare documenti o dati da provider supportati, vedere su quali account sono allocati gli asset, distinguere margini da movimenti reali, riconciliare gli eventi, correggere eventuali errori, verificare giacenze e costi fiscali, e ottenere un prospetto fiscalmente tracciabile e separato per i derivati **solo quando il relativo profilo normativo è approvato e i dati sono completi**.

### 1.2 Interpretazione di «completo al 100%»

Vuol dire copertura **end-to-end di tutte le operazioni, schermate e workflow applicabili ai prodotti dichiarati supportati**, non supporto immediato a ogni tipo di strumento finanziario. Un prodotto non supportato deve essere identificato senza inventare risultati o mutare silenziosamente il patrimonio.

---

## 2. Gerarchia e regole del documento

- **MUST:** requisito necessario per dichiarare la feature pronta nel perimetro previsto.
- **SHOULD:** requisito richiesto per la qualità di prodotto; eventuale rinvio necessita decisione e motivazione esplicite.
- **GATED:** comportamento che può essere sviluppato come infrastruttura, ma non abilitato come fiscale definitivo prima di approvazione normativa.
- **OUT:** escluso dalla prima copertura; va comunque rilevato e segnalato.
- **AC:** criterio di accettazione misurabile, accompagnato da una fixture o da una procedura di prova.

Identificatori stabili: REQ-DOM, REQ-ACC, REQ-IMP, REQ-REC, REQ-BAL, REQ-SET, REQ-TAX, REQ-UI, REQ-EDIT, REQ-REP, REQ-PER, REQ-QA, NFR e AC. La futura System Architecture Spec e i test devono riportare gli ID, non creare un sistema di requisiti parallelo.

**Responsabilità delle scelte:** questo PRD definisce *che cosa* deve essere vero e osservabile. Persistenza degli eventi, API, struttura delle tabelle, possibilità di un sidecar H2 o di proiezioni derivate, strategia di ricalcolo ed ergonomia dettagliata appartengono alla successiva Design Spec/ADR.

---

## 3. Perimetro del primo rilascio

### 3.1 MUST: perimetro funzionale da coprire per intero

- Futures e perpetual **lineari, regolati in USDT/USDC**, con risultati effettivamente registrati dall'exchange. La copertura reale è dichiarata per provider, prodotto e formato/periodo di export.
- Conti operativi Funding/Fund, Spot, Futures e analoghi, anche sullo stesso exchange.
- Depositi/prelievi/trasferimenti interni; saldo disponibile e margine vincolato se ricavabili da dati supportati, senza considerare nominali o PnL latenti come asset posseduti.
- PnL realizzato positivo/negativo, regolamenti periodici accertati, funding pagato/ricevuto, fee, rimborsi fee, liquidazioni, rettifiche identificabili.
- Import CSV e configurazioni JSON esistenti; integrazione con API quando gli export corrispondenti sono disponibili e verificati. Una sorgente non supportata non deve diventare un import «riuscito».
- Reimport, import incrociati, rettifiche, controlli per account, visualizzazione, filtri, modification history, backup, restore, report e verifiche di regressione.
- Risultato fiscale c-quater e collegamento al costo fiscale degli asset di regolamento **solo per regole approvate e versionate**.

### 3.2 Informazioni sulle posizioni

Open/close, LONG/SHORT, contratto, quantità, esposizione nominale, leverage e PnL latente possono essere conservati come **contesto e riconciliazione**. Non sono automaticamente movimenti patrimoniali o componenti fiscali. Se i dati non consentono di ricostruire una posizione completa, il prodotto deve dichiarare «posizione incompleta» senza impedirne necessariamente la riconciliazione dei settlement.

### 3.3 OUT nella prima release fiscale

Contratti inversi/coin-margined, consegna fisica, opzioni, prodotti strutturati e Dual Investment con regole non validate, portfolio margin multi-collateral, prestiti, operazioni di impresa e derivati OTC privi di tracciati verificati. Potranno entrare dopo una successiva decisione di perimetro e una matrice fiscale dedicata.

Per questi casi sono comunque MUST: riconoscimento come non coperti, tracciabilità dell'origine, assenza di calcoli fiscali fittizi e chiara informazione all'utente.

---

## 4. Personas, obiettivi e casi d'uso

**Utente privato:** possiede Spot e derivati su uno o più exchange; vuole dichiarare gli importi corretti, riconciliare le giacenze, conservare prove e correggere gli errori.

**Utente avanzato:** importa estratti da più provider, con timeframe, fusi e granularità diversi, e deve distinguere gli aggregati finanziari dai movimenti del portafoglio.

**Revisore/professionista:** deve poter ricostruire qualsiasi numero di report fino ai documenti e alla regola fiscale applicata, senza dipendere da schermate o assunzioni nascoste.

| UC | Flusso atteso | Esito |
|---|---|---|
| UC-01 | Importa Fund + Spot + Futures da file separati | Account ricostruiti, trasferimenti riconciliati |
| UC-02 | Importa Transaction History e Order History con PnL sovrapposto | Un solo evento economico e più riferimenti fonte |
| UC-03 | Effettua apertura senza PnL con fee | Solo fee effettivamente addebitata, nessuna vendita del sottostante |
| UC-04 | Chiude in utile/perdita e paga/riceve funding | Saldo asset e risultato derivato distinti e coerenti |
| UC-05 | Controlla saldo iniziale/finale di ciascun sotto-account | Scostamenti rilevati con ragione e stato |
| UC-06 | Corregge data/importo/tipo di un evento o un abbinamento | Tutte le viste interessate si ricalcolano |
| UC-07 | Elimina un documento e annulla/riconferma | Nessun residuo orfano; restore coerente |
| UC-08 | Esamina risultati 2025 e perdita riportabile | Report finanziario separato dai risultati crypto |
| UC-09 | Apre backup su installazione pulita | Identici movimenti, associazioni e risultati |
| UC-10 | Importa derivato non supportato | Blocco fiscale localizzato e diagnosi chiara |
| UC-11 | Una posizione attraversa il 31 dicembre | Soltanto i settlement fiscalmente rilevanti entrano nel periodo prescritto |
| UC-12 | Stesso asset ricevuto in PnL e poi venduto Spot | Reddito derivato riconosciuto una volta; lotti crypto coerenti |

---

## 5. Terminologia di dominio e invarianti

### 5.1 Termini (contratti concettuali, non nomi di classi obbligatori)

- **Raw record:** riga o messaggio di sorgente, con documento, posizione, versione, timezone e contenuto originali.
- **Evento economico canonico:** fatto economico identificato senza doppioni, collegato a uno o più raw record.
- **Posting patrimoniale:** variazione effettiva di quantità per asset e account, oppure vincolo/svincolo di disponibilità, senza duplicare il patrimonio.
- **Settlement:** addebito/accredito derivante da un contratto; non è sinonimo di PnL imponibile senza policy fiscale.
- **Account operativo:** contenitore dell'exchange (Fund, Spot, Futures, Unified…), distinto da gruppo fiscale, wallet on-chain e titolare.
- **Proiezione fiscale:** elaborazione dell'evento in base al regime e all'anno fiscale, con fonte normativa, trattamento, valore EUR, costo fiscale e stato di verifica.
- **Proiezione giacenze/costi:** saldo asset e lotti fiscali; non coincide con il valore nominale delle posizioni aperte.
- **Documento autorevole per componente:** fonte da cui il programma trae un certo importo o attributo, configurata e motivata per provider/prodotto.
- **Stato di riconciliazione:** matched, unmatched, ambiguous, ignored/unsupported, overridden, incomplete oppure reconciled (terminologia definitiva da fissare nelle specifiche).

### 5.2 Invarianti non negoziabili

| INV | Regola | Verifica |
|---|---|---|
| INV-01 | Un evento economico genera il posting patrimoniale una sola volta, indipendentemente dal numero di report | Reimport e multi-source invarianti |
| INV-02 | Un trasferimento interno tra account dello stesso titolare ha somma quantità zero a livello aggregato, salvo fee separata | Fund -q; Futures +q; totale 0 |
| INV-03 | Nominale, leva e collateral non sono implicitamente acquisti/vendite del token sottostante | Open/close nominale invariato nei saldi |
| INV-04 | La classificazione fiscale derivata non trasforma i settlement in reward o plusvalenza c-sexies per difetto | Confronto fra proiezioni |
| INV-05 | Tutti i valori in EUR sono riconducibili a quantità, istante, cambio, fonte e regola | Drill-down a prova |
| INV-06 | Ogni movimento derivato conserva il documento sorgente o dichiara la mancanza | Verifica campo 41/collegamenti |
| INV-07 | Cambiare classificazione, settlement, data o prezzo non lascia cache fiscali stantie | Equivalenza completa/incrementale |
| INV-08 | Un documento cancellato e un override non lasciano eventi/posting fiscali orfani | Integrità dopo annulla/restore |
| INV-09 | I dati senza derivati producono gli stessi esiti fiscali di prima, salvo differenze approvate e documentate | Golden master |
| INV-10 | Gli eventi non supportati o legalmente non qualificati non sono presentati come fiscali definitivi | Stato bloccante |
| INV-11 | Il saldo di un account è ricostruibile dalla somma dei posting rilevanti più saldo iniziale noto, senza usare PnL latente | Quadratura per asset e data |
| INV-12 | Una conversione d'asset di settlement non perde la provenienza del reddito derivato, né crea automaticamente un secondo reddito dello stesso evento | Test lotti e realizzo successivo |
| INV-13 | Le operazioni di editing, export, backup e restore preservano l'identità, la storia e la classificazione dell'evento | Test lifecycle |
| INV-14 | Un risultato fiscale dichiarato definitivo implica copertura del periodo, nessun blocco rilevante e policy validata | Gate report |

---

## 6. Requisiti funzionali

### 6.1 Dominio e responsabilità

| ID | Priorità | Requisito e criterio osservabile |
|---|---|---|
| REQ-DOM-01 | MUST | Distinguere formalmente asset, sotto-account, contratto, posizione, evento economico, movimento di saldo e effetto fiscale. Un valore nominale non deve essere riportato come saldo. |
| REQ-DOM-02 | MUST | Mantenere un'unica consistenza patrimoniale per contribuente: le viste crypto e derivati possono avere calcoli diversi ma non posting concorrenti dello stesso importo. |
| REQ-DOM-03 | MUST | Preservare in archivio la classificazione originale di import, quella normalizzata e le correzioni applicate, in modo tracciabile. |
| REQ-DOM-04 | MUST | Identificare esplicitamente provider, prodotto, valuta di regolamento, periodi di supporto, formato dati e titolarità/account; valori non conoscibili sono «sconosciuti», non ipotizzati. |
| REQ-DOM-05 | MUST | Gestire versioni di regole fiscali dipendenti dall'anno; cambiare una policy deve essere osservabile e determinare ricalcolo controllato. |
| REQ-DOM-06 | MUST | Preservare gli archivi preesistenti senza richiedere reimport obbligatorio o riclassificazione irreversibile. Eventuali migrazioni sono esplicite, ripetibili e verificabili. |

### 6.2 Account, trasferimenti e saldi

| ID | Priorità | Requisito e criterio osservabile |
|---|---|---|
| REQ-ACC-01 | MUST | Distinguere account operativi dello stesso exchange e stesso titolare, includendo almeno Fund/Spot/Futures; i nomi provider-specifici si normalizzano senza alterare gli originali. |
| REQ-ACC-02 | MUST | Ricostruire saldi per (titolare, exchange, account, asset) e il corrispondente patrimonio aggregato; il sotto-account non crea automaticamente un gruppo LIFO indipendente. |
| REQ-ACC-03 | MUST | Riconoscere Fund → Spot, Fund → Futures, Spot ↔ Futures e ritorni come trasferimenti propri, con quantità in uscita e in entrata correlate. |
| REQ-ACC-04 | MUST | Un solo lato disponibile non viene inventato come prova dell'altro: una gamba inferita è marcata come tale, con provenienza e conferma o policy documentata. |
| REQ-ACC-05 | MUST | Trasferimenti interni con fee devono preservare la quota trasferita e contabilizzare la fee realmente addebitata separatamente. |
| REQ-ACC-06 | MUST | Non trasformare un trasferimento fra titolari differenti in trasferimento proprio senza evidenza della titolarità. |
| REQ-ACC-07 | MUST | Margin lock/unlock, collateral disponibile/impegnato e unrealized PnL sono categorie distinte; il saldo contabile di asset non deve variare per il solo notional. |
| REQ-ACC-08 | MUST | Calcolare differenze tra saldi ufficiali, saldo ricostruito e snapshot storici per asset/account e periodo; mostrare copertura e componenti mancanti. |
| REQ-ACC-09 | MUST | Supportare saldo iniziale esplicito e suoi documenti; se la copertura storica è insufficiente, non dichiarare completa la quadratura. |
| REQ-ACC-10 | MUST | Gestire opzione LIFO per gruppo wallet senza perdere il costo fiscale nei trasferimenti; sono richiesti test di uscita e rientro fra gruppi. |

### 6.3 Importazione e normalizzazione

| ID | Priorità | Requisito e criterio osservabile |
|---|---|---|
| REQ-IMP-01 | MUST | Riutilizzare il sistema di import esistente e i documenti di origine; ogni evento normalizzato deve essere tracciabile a file/API e record specifico. |
| REQ-IMP-02 | MUST | Validare tipologia prodotto e copertura temporale **prima** di classificare e contabilizzare. I record sconosciuti restano ispezionabili. |
| REQ-IMP-03 | MUST | Riconoscere esplicitamente segni, moneta, precisione, fee già incluse e lordo/netto del PnL: non è lecito sottrarre due volte fee o funding. |
| REQ-IMP-04 | MUST | Interpretare date sorgente nel fuso dichiarato, normalizzare un istante UTC, conservare orario/fuso originali e usare la regola di competenza approvata per anno fiscale. |
| REQ-IMP-05 | MUST | Separare righe descrittive di ordini/posizioni dai posting di saldo; l'import di un trade non può generare automaticamente compravendita Spot del sottostante. |
| REQ-IMP-06 | MUST | Consentire import multipli, parziali, fuori ordine e sovrapposti; reimport del medesimo contenuto deve essere idempotente. |
| REQ-IMP-07 | MUST | Includere una prova non distruttiva che mostri causali, importi, account, posting proposti, duplicati, scarti e limitazioni prima di confermare. |
| REQ-IMP-08 | MUST | Distinguere errore di parsing, dato assente, operazione non supportata, ambiguità e prezzo non disponibile; evitare successi silenziosi su dataset incompleti. |
| REQ-IMP-09 | MUST | Registrare la versione della configurazione, la sorgente e la revisione del tracciato che hanno prodotto ogni evento. |
| REQ-IMP-10 | SHOULD | Consentire espansione provider tramite configurazioni/adapter senza cambiare il kernel fiscale generale. |
| REQ-IMP-11 | MUST | Non aggregare micro-funding/fee perdendo l'identità dei record prima della riconciliazione; aggregazioni di visualizzazione sono consentite se reversibili. |

### 6.4 Riconciliazione e deduplicazione

| ID | Priorità | Requisito e criterio osservabile |
|---|---|---|
| REQ-REC-01 | MUST | Unificare più descrizioni dello stesso evento economico senza duplicarne saldo o reddito, conservando tutti i riferimenti originali. |
| REQ-REC-02 | MUST | Usare identificatori diretti verificati quando esistono; il solo timestamp o una somiglianza di importo non produce automaticamente un match certo. |
| REQ-REC-03 | MUST | Rendere esplicita la regola di autorità dei singoli campi per provider e report (per esempio saldo/PnL dal Transaction History; contratto/ordine dall'Order History, se verificato). |
| REQ-REC-04 | MUST | Segnalare match ambiguo, differenza di importo, fee discordante, missing side, giorni/fusi disallineati, classificazione contraddittoria e dati fuori periodo. |
| REQ-REC-05 | MUST | Consentire abbinare/scollegare con scelta consapevole, provenienza e storico, senza creare posting doppi. |
| REQ-REC-06 | MUST | Il reimport dopo modifica/cancellazione deve rispettare le scelte dell'utente oppure richiedere una risoluzione esplicita. |
| REQ-REC-07 | MUST | L'esito di una riconciliazione non deve dipendere dall'ordine in cui i file sono stati importati. |
| REQ-REC-08 | MUST | Un match «suggerito» non equivale a uno verificato; report fiscale bloccato per le ambiguità che possono cambiarne l'imponibile. |

### 6.5 Posting dei derivati e settlement

| ID | Priorità | Requisito e criterio osservabile |
|---|---|---|
| REQ-SET-01 | MUST | Supportare almeno PNL_POS, PNL_NEG, FUNDING_IN, FUNDING_OUT, FEE, FEE_REFUND, LIQUIDATION, SETTLEMENT, INTERNAL_TRANSFER e STATE_ONLY, con mappatura provider e conservazione della causale originale. |
| REQ-SET-02 | MUST | Aperture e chiusure senza regolamento non creano un posting patrimoniale. Fee di apertura/chiusura restano posting distinti se l'exchange le addebita. |
| REQ-SET-03 | MUST | Un PnL realmente accreditato o addebitato varia il saldo dell'asset di regolamento **una sola volta**. Unrealized PnL non è un posting di asset. |
| REQ-SET-04 | MUST | Funding positivo/negativo, rebate e liquidazione sono tracciati come eventi distinti, senza presumerne la classificazione fiscale definitiva. |
| REQ-SET-05 | MUST | In caso di settlement in asset non supportato, il motore non deve forzare USDT/USDC o creare prezzi inventati. |
| REQ-SET-06 | MUST | Se PnL dichiarato dall'exchange è netto di fee/funding, il sistema deve conservare la definizione del dato e riconciliare i dettagli senza sottrazione duplicata. |
| REQ-SET-07 | MUST | Eventi di settlement e fee devono supportare unità, quantità e controvalore EUR secondo la policy di valutazione verificata, con prezzo e fonte consultabili. |
| REQ-SET-08 | MUST | Le operazioni di liquidazione parziale/totale devono produrre gli addebiti/accrediti effettivi, senza inferire che l'intero notional sia una perdita. |
| REQ-SET-09 | SHOULD | Collegate all'evento, le informazioni di posizione e contratto devono permettere di ricostruire perché una liquidazione o chiusura è avvenuta, se il provider le offre. |

### 6.6 Lotti, costo fiscale e altri motori crypto

| ID | Priorità | Requisito e criterio osservabile |
|---|---|---|
| REQ-BAL-01 | MUST | Ogni quantità acquisita o consumata tramite derivati deve essere visibile in tutte le viste di giacenza, senza essere duplicata da un secondo ledger. |
| REQ-BAL-02 | GATED | Il valore fiscalmente attribuito agli asset ricevuti come regolamento deve seguire una regola approvata per giurisdizione/anno/prodotto, evitando doppia imposizione dello stesso importo. |
| REQ-BAL-03 | GATED | Il consumo di asset usati per perdite/fee deve aggiornare quantità e lotti secondo policy approvata, distinguendo onere derivato e possibile effetto fiscale crypto della disposizione. |
| REQ-BAL-04 | MUST | Le viste Calcoli_PlusvalenzeNew, Calcoli_RT e Principale_GiacenzeaData devono utilizzare risultati coerenti; divergenze sulle stesse operazioni devono fallire i test. |
| REQ-BAL-05 | MUST | Calcoli_RW, Calcoli_RW_Giacenze e le giacenze per account devono derivare dagli stessi posting canonici e non contabilizzare una seconda volta margini/nominali. |
| REQ-BAL-06 | MUST | L'invalidazione della classificazione derivati, costo settlement, prezzo EUR, data, gruppo fiscale e account deve propagarsi ai calcoli interessati. |
| REQ-BAL-07 | MUST | Il comportamento degli archivi privi di derivati resta invariato, verificato con golden master per LIFO, RT, RW e giacenze. |
| REQ-BAL-08 | MUST | Le anomalie LIFO o di costo non devono essere «sanate» aggiungendo implicitamente lotti a costo zero; il blocco deve essere diagnosticabile. |

### 6.7 Risultato fiscale derivati

| ID | Priorità | Requisito e criterio osservabile |
|---|---|---|
| REQ-TAX-01 | MUST | Separare esplicitamente redditi c-quater dalle plusvalenze e proventi crypto c-sexies nel riepilogo e nella dichiarazione. |
| REQ-TAX-02 | GATED | Calcolare per anno e rapporto la somma algebrica delle componenti ammesse dalla policy legale, evitando costi già incorporati nei differenziali. |
| REQ-TAX-03 | MUST | Conservare quantità originale, valuta, istante di competenza, prezzo EUR, fonte e trattamento per ogni componente del risultato. |
| REQ-TAX-04 | MUST | Non confondere risultato fiscale con saldo finanziario o con rendimento percentuale del conto. Un trasferimento o il capitale iniziale non è PnL. |
| REQ-TAX-05 | GATED | Gestire perdite riportabili, eventuali compensazioni ammesse, limiti temporali e anni di origine in un prospetto separato, secondo policy annuale approvata. |
| REQ-TAX-06 | GATED | Applicare righi, aliquote, sezioni e arrotondamenti secondo modelli e istruzioni per il **periodo d'imposta** e versione di dichiarazione selezionati. |
| REQ-TAX-07 | MUST | Non confermare come dichiarabile un anno con eventi fiscali non qualificati, prezzi mancanti rilevanti o riconciliazioni ambigue che possono cambiare il risultato. |
| REQ-TAX-08 | MUST | Un cambiamento della policy fiscale produce un report comparativo prima/dopo e traccia la regola/versione utilizzata. |
| REQ-TAX-09 | GATED | Distinguere in modo verificato le eventuali posizioni/rapporti esteri e i margini ai fini W/RW e IVAFE/IC, senza duplicare il valore delle cripto detenute. |
| REQ-TAX-10 | MUST | I dati informativi relativi a prodotti non supportati non devono confluire in totali fiscali definitivi. |
| REQ-TAX-11 | MUST | Dare accesso a una riconciliazione importo-sorgente per ogni totale del report. |

### 6.8 GUI, interazione e comprensibilità

| ID | Priorità | Requisito e criterio osservabile |
|---|---|---|
| REQ-UI-01 | MUST | In tabella movimenti e dettaglio indicare chiaramente asset, tipo derivato, account, stato riconciliazione, fonte, effetto sul saldo e trattamento fiscale applicato/non approvato. |
| REQ-UI-02 | MUST | Aggiungere filtri per account (incluso v[4] se applicabile), tipo derivato, contratto, provider, stato e documento, senza rompere i filtri attuali. |
| REQ-UI-03 | MUST | Consentire la navigazione da evento a tutti i raw record, movimenti e componenti fiscali collegati. |
| REQ-UI-04 | MUST | Mostrare separatamente saldo del conto, margine eventualmente vincolato, PnL realizzato e PnL latente; i valori sconosciuti devono apparire come tali. |
| REQ-UI-05 | MUST | Prevedere una vista di riconciliazione con gruppi ambigui, mismatch e proposte di correzione verificabili. |
| REQ-UI-06 | MUST | Integrare stati e avvisi con la UI già presente, non creare un secondo percorso di salvataggio o gestione errori senza motivazione. |
| REQ-UI-07 | MUST | Ogni correzione che cambia la fiscalità deve mostrare impatti e richiedere conferma; il risultato non deve cambiare silenziosamente. |
| REQ-UI-08 | SHOULD | Presentare PnL/funding/fee per periodo e provider con drill-down, ma mantenere distinzione da report fiscale ufficiale. |
| REQ-UI-09 | MUST | Il percorso dall'import al report deve poter essere completato usando l'interfaccia, senza editare a mano file CSV/DB o chiamare API interne. |

### 6.9 Modifica, eliminazione e storico

| ID | Priorità | Requisito e criterio osservabile |
|---|---|---|
| REQ-EDIT-01 | MUST | Supportare rettifiche manuali di causale/tipo, data, importo, asset, cambio e relazioni fra fonti con audit trail di autore, data, valore precedente, motivo e documento. |
| REQ-EDIT-02 | MUST | Conservare ID sorgente e collegamenti validi anche quando cambia l'ID del movimento legacy; i dati non devono sparire da report o costi fiscali. |
| REQ-EDIT-03 | MUST | Definire comportamento di split/merge/duplicazione/trasla-orario per un evento derivato; non separare automaticamente i due effetti patrimoniale/fiscale. |
| REQ-EDIT-04 | MUST | Cancellazione, annullamento e ripristino sono coerenti con i buffer, lo storico e i documenti già esistenti; nessun evento derivato orfano. |
| REQ-EDIT-05 | MUST | Il reimport non sovrascrive silenziosamente modifiche manuali qualificate; risolve conflitti con scelta esplicita. |
| REQ-EDIT-06 | MUST | Tutti i calcoli e i report che dipendono da un evento corretto sono invalidati e ricostruibili; lo stato precedente rimane ispezionabile. |
| REQ-EDIT-07 | MUST | La modifica di dati presenti in una dichiarazione marcata definitiva deve mostrare che una ristampa può differire dalla versione archiviata. |

### 6.10 Report, export e archivio dichiarazioni

| ID | Priorità | Requisito e criterio osservabile |
|---|---|---|
| REQ-REP-01 | MUST | Rendiconto derivati per anno e per rapporto, con PnL, funding, fee, rimborsi, rettifiche, convertito in EUR, quantità originale e totali. |
| REQ-REP-02 | MUST | Riepilogo c-quater indipendente dal riepilogo c-sexies, con controlli che impediscano compensazioni improprie. |
| REQ-REP-03 | MUST | Prospetto completo di riconciliazione per account/asset/periodo, saldi iniziali, posting, saldi finali e differenze. |
| REQ-REP-04 | MUST | Report di provenienza e modifiche: regola fiscale, versione, file hash, raw record, matching, cambio/fonte, override e blocchi. |
| REQ-REP-05 | GATED | Compilazione delle sezioni del modello 730/Redditi PF pertinente all'anno solo dopo validazione legale e con stato «pronto» verificato. |
| REQ-REP-06 | MUST | Integrare il nuovo output nell'archivio Dichiarazioni (manifest, totali, confronto, impronte, allegati, copia di documenti e relazione al backup). |
| REQ-REP-07 | MUST | I report definitivi devono conservare uno snapshot riproducibile anche se dopo cambiano prezzi, classificazioni e regole. |
| REQ-REP-08 | MUST | L'assenza di informazioni necessarie deve rendere il report preliminare/limitato, non occultare problemi con zeri. |
| REQ-REP-09 | SHOULD | Esportare dettaglio e riepilogo in formati già utilizzati dall'app (ad es. Excel/PDF), con stessa logica di calcolo dei dati visualizzati. |

### 6.11 Persistenza, migrazione, backup e restore

| ID | Priorità | Requisito e criterio osservabile |
|---|---|---|
| REQ-PER-01 | MUST | Conservare il contratto legacy a 46 campi e i dati esistenti salvo migrazione espressamente progettata, retrocompatibile e testata. |
| REQ-PER-02 | MUST | Lo stato derivati necessario al ripristino non può vivere soltanto in cache non salvate o in un risultato UI; dati sorgente, identità e override devono essere durabili. |
| REQ-PER-03 | MUST | Nuove tabelle o nuovi file persistenti devono essere inclusi esplicitamente in Backup_Restore e nel controllo Backup_Compatibilita. |
| REQ-PER-04 | MUST | Migrazioni idempotenti, ripetibili e compatibili con vecchi archivi; fallimenti non devono eliminare dati esistenti. |
| REQ-PER-05 | MUST | Un restore completo deve riprodurre saldi, matching, costi fiscali, perdite pregresse, report e stato delle classificazioni. |
| REQ-PER-06 | MUST | Definire regole di restore parziale e dipendenze: vietato ripristinare un derivato senza fonti/identità necessarie lasciando apparire un risultato corretto. |
| REQ-PER-07 | MUST | Mantiene la provenienza dei dati da CSV/API e la cronologia manuale, rispettando cancellazione e reinserimento dei documenti. |
| REQ-PER-08 | MUST | Il salvataggio deve essere coerente per evento, postings e riferimenti: in caso di errore non deve restare una metà dell'operazione. |
| REQ-PER-09 | SHOULD | Consentire diagnostica/migrazione del pregresso con modalità solo-preview e applicazione esplicita, senza imporre reimport totale. |

---

## 7. Sicurezza, privacy, affidabilità e performance

| ID | Priorità | Requisito |
|---|---|---|
| NFR-01 | MUST | Usare aritmetica decimale per importi, conversioni, somme e riconciliazione; arrotondamenti dichiarati e applicati al punto corretto, non ad ogni step arbitrario. |
| NFR-02 | MUST | Conservare UTC e timezone originale; testare DST, cambio d'anno, minuti/secondi e ordine deterministico a timestamp uguale. |
| NFR-03 | MUST | I record fiscali e le modifiche devono essere riproducibili; stessa sorgente+policy+prezzi ⇒ stesso risultato. |
| NFR-04 | MUST | Non inserire dati personali reali, account ID, documenti integrali o credenziali nei test/repository pubblici; usare fixture sintetiche o anonimizzate. |
| NFR-05 | MUST | Nessun peggioramento del comportamento per archivi senza derivati: equivalenza funzionale e baseline misurata delle prestazioni; regressioni vanno spiegate e approvate. |
| NFR-06 | MUST | Supportare volumi con migliaia di micro-funding; misurare import, matching, ricalcolo, memoria, filtri e report prima e dopo. |
| NFR-07 | MUST | Operazioni lunghe interrompibili con rollback/stato coerente; non lasciare import parziali presentati come completi. |
| NFR-08 | MUST | Il sistema non inventa prezzi, segni, posizioni o rapporti contrattuali in assenza di prove; esplicita ogni inferenza. |
| NFR-09 | SHOULD | Operazioni di riconciliazione e aggregazione incrementali ove dimostrato utile, senza duplicazione massiva dell'archivio. |
| NFR-10 | MUST | La UX di errori e avvisi è accessibile, non dipende solo da colore e consente recupero e comprensione della causa. |

I valori quantitativi di performance non sono fissati arbitrariamente in questo PRD: vanno stabiliti nel piano benchmark a partire da una baseline reale pubblicabile/anonimizzata.

---

## 8. Matrice di copertura end-to-end

Per ogni tipo di evento dichiarato supportato, il revisore deve attraversare tutte le colonne applicabili; «N/A» richiede motivazione.

| Evento | Import + matching | Saldo account | Giacenza/LIFO | Fisco derivati | GUI + modifica | Report + backup |
|---|---|---|---|---|---|---|
| Fund ↔ Spot/Futures | MUST | MUST | MUST | MUST: zero | MUST | MUST |
| Apertura/chiusura senza settlement | MUST | MUST: zero | MUST: zero | MUST: zero salvo fee | MUST | MUST |
| PnL positivo/negativo realizzato | MUST | MUST | GATED costo | GATED regole fiscali | MUST | MUST |
| Funding pagato/ricevuto | MUST | MUST | GATED costo | GATED qualificazione | MUST | MUST |
| Commissione e rimborso commissione | MUST | MUST | GATED costo | GATED onere | MUST | MUST |
| Liquidazione | MUST | MUST | GATED costo | GATED criterio | MUST | MUST |
| Margine vincolato/svincolato | MUST o stato non coperto | MUST: non duplicare | MUST: invariato | MUST: non automaticamente PnL | MUST | MUST |
| Operazione non supportata | MUST: dichiarare limite | MUST: non inventare | MUST: non inventare | MUST: bloccare | MUST | MUST |
| Correzione, sostituzione, reimport | MUST | MUST | MUST | MUST | MUST | MUST |

**Gate di rilascio:** tutte le caselle MUST sono verificate da test o script di accettazione; le caselle GATED diventano MUST per dichiarare la feature fiscale completa. Finché uno dei gate fiscali resta aperto, il software può rilasciare contabilità e diagnostica ma NON un report fiscale «pronto per la dichiarazione».

---

## 9. Scenari numerici di accettazione

Gli scenari sono sintetici: assumono per semplicità 1 USDT = 1 EUR al momento rilevante, costo USDT uguale al valore corrente, nessun'altra movimentazione se non quelle dichiarate. Non definiscono un trattamento giuridico definitivo delle stablecoin.

### AC-01 — Patrimonio iniziale, Spot e trasferimento

**Input:** 2.000 USDT in Fund; trasferimento 1.000 USDT a Spot e acquisto di 1 BTC per 1.000 USDT; trasferimento dei 1.000 USDT residui da Fund a Futures.

**Atteso:** Fund 0 USDT, Spot 1 BTC, Futures 1.000 USDT; totale patrimoniale 1 BTC + 1.000 USDT; nessun reddito derivati attribuito ai trasferimenti o al notional; costi fiscali degli asset coerenti con le operazioni. La neutralità del trasferimento non deve nascondere l'eventuale trattamento fiscale distinto dell'operazione Spot.

### AC-02 — Derivati chiusi in profitto

Partendo da AC-01, il conto Futures registra +400 USDT netti effettivamente regolati (non semplicemente latenti). **Atteso:** Futures 1.400 USDT, patrimoniale totale 1 BTC + 1.400 USDT; differenziale derivato +400 EUR nell'esempio, separato da Spot. Nessun doppio accredito in caso di secondo file descrittivo.

### AC-03 — Derivati chiusi in perdita

Partendo da AC-01, il conto Futures registra -400 USDT netti effettivamente regolati. **Atteso:** Futures 600 USDT, patrimoniale totale 1 BTC + 600 USDT; differenziale derivato -400 EUR nell'esempio, separato da Spot. Il consumo dei lotti segue solo la policy validata: mai assumere costo zero senza dati.

### AC-04 — Fee all'apertura e funding

Partendo da 1.000 USDT disponibili sul conto Futures: apertura nominale 10.000 USDT senza PnL, fee 2 USDT, funding +3 USDT, chiusura PnL +150 USDT, fee 1 USDT. **Atteso:** saldo finale 1.150 USDT (= 1.000 - 2 + 3 + 150 - 1), e componenti del risultato visualizzate separatamente senza usare il nominale come posting. Il corrispondente risultato EUR segue la policy approvata.

### AC-05 — PnL e commissioni già netti

Una chiusura mostra PnL +50 USDT **già netto** delle commissioni -2 USDT, annotate anche in Order History. **Atteso:** accredito effettivo +50 USDT, non +48; il dettaglio fee è riconciliato come componente già inclusa, non diventa secondo addebito. Testare anche export dove PnL +50 è lordo e fee -2 è separata: saldo +48.

### AC-06 — Duplicato tra report

Transaction History e Order History descrivono la medesima chiusura con ID diversi. **Atteso:** un solo risultato e un solo posting, due riferimenti fonte. Reimport in ordine inverso produce stesso saldo e stesso risultato; ripetere n volte non cambia i numeri.

### AC-07 — Fund → Futures con due gambe nello storico

Il provider mostra -1.000 USDT da Fund e +1.000 USDT su Futures, eventualmente nei due file. **Atteso:** nessuna gamba speculare aggiuntiva se entrambe esistono; quadratura zero del patrimonio; reporting specifico per account e storico del trasferimento.

### AC-08 — Overnight anno e timezone

Chiusura del 31 dicembre 2025 alle 16:42 Europe/London, visualizzata il 1 gennaio 2026 alle 00:42 Asia/Shanghai. **Atteso:** un solo evento e anno italiano determinato dalla regola fiscale approvata con timezone corretto; nessuna perdita dell'operazione ai confini degli export.

### AC-09 — Modifica di un evento dell'anno precedente

Dopo report 2025, correggere un PnL, un funding, un timestamp o una fonte prezzo del 2024. **Atteso:** invalidazione e ricalcolo coerenti di 2024 e anni successivi interessati, snapshot dei report definitivi precedenti intatti, differenze visibili.

### AC-10 — Errore e recupero

Import con prezzo EUR mancante, movimento non riconosciuto e trasferimento senza controparte. **Atteso:** eventi visibili e diagnosticati, giacenza non inventata, report fiscale con stato non definitivo; completando i dati e ricalcolando si ottiene risultato stabile.

### AC-11 — Backup, restore, editing

Importare fixture, abbinare un transfer e un fee, eseguire override, produrre un report, fare backup e restore su installazione pulita. **Atteso:** stessi saldi, identificativi, report, documenti, storico e policy; nessun record orfano. Testare anche ripristino parziale autorizzato e conflitti.

### AC-12 — Senza derivati

Usare archivi storici che contengono solo operazioni crypto già gestite. **Atteso:** output LIFO, RT, RW, giacenze, documenti e comportamento UI identici alle baseline, salvo modifiche esplicitamente concordate.

### AC-13 — Fisco e asset di regolamento

Un PnL +100 USDT genera asset ricevuti; successivamente parte degli USDT è utilizzata o venduta. **Atteso:** sistema mostra separatamente reddito derivato e movimenti/costo degli USDT; nessuna doppia tassazione del medesimo componente; eventuale nuovo reddito crypto è calcolato solo se previsto dalla policy approvata per la fattispecie.

### AC-14 — Prodotto fuori scope

Import di contratto inverse coin-margined. **Atteso:** tracciato conservato e identificato «non supportato»; nessun PnL fiscale o saldo sintetico derivato dal notional, nessuna dichiarazione definitiva con quel blocco irrisolto.

---

## 10. Policy fiscale, prove normative e blocchi di conformità

### 10.1 Baseline supportata dalla normativa generale

Per i derivati che ricadono nella lettera c-quater dell'articolo 67 TUIR, l'articolo 68, comma 8, disciplina la somma algebrica dei differenziali e degli altri proventi/oneri. La Circolare 30/E del 27 ottobre 2023, §2.3.2, inquadra i contratti derivati con sottostante cripto in questa categoria, distinta dal regime delle cripto-attività c-sexies.

Per i modelli riferiti al periodo 2025, la guida ufficiale del 730/2026 distingue il quadro T, sezione II (redditi diversi finanziari di cui alle lettere c-bis–c-quinquies), dalla sezione V (cripto-attività). Le istruzioni Redditi PF 2026 distinguono analogamente le sezioni II-A e V-A del quadro RT. **Le mappature effettive devono essere rilette per ogni modello e periodo d'imposta: non hardcodare una posizione eterna.**

**Riferimenti di controllo:**
- Testo storico/temporale articolo 68 TUIR (Dipartimento delle Finanze): https://def.finanze.it/DocTribFrontend/getAttoNormativoDetail.do?ACTION=getArticolo&articolo=Articolo+68&codiceOrdinamento=0000000000000680000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000&id=%7B31D694E8-4398-4030-873B-FEAF5A6647F9%7D
- Circolare n. 30/E del 27/10/2023, documento Agenzia Entrate: https://www.agenziaentrate.gov.it/portale/documents/20143/5589638/Circolare+criptoattivita+del+27+ottobre+2023.pdf/1154a95a-80ea-a6ec-bcc0-731b844db9e6
- Istruzioni Agenzia Entrate quadro T per 730/2026: https://infoprecompilata.agenziaentrate.gov.it/portale/quadro-t
- Istruzioni Agenzia Entrate quadro RT per Redditi PF 2026: https://infoprecompilata.agenziaentrate.gov.it/portale/quadro-rt

### 10.2 Legal Decision Register — obbligatorio prima del rilascio fiscale

| Decisione | Da validare | Se non validata |
|---|---|---|
| LEG-01 | Qualificazione per tipologia concreta di contratto (non solo etichetta «Futures») e tipo di regolamento | Import informativo; report fiscale bloccato |
| LEG-02 | Momento di competenza per differenziali realizzati, parziali, liquidazioni, settlement periodici, margini e posizioni aperte | Nessuna tassazione automatica |
| LEG-03 | Natura fiscale di funding, fee, rebate, voucher e recuperi, inclusione nel PnL del provider e limiti di oneri | Esposizione contabile, non imponibile finale |
| LEG-04 | Valore fiscale degli USDT/USDC ricevuti, uso di asset preesistenti per fee/perdite e possibile effetto c-sexies successivo | Lotti con stato di policy aperta e report bloccato dove materiale |
| LEG-05 | Minusvalenze di derivati: compensazioni ammesse, riporto e relative modalità per anno, regime e modello | Non applicare compensazione automatica |
| LEG-06 | Valutazione EUR per asset di regolamento, prezzi, data di competenza e metodi documentabili | Totali preliminari |
| LEG-07 | Monitoraggio W/RW, natura del rapporto estero, margine, eventuale IVAFE e/o imposta crypto senza doppio conteggio | Non generare righi automatici di derivati |
| LEG-08 | Punti dei quadri 730/Redditi PF, aliquote e arrotondamenti per ciascun anno e tipo di reddito | Nessun PDF fiscale definitivo |
| LEG-09 | Eventuali casi di differenza fra accordo/settlement specifico di un provider e categoria generale c-quater | Richiedere revisione esperta |

**Ciascuna decisione LEG dovrà avere:** fonti esatte, testo o interpretazione applicata, validità temporale, responsabile della revisione, alternative considerate, effetto sui dati, versione policy e casi di test. Non trasformare in default implicito una regola ricavata soltanto dalla documentazione di un software terzo.

---

## 11. UX e interoperabilità: workflow minimo da preservare

### 11.1 Percorso importazione

L'utente seleziona provider, account, fonte e documento; la prova mostra copertura, parsing, causali e posting proposti; la riconciliazione confronta i report e restituisce match certi/ambigui; soltanto i posting validati aggiornano l'archivio. Tutte le azioni devono essere richiamabili dai menu esistenti e rispettare annullamento/interruzione.

### 11.2 Percorso verifica

La vista per sotto-account mostra per ciascun asset: saldo iniziale noto o ignoto, trasferimenti, risultati, fee, funding, saldo ricostruito, saldo ufficiale e differenza. Nel dettaglio di ogni riga si apre l'evidenza documentale. La sommatoria dei sotto-account coincide con il patrimonio globale, senza collaterale contato una seconda volta.

### 11.3 Percorso modifica

Un movimento può essere corretto con motivazione, senza cancellare la fonte originale. Le correzioni confluiscono nei ricalcoli e nei report. L'utente può consultare le versioni precedenti e capire quali prospetti cambiano.

### 11.4 Percorso dichiarazione

Una sezione distinta per i derivati mostra stato di completezza, questioni normative eventualmente irrisolte, prospetto annuo e riconciliazioni. Finché ci sono blocchi, il sistema può esportare documentazione di lavoro chiaramente preliminare, ma non un report presentato come pronto alla dichiarazione.

---

## 12. Requisiti di verifica, test e DoD

### 12.1 Prove indispensabili

1. Test unitari delle regole di normalizzazione/segni/fee/netto/lordo, senza dipendere dalla GUI.
2. Test di proprietà: conservazione quantità, zero-sum dei trasferimenti, idempotenza, ordine di import ininfluente.
3. Test di integrazione: raw → canonical → posting → saldo → proiezione fiscale → report/GUI.
4. Test storici: modifica, eliminazione, ricostruzione, annulla, reimport, documenti, backup e restore.
5. Golden master su archivi crypto senza derivati, con risultato invariato; fixture sintetiche nel repository e dataset realistici privati fuori Git.
6. Prove anno fiscale, DST, cambio USDT/EUR variabile, prezzi mancanti e valori microscopici.
7. Confronto tra ricalcolo intero e incrementale, con test di invalidazione del campo 44 e delle nuove policy.
8. Benchmark base vs feature su dataset comparabili: migliaia di funding, molte posizioni, import sovrapposti, memoria di picco.
9. Verifica manuale GUI/forme e stampe PDF/Excel su almeno un caso completo e uno bloccato.
10. Riconciliazione con estratti reali pseudonimizzati e valori certificati indipendentemente dal programma.

Suite da cui partire: DerivatiTest; ImportazioneGenericaAvvisoDerivatiTest; ImportazioneGenericaBybitTest; CommissioniCollegateTest; GruppoOperazioneTest; CalcoliPlusvalenzeNewEquivalenzaIncrementaleTest; CalcoliPlusvalenzeNewGoldenMasterTest; Calcoli_RW_GoldenMasterTest; Calcoli_RW_GiacenzeTest; GUI_ModificaMovimentoSoloPrezzoTest; Principale_Movimenti_SeparaUnisciTest; DocumentiFonteTest; MovimentiStoricoTest; Backup_RestoreTest; Backup_CompatibilitaTest; DichiarazioniTest.

**Limitazione esistente:** alcuni golden master su dati reali dipendono da nocommit/GoldenMaster, assente dal repository pubblico; occorrono fixture sintetiche versionate e baseline locali conservate fuori Git.

### 12.2 Definition of Done per un vertical slice

Un requisito non è chiuso soltanto perché la classe corrispondente compila. Per ogni tipo di evento supportato vanno controllati: import, preview, posting, saldo, lotti, risultato derivato, filtro, dettaglio, modifica, cancellazione, ricalcolo, report, backup e restore. Le voci non applicabili devono essere giustificate, non ignorate.

### 12.3 Release gate

- **RG-01:** zero differenze contabili non spiegate su fixture supportate; saldi riconciliati o esplicitamente incompleti.
- **RG-02:** zero duplicati di posting o reddito su reimport e fonte multipla.
- **RG-03:** nessuna regressione non approvata nei golden master senza derivati.
- **RG-04:** passata incrementale e integrale equivalenti per tutti i casi verificati.
- **RG-05:** backup e restore completi equivalenti; compatibilità vecchi archivi verificata.
- **RG-06:** tutti i blocchi fiscali materiali risolti e policy normative approvate prima del report dichiarativo definitivo.
- **RG-07:** GUI completa, errori comprensibili, tutte le operazioni fondamentali disponibili senza editor di database.
- **RG-08:** baseline/benchmark registrati e risultati comparati senza regressioni prestazionali non motivate.
- **RG-09:** nessuna assunzione nascosta per valori EUR, fiscalità di stablecoin o proprietario dell'account.

---

## 13. Dipendenze con i componenti esistenti

Le seguenti classi sono **touchpoint da analizzare**, non istruzioni a modificarle tutte.

| Tema | Classi/componenti principali |
|---|---|
| Import e registrazione | ImportazioneGenerica; Importazioni; Importazioni_Gestione; ProvaConfigurazioneImport; DocumentiFonte; ScartiImport |
| Saldi, gruppi e trasferimenti | MovimentiCrypto; GUI_ClassificazioneMovimento; GirocontiFiat; DatabaseH2; Principale_GruppiWalletRW |
| Identità contratti e commissioni | Derivati; GruppoOperazione; OperazioniCalcolate; CommissioniCollegate; Principale_CommissioniCollegate |
| Kernel crypto | Calcoli_PlusvalenzeNew; Calcoli_RT; Principale_GiacenzeaData; Prezzi |
| Monitoraggio | Calcoli_RW; Calcoli_RW_Giacenze; Calcoli_RW_Fiat; Calcoli_RW_PeriodiCrypto; Principale_QuadroRW |
| Lifecycle movimenti | GUI_ModificaMovimento; Principale_Movimenti_SeparaUnisci; Principale_TraslaOrario; MovimentiStorico; MovimentiCollegati |
| UI | Principale; Principale_FiltriMovimenti; GUI_DettaglioTransazione; GUI_GestioneWallets; GUI_ProvaConfigurazione |
| Reporting | Principale_RT_Tipologie; Principale_Dichiarazioni; Dichiarazioni; Stampe |
| Persistenza | DatabaseH2; Backup_Restore; Backup_Compatibilita; VarStatiche |

**Attenzione:** il file Principale.java è molto esteso e non ancora ispezionato end-to-end; deve essere censito con i callback delle schermate e le dipendenze reali prima della specifica di implementazione. L'assenza di una classe da questa tabella non significa che sia fuori impatto.

---

## 14. Piano documentale e di delivery, non ancora implementation plan

| Gate | Output verificabile | Condizione di uscita |
|---|---|---|
| G0 — Completamento audit | Touchpoint UI e Principale, test/build e benchmark baseline, mappa flussi | Tutti i punti critici mappati o dichiarati come gap |
| G1 — Approvazione PRD | Questo PRD revisionato + decisioni di perimetro/prodotto | Requisiti e AC accettati |
| G2 — System Architecture Spec + ADR | Contratti fra raw event, posting, asset, fiscalità; decisioni su storage, identità e migrazione | Invarianti dimostrabili, nessuna sovrapposizione non motivata |
| G3 — Specifiche modulari | Ingest/reconciliation; account/transfer; settlement/LIFO; tax; UI/editor; report/backup; performance | Ogni requisito collegato a un componente e test |
| G4 — Vertical slice contabile | Import reale → riconciliazione → saldo → UI → modifica → backup → report diagnostico | AC-01,04,05,06,07,08,10,11 e gate pertinenti |
| G5 — Vertical slice fiscale | Policy approvate → settlement bridge → motori e report c-quater → verifica fiscale | AC-02,03,09,13 e LEG risolti |
| G6 — Validazione di sistema | Suite completa, benchmark, audit di impatto e regressione, migrazioni e prove restore | Tutti i release gate soddisfatti |

Ogni piano tecnico deve indicare file/contratti toccati, rischi, sequenza, test da aggiungere, procedura di rollback, osservabilità e criteri di accettazione. La progettazione deve essere globale, **l'implementazione incrementale per flussi verticali completi**.

---

## 15. Registro delle decisioni aperte per il prossimo passaggio

| ADR candidato | Questione | Default non autorizzato |
|---|---|---|
| ADR-01 | Dove persistono identità economiche, raw link, override e classificazione? Schema H2 aggiuntivo, modello legacy, o approccio misto | Non duplicare tutto il registro senza prova |
| ADR-02 | Autorità, deduplica e matching provider-specifici | Non affidarsi al solo timestamp |
| ADR-03 | Contratto dei sotto-account e differenza da gruppi fiscali | Non sostituire retroattivamente il nome exchange |
| ADR-04 | API comuni per lotti, calcoli RT e giacenze a data | Non implementare quarta copia del LIFO |
| ADR-05 | Eventi fiscali derivati, versioni normativa e dependency/invalidation | Non produrre imposte dal solo saldo |
| ADR-06 | Workflow e rollback di modifica/cancellazione/restore | Non lasciare dati orfani |
| ADR-07 | Filtro UI, navigazione fonte e stato report | Non nascondere le incertezze |
| ADR-08 | Strategia per archivi storici e doppie sorgenti già importate | Nessuna migrazione distruttiva |
| ADR-09 | Strategia benchmark e granularità degli eventi ad alta frequenza | Non aggregare irreversibilmente prima del matching |
| ADR-10 | Supporto fiscale dei prodotti oltre il perimetro iniziale | Non classificare tutto come c-quater automaticamente |

---

## 16. Criterio finale di approvazione del PRD

Per approvare questo PRD bisogna confermare: (i) il perimetro iniziale di prodotti e provider, (ii) gli invarianti INV, (iii) i requisiti MUST e i casi AC, (iv) i gate legali e chi è autorizzato a chiuderli, (v) la distinzione fra report diagnostico e fiscale definitivo, (vi) le dipendenze e i vincoli anti-regressione.

**Non confondere approvazione del PRD con completamento della feature.** La conformità effettiva richiede implementazione verificata, test, migrazioni e audit dei risultati fiscali. Il master resta immutato fino a un'integrazione esplicitamente approvata.
