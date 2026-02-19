# 1) Analisi e resilienza della connessione (`check_network.sh`)

Nel sistema di casse analizzato, la **connessione tra cassa e server deve essere stabile**.  
Instabilità o latenze elevate possono causare:

- Blocco temporaneo della cassa  
- Perdita o corruzione dei dati di vendita  
- Difficoltà nel distinguere tra server down e lentezza di rete  

In ambienti con funzionamento **offline/online intermittente**, una connessione instabile può:
- Far passare la cassa in modalità offline senza preavviso  
- Ritardare le transazioni o generare errori nei log  
- Compromettere la continuità operativa del sistema

---

## Motivazione del controllo
Il monitoraggio della rete è stato introdotto perché:
- La continuità operativa dipende dalla connessione stabile  
- I log e le vendite devono essere registrati correttamente  
- Allertare immediatamente il personale tecnico riduce i rischi di interruzione  

Il controllo della latenza e della raggiungibilità consente:
- Prevenire blocchi del sistema  
- Registrare in modo accurato gli eventi nel log  
- Fornire alert chiari in caso di problemi

---

## Funzionamento dello script
Lo script verifica la **connessione della cassa al server centrale** e gestisce automaticamente la modalità offline se necessario.  

In particolare:
- Crea `cassa.log` se non esiste  
- Invia un ping al server (`ping -c 1 -W 2 $SERVER_IP`) per verificarne la raggiungibilità  
- Estrae la latenza e la confronta con la soglia configurabile (`SOGLIA_MS`)  
- Se la latenza è troppo alta o il server non risponde:  
  - La cassa passa in modalità offline  
  - Viene generato un messaggio di warning a video  
  - L’evento viene registrato nel log con dettagli  
- Riassume gli errori storici nel log, distinguendo tra server irraggiungibile e connessione lenta

---

## Configurazione
SERVER_IP="8.8.8.8" # IP del server da controllare
LOG_FILE="cassa.log" # File log
SOGLIA_MS=200 # Latenza massima (ms)
CASSA_ID="CASSA_01" # ID univoco della cassa

---

##  Esecuzione
Esecuzione standard (usa IP e soglia di default)
./check_network.sh
Esecuzione con IP e soglia personalizzati
Esempio: server locale con soglia 100ms
./check_network.sh 192.168.1.50 100

# 2) Svuotamento sicuro del buffer vendite (`svuota_buffer.sh`)

Nel sistema di casse analizzato, **le vendite registrate offline devono essere salvaguardate**.  
Un buffer locale non gestito correttamente può causare:

- Perdita di dati di vendita  
- Incoerenze tra database locale e server  
- Errori nella ricostruzione dei log  
- Problemi durante la sincronizzazione offline → online  

In ambienti con funzionamento **offline/online intermittente**, la gestione del buffer è fondamentale per:
- Proteggere i dati fiscali  
- Garantire la coerenza dei report  
- Evitare svuotamenti prematuri del buffer

---

## Motivazione del controllo
La gestione del buffer è stata introdotta perché:
- Le vendite offline devono essere conservate fino alla conferma server  
- Gli errori nella cancellazione prematura del buffer possono corrompere i dati  
- La tracciabilità richiede log affidabili e completi  

Il controllo consente:
- Protezione dei dati fiscali  
- Allineamento costante tra cassa e server  
- Alert immediati in caso di problemi

---

## Funzionamento dello script
Lo script gestisce il **buffer delle vendite offline** salvato in `vendite_buffer.csv`.  

In particolare:
- Verifica l’esistenza del file buffer prima di procedere  
- Conta il numero di transazioni presenti (oltre l’intestazione)  
- Cancella il buffer solo se il server conferma la ricezione con "OK"  
- Mantiene l’intestazione CSV per preservare il formato  
- Registra ogni operazione in `cassa.log` con timestamp, stato e dettagli  

---

## Configurazione
BUFFER="vendite_buffer.csv" # File temporaneo vendite offline  
LOG_FILE="cassa.log"        # File log  
CASSA_ID="CASSA_01"        # ID univoco della cassa  

---

## Esecuzione
Rendi eseguibile lo script  
chmod +x svuota_buffer.sh  

Esecuzione standard  
./svuota_buffer.sh

---

# 3) Ricostruzione scontrini quando il server non è raggiungibile (`genera_scontrino.sh`)

Nel sistema di casse analizzato, **le vendite offline devono essere presentate in modo leggibile**.  
Un buffer tecnico con soli codici prodotto può causare:

- Scontrini illeggibili per clienti e personale  
- Difficoltà nel verificare rapidamente i prodotti venduti  
- Errori di calcolo nei totali degli scontrini  

In ambienti con funzionamento **offline/online intermittente**, la generazione corretta degli scontrini è fondamentale per:
- Garantire trasparenza nei confronti del cliente  
- Assicurare la coerenza dei dati fiscali  
- Mantenere la tracciabilità delle vendite

