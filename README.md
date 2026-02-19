## 1) Problema individuato

## 1. Descrizione
`check_network.sh` è uno script bash progettato per garantire la continuità operativa della cassa in presenza di problemi di rete.  
Lo script verifica la raggiungibilità del server centrale e la qualità della connessione, passando automaticamente la cassa in **modalità offline** se necessario e registrando ogni evento nel file di log `cassa.log`.

**Obiettivi principali:**
- Prevenire blocchi del sistema causati da rete lenta o server irraggiungibile.  
- Fornire alert immediati al personale tecnico.  
- Tenere un registro storico degli eventi per diagnosi e analisi.

---

## 2. Problema Individuato
- **Instabilità della connessione:** il server può essere irraggiungibile o rispondere lentamente (>200 ms).  
- **Conseguenze:**  
  - Blocco temporaneo della cassa, rallentando le operazioni.  
  - Possibile perdita o corruzione dei dati di vendita.  
  - Difficoltà nell’identificare se il problema è server down o semplicemente lentezza della rete.

---

## 3. Funzionamento dello Script
- **Preparazione log:** crea `cassa.log` se non esiste (`touch "$LOG_FILE"`).  
- **Ping al server:** invia un pacchetto ICMP (`ping -c 1 -W 2 $SERVER_IP`). Mancata risposta identifica il server come irraggiungibile.  
- **Verifica latenza:** se il server risponde, estrae la latenza e la confronta con la soglia (`SOGLIA_MS=200`). Latenza superiore forza modalità offline e genera un warning.  
- **Segnalazioni visive:** messaggi critici o warning stampati a video per informare il personale.  
- **Riepilogo log:** conteggia gli errori storici nel file di log, distinguendo tra server non raggiungibile e connessione lenta.

---

## 4. Configurazione
SERVER_IP="8.8.8.8" # IP del server da controllare
LOG_FILE="cassa.log" # File log
SOGLIA_MS=200 # Latenza massima (ms)
CASSA_ID="CASSA_01" # ID univoco della cassa

## 5. Esecuzione
# Esecuzione standard (usa IP e soglia di default)
./check_network.sh

# Esecuzione con IP e soglia personalizzati
# Esempio: server locale con soglia 100ms
./check_network.sh 192.168.1.50 100

---




## 2) Problema individuato

## 1. Descrizione
`svuota_buffer.sh` è uno script bash progettato per garantire la **sicurezza dei dati di vendita** quando la cassa opera offline.  
Durante l’assenza di connessione, le vendite vengono salvate nel file `vendite_buffer.csv`. Lo script gestisce la sincronizzazione con il server centrale evitando perdite o corruzioni di dati.

**Obiettivi principali:**
- Proteggere i dati fiscali fino a conferma server.  
- Evitare svuotamenti prematuri del buffer.  
- Fornire alert immediati in caso di errori.  
- Tenere traccia di ogni operazione nel file di log `cassa.log`.

---

## 2. Problema Individuato
- **Buffer locale:** le vendite offline sono salvate in `vendite_buffer.csv`.  
- **Rischi:**  
  - Perdita dati se il buffer viene rimosso prematuramente senza conferma.  
  - Database non allineati tra vendite locali e server.  
  - Errori di lettura se il file manca o non ha intestazione.

---

## 3. Funzionamento dello Script
- **Verifica file:** controlla che `vendite_buffer.csv` esista prima di procedere (`if [ ! -f "$BUFFER" ]; then exit 1; fi`).  
- **Conteggio record:** analizza il numero di transazioni oltre l’intestazione (`wc -l`) e procede solo se ci sono dati.  
- **Handshake server:** la cancellazione avviene solo se il server risponde "OK". Se riceve "FAIL", i dati restano protetti e viene generato un alert.  
- **Pulizia selettiva:** se confermato, mantiene l’intestazione CSV (`sed -i '1!d' "$BUFFER"`).  
- **Logging:** ogni operazione è registrata in `cassa.log` con timestamp, stato e dettagli.

---

## 4. Configurazione
BUFFER="vendite_buffer.csv" # File temporaneo vendite offline  
LOG_FILE="cassa.log"        # File log  
CASSA_ID="CASSA_01"        # ID univoco della cassa  

## 5. Esecuzione
# Rendi eseguibile lo script  
chmod +x svuota_buffer.sh  

# Esecuzione standard  
./svuota_buffer.sh

---

## 3) Problema individuato

## 1. Descrizione
`genera_scontrino.sh` è uno script bash progettato per generare **scontrini leggibili** quando la cassa opera offline.  
Durante la modalità offline, le vendite sono salvate nel file `vendite_buffer.csv` utilizzando codici prodotto tecnici (es. P001). Lo script trasforma questi codici in nomi commerciali, calcola i totali e produce un documento chiaro, pronto per essere stampato o visualizzato a terminale.

**Obiettivi principali:**
- Trasformare codici tecnici in nomi prodotti leggibili.  
- Ricalcolare correttamente i totali di ogni scontrino.  
- Garantire trasparenza e continuità del servizio anche senza connessione.  
- Registrare l’evento nel log `cassa.log` per tracciabilità.

---

## 2. Problema Individuato
- **Buffer tecnico:** le vendite offline contengono solo codici prodotto.  
- **Rischi:**  
  - Mancanza di trasparenza: scontrini illeggibili per clienti e personale.  
  - Difficoltà di verifica: impossibile risalire rapidamente al nome del prodotto venduto.  
  - Errori di calcolo: totale scontrino potrebbe non corrispondere ai dati grezzi del buffer.

---

## 3. Funzionamento dello Script
- **Identificazione transazione:** estrae l’ultimo ID scontrino dal buffer usando `tail` e `cut`.  
- **Data matching locale:** converte codici prodotto in nomi commerciali tramite `prodotti_default.csv`.  
- **Ricalcolo totale con AWK:** somma i totali riga per garantire coerenza matematica.  
- **Formattazione professionale:** allinea nomi prodotti e prezzi a terminale usando `printf`.  
- **Tracciabilità (Logging):** registra l’evento in `cassa.log` con tag `PRINT_RECEIPT`.

---

## 4. Configurazione
BUFFER="vendite_buffer.csv"    # File buffer vendite offline  
PRODOTTI="prodotti_default.csv" # File anagrafica prodotti  
LOG_FILE="cassa.log"           # File log  
CASSA_ID="CASSA_01"           # ID univoco della cassa  

---

## 5. Esecuzione
# Rendi eseguibile lo script  
chmod +x genera_scontrino.sh  

# Esecuzione standard  
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





