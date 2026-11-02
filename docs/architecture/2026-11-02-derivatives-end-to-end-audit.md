# Audit architetturale — integrazione nativa dei derivati in GiacenzeCrypto

**Data:** 2026-11-02  
**Repository:** `dimkadiver-cpu/GiacenzeCrypto`  
**Branch di lavoro:** `feature/derivatives-integration-design`  
**Baseline del fork:** `24c9b4ab3851caf4bceac7caf252bcb18df4f70d`  
**Stato:** audit statico preliminare, propedeutico a PRD master, ADR, specifica architetturale e piani modulari.  
**Nessuna modifica al runtime.** Non sono stati eseguiti build, test Maven, prove GUI o benchmark. Non costituisce una certificazione fiscale.

## 1. Scopo e criterio di completezza

Obiettivo: introdurre futures/perpetual come **dominio nativo** e non come importatore accessorio, con percorso completo da documento/API di origine a contabilizzazione, giacenze, costo fiscale, imposta, GUI, modifiche, report e backup. Il livello di integrazione si giudica *end-to-end* per i prodotti espressamente supportati, non dalla mera presenza di nuove classi.

Principi da confermare nel PRD:

1. Un solo patrimonio in asset, distinto per account operativi e per gruppi fiscali; **nessuna duplicazione di saldi** fra registro crypto e registro derivati.
2. I contratti e le loro esposizioni non sono compravendite Spot del sottostante; nominale e margine non sono automaticamente spese, né ricavi.
3. Il risultato derivato (c-quater, ove applicabile) e il trattamento dell'asset di regolamento sono **due effetti di un medesimo evento**, non due importazioni indipendenti.
4. Fund/Spot/Futures sono sotto-account contabili: i trasferimenti fra conti dello stesso titolare non producono da soli un reddito derivato.
5. L'interpretazione fiscale e il suo periodo di efficacia vanno versionati; i punti normativi controversi non si codificano come verità implicite.
6. Nessuna nuova logica deve alterare gli archivi senza derivati: equivalenza storica dei calcoli e dei report.

Perimetro candidato del primo prodotto interamente supportato: futures/perpetual **lineari regolati in USDT/USDC** con PnL realizzato, funding, fee, liquidazioni, trasferimenti e documenti di origine. **Non è ancora una decisione del PRD.** Inversi coin-margined, consegna fisica, opzioni, portfolio margin, prestiti e liquidazioni con asset multipli vanno esplicitamente ammessi dopo verifica oppure segnalati come non supportati.

## 2. Inventario verificato della codebase