---

## Motivazione del controllo
La generazione degli scontrini offline è stata introdotta perché:
- I codici tecnici da soli non sono comprensibili  
- I totali devono corrispondere ai dati grezzi del buffer  
- La tracciabilità richiede registrazioni leggibili e complete  

Il controllo consente:
- Trasformare codici prodotto in nomi commerciali  
- Ricalcolare correttamente i totali  
- Registrare ogni evento nel log con tracciabilità chiara

---

## Funzionamento dello script
Lo script trasforma le vendite offline in **scontrini leggibili**.  

In particolare:
- Estrae l’ultimo ID scontrino dal buffer (`tail` + `cut`)  
- Converte codici prodotto in nomi commerciali tramite `prodotti_default.csv`  
- Ricalcola il totale dello scontrino con `awk`  
- Allinea nomi prodotti e prezzi a terminale usando `printf`  
- Registra l’evento in `cassa.log` con tag `PRINT_RECEIPT`  

---

##  Configurazione
BUFFER="vendite_buffer.csv"    # File buffer vendite offline  
PRODOTTI="prodotti_default.csv" # File anagrafica prodotti  
LOG_FILE="cassa.log"           # File log  
CASSA_ID="CASSA_01"           # ID univoco della cassa  

---

##  Esecuzione
Rendi eseguibile lo script  
chmod +x genera_scontrino.sh  

Esecuzione standard  
./genera_scontrino.sh  

**Output:** lo scontrino viene stampato a terminale e l’evento registrato in `cassa.log`.


## 7) Problema individuato

**Problema:** duplicazione delle vendite nei file `vendite_buffer.csv`.  
Quando una cassa è offline, tutte le vendite vengono salvate localmente in `vendite_buffer.csv`.

Se lo stesso file viene inviato più volte al server oppure contiene righe duplicate interne, si rischia di creare **record doppi nel database centrale**.

### Può causare

- **Errori contabili:** i totali delle vendite sarebbero errati.  
- **Incoerenza dei dati:** il server avrebbe più righe della stessa vendita, rendendo difficile il tracciamento.  
- **Problemi di performance:** dati duplicati aumentano inutilmente il carico del database.

### Rilevanza nel contesto

- Nel supermercato, la **continuità operativa** e l’**integrità dei dati** sono fondamentali.  
- Evitare duplicazioni garantisce:  
  - accuratezza delle vendite,  
  - corretto aggiornamento del magazzino,  
  - fiducia nei dati di analisi.  
- La **sicurezza dei dati** viene indirettamente tutelata: duplicati potrebbero anche mascherare errori reali o malfunzionamenti del sistema di sincronizzazione.

### Sintesi dello script

Lo script gestisce l'invio dei file `vendite_buffer.csv` al server centrale, prevenendo duplicazioni e garantendo l’integrità dei dati.

#### Funzionalità principali
- Controlla che il file `vendite_buffer.csv` esista e sia corretto.
- Rileva eventuali **duplicati interni** al file e li elimina, evitando che lo stesso record venga inviato più volte.
- Verifica nel **log della cassa** se il file è già stato inviato al server; se sì, blocca un nuovo invio.
- Invia il file al server tramite `curl` se non è già stato inviato.
- Aggiorna il log locale con esito dell’operazione, così da tracciare tutte le azioni.

#### Come usare lo script
```bash
./ex7.sh vendite_buffer_12345.csv
```
---

## 8) Problema individuato

Durante l’avvio del sistema di cassa possono verificarsi condizioni anomale critiche che, se non rilevate subito, compromettono:  

- Il corretto funzionamento del software di cassa  
- La continuità operativa  
- La sicurezza e l’integrità dei dati  

### Problemi principali individuati

1. **Interfaccia di loopback (lo) non attiva**  
   L’interfaccia di loopback è fondamentale per:  
   - La comunicazione tra processi locali  
   - Il corretto funzionamento di servizi interni (database locali, servizi di sincronizzazione, logging)  

   Se lo è DOWN:  
   - Alcuni servizi possono fallire silenziosamente  
   - La cassa può sembrare operativa ma produrre dati inconsistenti  
   - I meccanismi di sincronizzazione con il server possono non funzionare correttamente  

2. **Crescita incontrollata del file di log**  
   Il file `log_cassa.log` cresce continuamente registrando:  
   - Vendite  
   - Errori  
   - Sincronizzazioni  
   - Eventi di rete  

   Se non gestito:  
   - Può raggiungere dimensioni elevate  
   - Saturare lo spazio disco (la cassa ha memoria limitata)  
   - Portare al blocco delle vendite o del sistema operativo  

### Motivazione della scelta

Intercettare all’avvio:  
- Il problema del loopback  
- La dimensione del file di log  

