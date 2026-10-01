# SmartPump - Specifiche di Progetto e Linee Guida per lo Sviluppo

Questo documento definisce l'architettura tecnica, i requisiti funzionali e le scelte implementative per **SmartPump**, un'applicazione Android **Local-First** sviluppata in **Java** che permette di individuare i distributori di carburante più economici nelle vicinanze sfruttando i dati Open Data ufficiali del MIMIT (Ministero delle Imprese e del Made in Italy).

---

## 1. Panoramica e Obiettivi dell'App
* **Core Business:** Trovare la pompa di benzina più economica nel raggio di pochi chilometri dalla posizione GPS corrente dell'utente.
* **Architettura Local-First:** Zero dipendenze da server o database cloud di terze parti. L'applicazione scarica i dati grezzi direttamente dalle fonti istituzionali e li elabora interamente sul dispositivo Android tramite un database SQLite locale.
* **Target Iniziale:** Territorio italiano (estendibile in futuro).
* **Linguaggio:** Java.

---

## 2. Stack Tecnologico Consigliato
* **Linguaggio:** Java 11+
* **UI:** XML Layouts / Material Design 3 (o Jetpack Compose se preferito, ma l'architettura base è ottimizzata per approcci standard robusti).
* **Database Locale:** **Room Persistence Library** (SQLite).
* **Gestione Background Tasks:** **WorkManager** (per il download e l'aggiornamento giornaliero dei file CSV).
* **Geolocalizzazione:** `FusedLocationProviderClient` (Google Play Services).
* **Networking (Download CSV):** OkHttp o Retrofit.

---

## 3. Gestione dei Dati (Open Data MIMIT)
Il sistema si alimenta tramite due flussi CSV giornalieri pubblicati dal Ministero:
1. **Anagrafica Impianti (`anagrafica_impianti_attivi.csv`):**
   * Contiene l'ID impianto, bandiera (marchio), nome, indirizzo, comune, provincia, latitudine e longitudine.
2. **Prezzi Praticati (`prezzo_alle_8_di_mattina.csv`):**
   * Contiene l'ID impianto, il tipo di carburante (Benzina, Gasolio, GPL, Metano), la modalità (Self o Servito), il prezzo e la data di aggiornamento.

---

## 4. Schema del Database Locale (Room)

Il database deve essere composto da due entità relazionate:

### A. Entità `StationEntity` (Tabella `stations`)
* `id_impianto` (LONG, Primary Key)
* `bandiera` (STRING)
* `nome` (STRING)
* `indirizzo` (STRING)
* `comune` (STRING)
* `latitudine` (DOUBLE)
* `longitudine` (DOUBLE)

### B. Entità `PriceEntity` (Tabella `prices`)
* `id` (LONG, Primary Key, AutoGenerate)
* `id_impianto` (LONG, Foreign Key -> `stations.id_impianto`)
* `tipo_carburante` (STRING: es. "Benzina", "Gasolio")
* `is_self` (BOOLEAN)
* `prezzo` (DOUBLE)
* `data_aggiornamento` (STRING)

---

## 5. Logica di Ricerca Spaziale (Bounding Box + Haversine)

Per evitare calcoli pesanti su tutti i 25.000 distributori italiani ad ogni movimento dell'utente, la ricerca spaziale deve avvenire in due step:

### Step 1: Filtro Preliminare (Bounding Box SQL)
Prima di calcolare la distanza esatta, si estraggono solo le stazioni all'interno di un rettangolo geografico circostante la posizione GPS del telefono ($\text{lat}, \text{lon}$) con un margine di $\pm \Delta$ (es. circa $0.05$ gradi corrispondono a $\approx 5\text{ km}$):

$$\text{lat}_{\min} = \text{lat} - 0.05, \quad \text{lat}_{\max} = \text{lat} + 0.05$$
$$\text{lon}_{\min} = \text{lon} - 0.05, \quad \text{lon}_{\max} = \text{lon} + 0.05$$

### Step 2: Calcolo di Haversine di Precisione
Sulle poche decine di stazioni filtrate nel bounding box, si calcola la distanza reale in metri utilizzando la formula di Haversine:

$$d = 2R \arcsin\left(\sqrt{\sin^2\left(\frac{\Delta \phi}{2}\right) + \cos(\phi_1)\cos(\phi_2)\sin^2\left(\frac{\Delta \lambda}{2}\right)}\right)$$

Dove $R$ è il raggio medio della Terra ($\approx 6371\text{ km}$).

---

## 6. Flusso Operativo dell'App (Workflow)

1. **Primo Avvio:** L'app controlla se il database locale è vuoto. Se vuoto, scarica i file CSV del MIMIT in background, esegue il parsing riga per riga (gestito tramite Coroutines o AsyncTask/Executor in Java) e popola le tabelle Room.
2. **Aggiornamento Quotidiano:** Un `Worker` pianificato tramite `WorkManager` (eseguito preferibilmente sotto Wi-Fi una volta al giorno) scarica l'ultimo file dei prezzi e aggiorna la tabella corrispondente.
3. **Schermata Principale (Home):**
   * Richiede i permessi di localizzazione.
   * Ottiene la posizione corrente tramite `FusedLocationProviderClient`.
   * Esegue la query DAO filtrando per raggio (es. 5 km), tipo di carburante (es. Benzina) e modalità (Self).
   * Mostra i risultati ordinati per prezzo crescente (dal più economico).
4. **Navigazione:** Al click su un elemento della lista, l'app lancia un `Intent` verso Google Maps o Waze passando le coordinate geografiche della pompa selezionata.

---

## 7. Istruzioni per Gemini in Android Studio
* Scrivi codice modulare, pulito e rigorosamente in **Java**.
* Evita memory leak associando correttamente il ciclo di vita dei LifecycleOwner ai LiveData o ai Flow/Callbacks di Room.
* Gestisci sempre i thread secondari (Background Threads) per le operazioni di I/O sul database e il parsing dei CSV, evitando blocchi della UI (ANR).