| Area | Esistente | Lacuna rispetto ai derivati |
|---|---|---|
| Registro movimenti | [`Importazioni.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Importazioni.java#L155-L171): `ColonneTabella=46`, `MappaCryptoWallet`; serializzazione su file `movimenti.crypto.db` | Non esiste una rappresentazione separata e verificata del settlement fiscale |
| Modello movimento | [`MovimentiCrypto.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/MovimentiCrypto.java#L142-L154): `creaMovimento`, due gambe, costo/valore, tipologia | Le causali derivati sono ricondotte a trasferimenti/reward/commissioni crypto |
| Marcatura derivati | [`Derivati.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Derivati.java#L27-L66): campo 44, PNL/FUNDING/COMMISSIONE ecc. | Marcatura informativa: nessun motore la utilizza per tassazione |
| Identità operazione | [`GruppoOperazione.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/GruppoOperazione.java#L14-L51): campo 45, identità contratto; [`OperazioniCalcolate.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/OperazioniCalcolate.java#L110-L195) raggruppamento derivato in memoria | L'identità calcolata `OP-<ID>` non è una chiave persistente da usare come evento fiscale |
| Collegamento commissioni | [`CommissioniCollegate.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/CommissioniCollegate.java#L14-L42): campo 43, collegamento per gruppo | Servono regole di onere c-quater e prevenzione del doppio addebito |
| Import generico | [`ImportazioneGenerica.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/ImportazioneGenerica.java#L153-L298): parser JSON, timestamp/fuso, batch prezzi, raggruppamento, scrittura | Non concilia nativamente Transaction History con Order History per unità economica derivata |
| Account secondari | [`ImportazioneGenerica.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/ImportazioneGenerica.java#L1319-L1324): `nomeExchange` / `nomeWallet`; [`ImportazioneGenerica.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/ImportazioneGenerica.java#L2545-L2571) gamba speculare | `[3]` = exchange fiscale, `[4]` = sottowallet; mancano identità/account e controlli saldi completi |
| Trasferimenti e LIFO | [`GirocontiFiat.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/GirocontiFiat.java#L93-L146), [`Calcoli_PlusvalenzeNew.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Calcoli_PlusvalenzeNew.java#L1025-L1075) | Il trasferimento `TI` non equivale automaticamente a PTW/DTW collegati quando i gruppi fiscali differiscono |
| Kernel crypto | [`Calcoli_PlusvalenzeNew.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Calcoli_PlusvalenzeNew.java#L811-L835): lotti LIFO, campi 16/17/19/33, checkpoint | Il campo 44 non decide trattamento fiscale né entra nelle impronte correnti |
| Analisi RT | [`Calcoli_RT.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Calcoli_RT.java#L45-L103), [`Principale_RT_Tipologie.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Principale_RT_Tipologie.java#L82-L127) | Aggrega `v[19]` del motore crypto; non separa i redditi c-quater |
| Giacenze a data | [`Principale_GiacenzeaData.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Principale_GiacenzeaData.java#L641-L725) | Terza copia delle regole dei lotti; rischio divergenza dopo modifiche |
| RW/monitoraggio | [`Calcoli_RW.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Calcoli_RW.java#L993-L1073), [`Calcoli_RW_Giacenze.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Calcoli_RW_Giacenze.java#L159-L186), [`Calcoli_RW_Fiat.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Calcoli_RW_Fiat.java#L24-L54) | Non c'è modello verificato per margine/rapporto derivato estero; evitare double count e inferenze normative |
| Persistenza | [`DatabaseH2.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/DatabaseH2.java#L295-L377): `DOCUMENTIFONTE`, `MOVIMENTI_STORICO`, `WALLETGRUPPO`, `GRUPPO_ALIAS` | Nuovo stato derivati, se aggiunto in H2, richiede DDL e migrazione idempotente |
| Provenienza | [`DocumentiFonte.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/DocumentiFonte.java#L226-L328): documenti, hash, campo 41 | Mapping stabile raw record → evento → movimento e rettifiche, con cancellazione coerente |
| Modifica e lineage | [`MovimentiStorico.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/MovimentiStorico.java#L12-L56), [`GUI_ModificaMovimento.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/GUI_ModificaMovimento.java#L1330-L1385) | Una rettifica dell'importo o tipo deve invalidare TUTTE le proiezioni, non soltanto il LIFO |
| Operazioni su movimenti | [`Principale_Movimenti_SeparaUnisci.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Principale_Movimenti_SeparaUnisci.java#L58-L87) | Unione, separazione, copia, traslazione, ricalcolo prezzi, duplicazione: contratto di comportamento da esplicitare |
| UI e filtri | [`Principale_FiltriMovimenti.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Principale_FiltriMovimenti.java#L49-L103), [`GUI_DettaglioTransazione.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/GUI_DettaglioTransazione.java#L60-L100), [`GUI_ProvaConfigurazione.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/GUI_ProvaConfigurazione.java#L24-L55) | Filtro per account operativo e tipo derivato, pannello riconciliazioni, campi modificabili, indicatori di errore |
| Dichiarazioni e report | [`Dichiarazioni.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Dichiarazioni.java#L22-L136): manifest, famiglie RW/RT, definitive; [`Principale_Dichiarazioni.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Principale_Dichiarazioni.java#L229-L315) | Output autonomo c-quater, metodo/versione calcolo, riconciliazione, snapshot fiscalmente riproducibile |
| Backup e restore | [`Backup_Restore.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Backup_Restore.java#L290-L396), [`Backup_Compatibilita.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Backup_Compatibilita.java#L11-L28) | Le nuove tabelle NON entrano automaticamente nel backup: aggiornare elenco, manifest e compatibilità |
| Errori / recupero | [`ScartiImport.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/ScartiImport.java#L31-L69): recupero da raw source, attualmente focalizzato su casi documentati | Stati di mismatch, mancata controparte, record sconosciuto, prezzi assenti, recovery sicuro per derivati |

### 2.1 Contratto della riga legacy: 46 campi

Verificato `Importazioni.ColonneTabella=46`, con:
- `[0]` ID movimento; `[1]` data; `[3]` exchange/wallet principale; `[4]` sottowallet; `[5]` categoria descrittiva; `[7]` causale originale;
- `[8..10]` gamba uscita; `[11..13]` gamba entrata; `[15]` controvalore; `[16]` e `[17]` costi fiscali; `[18]` sottotipo; `[19]` plusvalenza calcolata;
- `[20]` movimenti collegati; `[22]` manuale/importato/generato; `[24]` id/hash esterno; `[29]` timestamp; `[33]` flag di calcolo; `[38]` anomalie;
- `[41]` documento di origine; `[42]` lignaggio; `[43]` commissioni collegate; `[44]` marcatore derivati; `[45]` gruppo/identità operazione.

**Vincolo:** non introdurre nuovi significati su campi occupati né aumentare `ColonneTabella` per convenienza. Qualsiasi estensione deve avere schema, retrocompatibilità, serializzazione, copia, editor, backup e golden master verificati. Un eventuale sidecar H2 deve usare chiavi stabili e sincronizzate col ciclo di vita dei movimenti.

### 2.2 Differenza tra exchange, sotto-account e gruppo fiscale

`ImportazioneGenerica` chiama `MovimentiCrypto.creaMovimento(..., exchange, wallet, ...)`, ottenendo `v[3]=exchange` e `v[4]=nomeWallet` (`Fund`, `Spot`, `Futures` ecc.). Il codice documenta che `[3]` governa gruppo fiscale e deduplica dell'import, mentre `[3];[4];token` governa controlli saldi per sotto-wallet. [`MovimentiCrypto.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/MovimentiCrypto.java#L380-L405) e [`ImportazioneGenerica.java`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/ImportazioneGenerica.java#L2545-L2567).

**Decisione consigliata:** preservare `[3]=BingX` per gli archivi esistenti e introdurre/normalizzare l'identità operativa di account senza creare gruppi fiscali autonomi. Non cambiare retroattivamente `nomeExchange`: altererebbe grouping, dedup e prospetti. Verificare le limitazioni UI: il filtro di caricamento attuale su wallet legge `v[3]` o gruppo, non `v[4]`.

## 3. Criticità architetturali da affrontare

### P0 — Tre percorsi dei lotti e due percorsi fiscali

`Calcoli_PlusvalenzeNew.ElaboraMovimento`, `Calcoli_RT.CalcoliPlusvalenzeXAnno` e `Principale_GiacenzeaData.ElaboraLotti` contengono logiche proprie di carico/scarico dei lotti; `Calcoli_RW` e `Calcoli_RW_Giacenze` leggono lo stesso registro per il monitoraggio. La Javadoc delle giacenze dice esplicitamente che è la *terza copia* delle regole del LIFO. Il nuovo comportamento non può essere implementato soltanto in `Calcoli_PlusvalenzeNew`.

**Guardrail:** definire prima una matrice degli effetti per tutti i motori, poi consolidare le regole condivise solo se la modifica è comprovata dai test. Vietate due formule indipendenti per lo stesso trattamento di settlement.

### P0 — La marcatura dei derivati è ancora informativa

`Derivati.CAMPO=44` non entra nei calcoli. Oggi PnL e funding vengono importati come movimenti mono-gamba; le regole crypto possono valorizzarli a costo zero, escluderli o trattarli secondo classificazioni non pertinenti. La nuova soluzione deve:
- conservare la variazione reale dell'asset di regolamento nel patrimonio;
- attribuire il risultato derivato separatamente;
- assegnare ai lotti il trattamento fiscale approvato, con provenienza verificabile;
- impedire che il medesimo differenziale venga tassato di nuovo come semplice provento crypto.

### P0 — Riconciliazione e idempotenza tra fonti diverse

`DocumentiFonte` conserva l'hash del documento; `ScriviListaSuMappaCrypto` deduplica per ID movimento. Questa identità NON è sufficiente per deduplicare il medesimo PnL derivato descritto da due report con ID di riga differenti (es. Transaction History e Order History). Servono:
- chiave di evento sorgente stabile per provider/account/record;
- legame tra fonti e un evento economico canonico;
- matching deterministico prima, tolleranze dichiarate solo come ausilio e con ambiguità segnalata;
- distinzione `matched / unmatched / ambiguous / excluded / overridden`;
- gestione sicura del reimport, sovrascrittura, CSV diversi sullo stesso periodo e rettifiche.

**Non usare** il solo timestamp, un ID artificiale del movimento o `OperazioniCalcolate` per l'identità fiscale: questa classe calcola gruppi per la UI, anche in base a hash e finestre temporali, e non li salva.

### P0 — I movimenti interni devono pareggiare

`walletSpecularePerCausale` può emettere una seconda gamba `TI` senza stato; per ciascun account modifica il saldo, sul gruppo fiscale aggregato deve sommare zero. Se il provider consegna anche l'altra gamba reale, **non** generare automaticamente entrambe su ogni fonte: raddoppierebbe i trasferimenti. Decidere regola di autorità delle fonti per Fund/Spot/Futures, ricomporre origin/destination e verificare eventuali fee.

Se `PlusXWallet` crea pile fiscali distinte, il collegamento `PTW/DTW` e i riferimenti campo 20 richiedono test specifici. Non assumere che una riga `TI` con sotto-wallet diverso ricostruisca autonomamente i lotti su gruppi fiscali distinti.

### P0 — Cambiamento di un evento e dipendenze di ricalcolo

`Calcoli_PlusvalenzeNew` conserva impronte per movimento e `OpzioniRicalcolo.Epoca()`; le informazioni correnti del derivato non sono ancora fra gli input fiscali. Quando la loro classificazione diventerà determinante, qualsiasi modifica a `[44]`, costo settlement, regola fiscale, importo, data, account o collegamento deve invalidare il calcolo dal punto più antico effettivamente coinvolto, o forzare passata integrale. Confrontare passata incrementale e completa sui dataset di test.

### P1 — Editing, annullamento e provenienza

**Esito dell'approfondimento statico (2026-11-02): il sistema di classificazione manuale esiste già e va riutilizzato**, non replicato per i derivati. Le principali routine verificate:

- [`GUI_ClassificazioneMovimento`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/GUI_ClassificazioneMovimento.java) espone reward, acquisto, costo zero, trasferimento tra wallet, commissioni/cash-out, donazione/furto, collaterale, prestito, scambio differito. Supporta classificazioni singole e alcune multiple, collegamenti e annullamento.
- `ClassificaMovimenti(...)` chiama la pulizia dei collegamenti e muta **in place** i campi dell'array originale `[5]`, `[17]`, `[18]`, `[19]`, `[20]`, `[21]` e, in alcuni casi, prezzo `[15]`/`[35]`. **Non chiama direttamente `MovimentiStorico.AccodaModifica`**: la storia autonoma delle scelte di classificazione non risulta garantita da questo metodo.
- `RiportaTransazioniASituazioneIniziale(...)` scioglie `[20]`, rimuove gli eventuali movimenti generati `[22]=AU`, annulla la classificazione e ripristina il prezzo temporaneo. Il contratto degli override derivati dovrà evitare che un annullamento perda eventi economici o riferimenti.
- [`GUI_ModificaMovimento`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/GUI_ModificaMovimento.java) supporta la modifica manuale e usa `MovimentiStorico`; sui generati `AU` o sui movimenti collegati (`[20]`) limita la modifica a prezzo e note, salvo percorso che scioglie il gruppo.
- [`Importazioni`](../../src/main/java/com/giacenzecrypto/giacenze_crypto/Importazioni.java) con `Sovrascrivi esistenti` annulla classificazioni preesistenti, incrementa `ClassificazioniAnnullate` e avvisa. Ciò **non equivale alla conservazione di un override economico**; la nuova feature richiede confronto manuale/automatico prima della sovrascrittura.
- `MovimentiStorico` conserva il lignaggio `[42]` delle modifiche accodate e scrive il buffer allo stesso salvataggio dei movimenti; `DocumentiFonte` conserva il documento di origine e coordina le eliminazioni.

**Gap non coperti oggi**: categoria economica per derivati distinta da deposito/reward; editing di N componenti economiche di un unico delta saldo; verifica `sum(components) = observedDelta` per asset; override economico durevole e relazionato all'evento canonico; storico delle classificazioni; preservazione/risoluzione dei conflitti al reimport; commit/undo e invalidazione coerenti tra saldo, LIFO e report T/RT.

**Contratto richiesto dalla Design Spec:** la classificazione economica cambia il tipo (non il saldo); la correzione contabile può cambiare il delta e deve riconciliarsi; la policy fiscale è una terza dimensione. Il raw source resta immutabile; `automatic` e `manualOverride` compongono la proiezione `effective`. Un override deve sopravvivere a reimport identico, ma non deve essere applicato automaticamente a un evento diverso dopo un cambiamento del parser. Un conflitto blocca il report fiscale pertinente, senza inventare poste di quadratura.

Vedi [Design Spec eventi canonici, §10.6–10.11](2026-11-02-derivatives-canonical-event-contract-design.md) e [PRD di riferimento, §6.9](../product/2026-11-02-derivatives-integration-prd.md#69-modifica-eliminazione-e-storico).

**Limite:** questo approfondimento deriva dalla lettura delle routine nominate; resta da ispezionare integralmente il wiring `Principale.java`, eseguire test GUI e verificare gli effetti runtime dei diversi percorsi.

### P1 — Backup, stampe e dichiarazioni

`Backup_Restore.Tabelle` registra nominativamente le tabelle persistenti. Se si introducono tabelle di dominio derivati, inserirle nel gruppo `MOVIMENTI` o in gruppo specifico con dipendenze esplicite; verificare anche:
- `Backup_Compatibilita` (schema e chiavi), manifest, opzioni e prezzi;
- ripristino totale e parziale, cancellazione/merge delle tabelle, invalidazione cache;
- `Dichiarazioni`: immutabilità delle stampe definitive, tracciabilità dei valori, allegati di origine;
- RT/T e RW/W: non mescolare reddito c-quater e c-sexies; il monitoraggio di rapporto derivato/margine richiede policy giuridica specifica.

### P1 — Precisione e cut-off temporali

Conservare quantità con `BigDecimal`, timestamp sorgente, timezone sorgente, istante UTC e criterio di valorizzazione in euro (prezzo storico, fonte, eventuale override). Testare passaggio d'anno e ora legale; per confronto multi-fonte non bastano stringhe locali di data.

**Non assumere** 1 USDT = 1 EUR, né convertire ogni evento con il cambio del 31 dicembre. Costo fiscale e valore EUR alla data rilevante non sono sinonimi.

### P1 — Performance e UX

Il registro deve supportare import a lotti, aggregazioni periodiche deterministiche, ricalcolo incrementale e stato sincronizzato in UI. I checkpoint LIFO sono già su blocchi di 5.000 movimenti: conservare il caso senza derivati come baseline e verificare carico memoria/tempo su import massivi. La prova JSON esistente può essere estesa a: record canonici, matching fonti, scarti e quadratura per account.

## 4. Architettura da sottoporre a Design Spec (NON ancora decisione implementativa)

```text
Documento CSV/API, con record grezzi conservati
      |
Normalizzazione sorgente -> RawEvent (id deterministico, exchange, account, asset, tipo, UTC, origine)
      |
Riconciliazione / deduplicazione multi-report -> CanonicalEconomicEvent
      |
      +--> Proiezione conto/account (quantità e disponibilità, trasferimenti fra subaccount)
      |
      +--> Proiezione patrimonio crypto e lotti (settlement dell'asset; singola fonte di verità)
      |
      +--> Proiezione fiscale derivati (c-quater, PnL/funding/fee, valori EUR, carry)
      |
      +--> Audit trail / report / GUI / backup
```

**Nota:** potrebbe essere preferibile conservare le proiezioni come calcoli derivabili, e non come copie persistenti. È un ADR da decidere dopo i benchmark e il censimento delle operazioni di modifica. Gli eventi canonici non devono creare due scritture permanenti indipendenti dello stesso saldo.

Possibili riusi:
- `Derivati` per classificazione del movimento;
- `GruppoOperazione` / `OperazioniCalcolate` per visualizzazione e legami già esistenti; distinguere l'identità stabile fiscale dai gruppi effimeri della UI;
- `CommissioniCollegate` per attribuzione fee; usare un criterio di matching che non le conti due volte;
- `MovimentiStorico` / `DocumentiFonte` per provenienza e rettifiche;
- `MovimentiCrypto` e `Importazioni` per i movimenti patrimoniali, senza adattare i nominali dei contratti a scambi del sottostante;
- `Prezzi` e `Calcoli_PlusvalenzeNew` per valorizzazione EUR/lotti, con contratto comune e regole approvate.

## 5. Matrice di comportamento obbligatoria

| Caso | Saldi per account | Patrimonio complessivo | Reddito derivati | Lotti/costo fiscale | Audit/report |
|---|---|---|---|---|---|
| Fund → Futures, stesso titolare | `-q / +q` | `0` | `0` | invariato | sorgente e controparte |
| Margine vincolato | spostamento da disponibile a impegnato se tracciato | `0` | `0` | invariato | posizione/stato |
| Open/Close con nominale | non usare nominale come deposito/prelievo | `0` salvo settlement/fee | `0` salvo settlement/fee | invariato salvo fee | metadati contratto |
| PnL realizzato positivo | `+q` asset regolamento | `+q` | `+V_{EUR}` | lotto di entrata secondo policy validata | fonte e cambio |
| PnL realizzato negativo | `-q` | `-q` | `-V_{EUR}` | consumo lotti, eventuale effetto crypto secondo policy | fonte e cambio |
| Funding incassato/pagato | `+q/-q` | `+q/-q` | componente da policy c-quater | come regolamento | dettaglio |
| Fee trading/chiusura | `-q` | `-q` | onere se pertinente e non già netto | consumo lotti, trattamento validato | collegamento |
| PnL solo latente | nessun flusso finché non regolato | nessuna quantità movimentata | nessun reddito realizzato *salvo regimi/settlement da verificare* | invariato | informazione separata |
| Doppio report stesso PnL | un unico posting | un unico posting | una sola componente | una volta | entrambe le fonti |
| Cancellazione/sostituzione file | ricalcolo reversibile | quadratura riconfermata | risultato ricalcolato | lotti ricalcolati | nessun orfano |

Tutte le quantità con segno; per le colonne di fiscalità in EUR la tabella rappresenta un **contratto funzionale di progetto**, non un parere sulla qualificazione normativa di ogni voce.

## 6. Decisioni normative da validare prima del calcolo fiscale definitivo

Per ciascuna: fonte normativa o prassi, versione temporale, owner della decisione, test corrispondente.

1. Classificazione dei diversi tipi di contratti lineari/inversi, piattaforme e natura del regolamento.
2. Competenza fiscale di PnL regolato periodicamente, unrealized PnL e liquidation; differenze per prodotti particolari.
3. Qualificazione e imputazione del funding (non assumere una regola universale senza fonti).
4. Commissioni/oneri inclusi nel PnL dell'exchange versus separati; recuperi e rebate.
5. Costo fiscale degli asset ricevuti da settlement e rilevanza della successiva disposizione dell'asset, anche per USDT/USDC.
6. Minusvalenze c-quater e relazione con gli altri redditi finanziari, non con c-sexies; dichiarazione per singolo anno.
7. Monitoraggio del rapporto estero con derivati, margini crypto, eventuale IVAFE/IC e prevenzione di doppie valorizzazioni.
8. Valutazione in EUR: timestamp rilevante, fonte prezzo, conversioni, regole per prezzi mancanti/contestati.

Fino alla validazione, la GUI deve segnalare `TRATTAMENTO_FISCALE_NON_CONFERMATO` e non presentare il risultato fiscale come pronto per la dichiarazione.

## 7. Requisiti di test e regressione

### Test deterministici minimi
- saldo iniziale 2.000 USDT; acquisto BTC per 1.000; trasferimento altri 1.000 al Futures; chiusure +400 o -400; ricostruzione coerente dei saldi e reddito c-quater;
- funding +/-, apertura con fee e PnL 0, fee in token diverso, liquidation, rimborso fee, funding già incluso nel PnL;
- stesso trade descritto in entrambi gli export, multi-ordini, timestamp non identici, migrazione fra formati e righe annullate;
- stessa valuta su due sotto-account, LIFO globale e `PlusXWallet`, passaggi fra gruppi diversi, costi storici e saldi negativi;
- ricalcolo prima/dopo modifica classificazione di movimento derivato, data, cambio, valore, gruppo, documento fonte, cancellazione e annullamento;
- inizio/fine anno locale con offset e ora legale, cambio storico USDT/EUR, numeri decimali molto piccoli;
- rapporto W/RW, RT c-sexies, nuova sezione c-quater: totali distinti; report e backup-ripristino byte/equivalenza;
- archive senza derivati: valori e ordinamenti invariati, golden master invariati.

### Suite esistenti da riutilizzare ed estendere
- `DerivatiTest`, `ImportazioneGenericaAvvisoDerivatiTest`, `ImportazioneGenericaBybitTest`;
- `CalcoliPlusvalenzeNewEquivalenzaIncrementaleTest`, `CalcoliPlusvalenzeNewGoldenMasterTest`, `Calcoli_RW_GoldenMasterTest`, `Calcoli_RW_GiacenzeTest`;
- `CommissioniCollegateTest`, `GruppoOperazioneTest`, `Principale_Movimenti_SeparaUnisciTest`, `GUI_ModificaMovimentoSoloPrezzoTest`;
- `MovimentiStoricoTest`, `DocumentiFonteTest`, `ScartiImportTest`, `Backup_RestoreTest`, `Backup_CompatibilitaTest`, `DichiarazioniTest`.

I golden master su dataset reale leggono la cartella locale `nocommit/GoldenMaster`; **non risultano eseguibili integralmente dal solo repository pubblico**. Occorre predisporre fixture anonime/minime versionate in Git e una baseline riproducibile senza dati personali.

### Validazione prestazionale
Registrare: tempo import CSV/API, riconciliazione, memoria di picco, ricalcolo completo/incrementale, aggiornamento UI, generazione report. Confronto pre/post a parità di fixture, su dataset senza derivati e con molti funding. Non assumere che il solo kernel sia il collo di bottiglia.

## 8. Roadmap documentale e sequenza consigliata

**Gate 0 — Chiudere i punti di audit:** censimento dei callback dell'enorme `Principale.java` e delle schermate `.form`, verifica delle tabelle via H2 runtime, build/test, baseline temporali e prestazionali; matrice touchpoint/casi d'uso. L'audit attuale è statico e non prova l'esecuzione.

**Gate 1 — PRD Master:** perimetro prodotti, scenari utente, matrice end-to-end, tipi di evento, stati errore, comportamenti di modifica, output e criteri di accettazione; backlog questioni fiscali irrisolte.

**Gate 2 — ADR / System Architecture Spec:** proprietà delle informazioni, schema identità evento, raw record vs movimento vs proiezione, account, integrità e transazioni, compatibilità storage, API dei motori; valutazione di mantenere `String[46]` vs sidecar H2 senza duplicare il ledger.

**Gate 3 — Specifiche tecniche modulari:** (a) ingest/reconciliation, (b) account/transfer/saldo, (c) crypto settlement bridge e LIFO, (d) derivative tax kernel EUR, (e) UI e editing, (f) report e backup, (g) test/performance.

**Gate 4 — Piani e implementazione per vertical slice:** ogni fase deve arrivare almeno a prova importer → saldo → ricalcolo → UI → backup → report di controllo. Il report fiscale dichiarativo richiede prima le decisioni normative del §6.

**Gate 5 — Release acceptance:** green suite e benchmark, nessuna differenza regressiva non approvata, quadratura per exchange e account, riconciliazione documentale, ricalcolo riproducibile, report coerente, archivio ripristinabile.

## 9. Limiti dell'audit e conclusione

**Verificato:** il modello a 46 campi; marcatori 43/44/45; collegamento commissioni e operazioni; API principali di import/generazione/persistenza; kernel LIFO, RT, RW e giacenze; documenti, storico, backup, dichiarazioni; suite test esistenti; toolchain Java 21 / Maven / H2 / Swing / OpenPDF / FastExcel (`pom.xml`).

**Non verificato:** esecuzione runtime; test Maven; benchmark; percorso completo degli handler del monolitico `Principale.java` (file molto grande e non letto integralmente in questa analisi); codice di tutte le finestre `.form`; correttezza fiscale degli scenari ancora controversi; supporto reale di ciascun exchange; compatibilità su DB di utenti.

**Conclusione:** non sono necessarie, sulla base della sola analisi statica, due contabilità patrimoniali o un secondo LIFO indipendente. Occorre invece (1) un'identità economica canonica idempotente e riconciliata, (2) posting patrimoniale unico con account operativi, (3) proiezione fiscale dei derivati distinta e regole per settlement dell'asset, (4) integrazione coerente di tutti i motori e dei workflow esistenti. Il PRD dovrà fissare prima la copertura e i contratti, e la System Architecture Spec stabilirà il minimo cambiamento sicuro alla codebase.