È strategico perché:  
- L’avvio è il momento migliore per prevenire errori prima che la cassa inizi a vendere  
- Evita situazioni di degrado durante l’orario di apertura  
- Riduce il rischio di perdita o corruzione dei dati  

La gestione preventiva è più sicura ed efficace rispetto a un intervento reattivo.

## Cosa fa lo script (descrizione dettagliata)

Lo script `ex8.sh` viene eseguito all’avvio della cassa (tramite cronjob `@reboot ../cassa/ex8.sh`) e serve a prevenire condizioni critiche durante l’avvio del sistema.

### Funzionalità principali
- Controlla che la cassa parta in condizioni sicure verificando:
  - L’interfaccia **loopback (lo)** sia attiva.
  - La dimensione del log non superi la soglia di sicurezza (50 MB).
- Se il log è troppo grande:
  - Lo comprime in un archivio ZIP con timestamp.
  - Svuota il file originale senza modificare i permessi.
- Genera messaggi di errore e alert verso il sistemista in caso di anomalie.

---

## 9) Problema individuato

Nel sistema di casse analizzato, il **server centrale** può diventare temporaneamente non raggiungibile.

### Motivazione della scelta

Sapere quando e per quanto tempo il server non è stato disponibile è fondamentale per:
- Valutare SLA
- Individuare problemi di rete o infrastruttura
- Migliorare la resilienza del sistema
- Generare report affidabili sul funzionamento del sistema
- Confrontare periodi offline con i dati di vendita
- Individuare problemi di rete o anomalie di sincronizzazione

### Cosa fa lo script
Analizza `log_cassa.log` calcola:
- Data del giorno considerato
- Ora di inizio del periodo offline
- Ora di ripristino della connessione
- Durata totale in secondi del periodo di indisponibilità

Questo permette di avere un **report preciso e strutturato** dei momenti di mancata connessione, utile per analisi statistiche, estrazioni dati e controllo della continuità operativa.

Lo script gestisce anche:

- Controllo dei parametri in ingresso
- Validazione del formato della data
- Verifica dell’esistenza del file di log
- Creazione di un file di output strutturato con intestazione
- Evita di generare dati duplicati per lo stesso giorno
- Scrive i risultati in un file CSV-like (`status_offline.log`) pronto per estrazioni o report

### Come usare lo Script
```bash
./ex9.sh YYYY-MM-DD LOG_CASSA
```
- Lo script richiede due parametri:
  1. Il giorno da analizzare, nel formato `YYYY-MM-DD`
  2. Il file di log della cassa da cui estrarre i dati

- L’output viene scritto in `./log/status_offline.log`.  
  Se il file non esiste o è vuoto, viene creato con intestazione chiara.


## 10) Problema individuato

Nel sistema di casse analizzato, **cassa e server devono avere orari coerenti**.  
Una differenza significativa di tempo può causare:

- Errori nella ricostruzione delle vendite  
- Problemi di sincronizzazione dei file CSV  
- Incoerenze nei log di sicurezza  
- Difficoltà nell’analisi forense e negli audit  

In ambienti con funzionamento **offline/online intermittente**, una cassa con orario errato può:
- Registrare vendite con timestamp non attendibili  
- Inviare dati che il server interpreta come duplicati o fuori sequenza  

### Motivazione della scelta

Il controllo dell’orario è stato introdotto perché:
- I log e le vendite si basano sul timestamp  
- La sicurezza e la tracciabilità richiedono tempi affidabili 

Differenze di orario compromettono:
- L’ordine cronologico delle vendite  
- La corretta sincronizzazione offline → online  
- L’affidabilità dei report  

Questo controllo migliora:
- L’integrità dei dati  
- La coerenza dei log  
- La sicurezza operativa complessiva del sistema

L’uso di una **soglia di tolleranza configurabile** consente:
- Flessibilità operativa  
- Adattamento a contesti reali (latenza di rete, micro-scarti temporali)  

### Cosa fa lo script
Lo script verifica che l’orario della **cassa** e quello del **server centrale** siano coerenti, condizione fondamentale per garantire l’affidabilità dei log e dei dati di vendita.

In particolare:
- Confronta l’orario della cassa con quello del server
- Valida l’orario del server confrontandolo con una fonte esterna affidabile
- Calcola la differenza temporale tra cassa e server
- Verifica che tale differenza rientri in una soglia di tolleranza configurabile
- In caso di scarto eccessivo, aggiorna automaticamente l’orario della cassa
- Fornisce un riscontro chiaro sugli orari finali di cassa e server

### Come usare lo script
```bash
./ex10.sh 5
```
- Lo script richiede **un solo parametro**: il numero di secondi di tolleranza ammessi tra cassa e server.
- Se la differenza rientra nella soglia, il sistema viene considerato sincronizzato.
- Se la differenza supera la soglia, l’orario della cassa viene corretto automaticamente.
- Lo script è pensato per essere eseguito manualmente dal sistemista o integrato in procedure di controllo periodico.





