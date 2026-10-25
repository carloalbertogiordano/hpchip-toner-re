# Reverse engineering chip toner HP — log di lavoro

## Obiettivo

Analizzare il chip di memoria ("E-label", terminologia interna HP) estratto da una
cartuccia toner HP, per capire:
- se è flashabile/riprogrammabile
- come conserva i dati
- che tipo di dati contiene
- come è costruito fisicamente

Obiettivo pratico secondario: capire se è possibile resettare il contatore di
utilizzo/livello toner memorizzato sul chip **originale** (quello che funziona
davvero sulla stampante).

Lavoro autorizzato e coperto da accordo con l'università (esame di hardware
penetration testing).

## Hardware coinvolto

**Stampante (non disponibile fisicamente durante il lavoro):**
HP LaserJet MFP M234dw / M234dwe / M234sdne / M234sdw (famiglia M209/M234/M236)

**Cartuccia toner:** HP 135A / 135X (W1350A / W1350X)

**Chip originale:**
- Estratto dalla plastica strutturale della cartuccia (non da una "goccia" protettiva
  separata — era annegato nel corpo cartuccia stesso)
- Package: chip-on-board (COB) con glob-top epossidico nero, PCB grande quanto il
  chip stesso — non si stacca con aria calda (incollato/wire-bondato, non saldato)
- **2 soli contatti elettrici esposti**
- Marking illeggibile/parzialmente abraso — letture tentate: "Y0C1", "YC04" —
  nessuna corrispondenza trovata in database pubblici (marking custom/proprietario)

**Chip clone (compatibile, non funzionante sulla stampante anche prima dei test):**
- Marking: "2515 9V8HK" (non identificato in ricerche pubbliche)
- Stesso carrier/dimensioni dell'originale, ma con **4 pad totali**: 2 grandi
  (stessa posizione dei 2 pad dell'originale) + 2 piccoli (aggiunti solo su questo
  esemplare)
- Colori fili usati per riferimento: **Giallo, Rosso** (pad piccoli) — **Verde,
  Arancio** (pad grandi)
- Produttore probabile: **Apex Microelectronics** (Zhuhai, Cina) — maggior
  produttore mondiale di chip toner per mercato aftermarket

**Strumentazione:**
- Arduino UNO R4 WiFi (`/dev/ttyACM0`, FQBN `arduino:renesas_uno:unor4wifi`)
- Multimetro (modalità diodo, resistenza, DC volt)
- CH341A (posseduto, non ancora utilizzato utilmente — vedi note protocollo)
- Resistori disponibili: solo 470Ω (usati anche in parallelo/serie per ottenere
  altri valori)

## Metodologia e test eseguiti

### 1. Identificazione fisica
- Conteggio pad: confermati 2 sull'originale, 4 sul clone (2 grandi + 2 piccoli)
- Tentata lettura marking chip: inconcludente su entrambi
- Package identificato come COB/glob-top sull'originale — **decapsulazione
  scartata** (richiede attrezzatura failure-analysis da laboratorio, troppo
  distruttiva/rischiosa per questo contesto)

### 2. Test multimetro (modalità diodo) — identificazione pinout
Eseguita mappatura sistematica a coppie (modalità diodo) su entrambi i chip.

**Originale (2 pad):** letture bidirezionali che conducono in entrambe le
direzioni con valori simili (~1076-1280mV) — non firma di diodo singolo pulito,
compatibile con rete di protezione ESD interna (diodi verso VDD interno e verso
GND). Conclusione: **vero circuito integrato**, non elemento resistivo/fusibile
passivo (confermato anche da: nessuna resistenza misurabile in modalità Ω, dove
un diodo reale correttamente non conduce per tensione di test troppo bassa).

**Clone (4 pad):** mappatura 12 letture su tutte le coppie ha rivelato pattern:
- **Orange** → conduce verso tutti gli altri in modo stabile (~1090-1100mV) →
  ipotesi **GND/VSS**
- Tutti gli altri → **Green**: stabile (~1090-1180mV) → ipotesi **VDD-like /
  DATA con carica parassita**
- **Yellow, Red**: letture più alte e variabili/derivanti nel tempo → probabili
  **pin segnale puro** (non rail)

### 3. Test elettrico attivo — Arduino (OneWire + probe raw)

Due sketch creati:
- `toner_onewire_scan.ino` — usa libreria OneWire standard, cerca ROM ID
  (family code) via `ds.search()`
- `toner_raw_probe.ino` — bypassa la libreria, invia impulsi di reset di durata
  variabile (60-1000µs) e campiona la linea a grana fine (~1µs) per 700µs cercando
  qualunque attività (indipendente da timing standard 1-Wire)

**Setup:** pull-up esterno (mai diretto a 5V per sicurezza, sempre 3.3V) +
resistore serie di protezione su D2, montati su breadboard.

**Risultati — chip originale:**
- Nessuna risposta OneWire in nessuna polarità dei 2 fili
- Raw probe: nessuna attività rilevata a nessuna durata di reset testata
- **Comportamento elettrico normale**: la linea sale regolarmente a livello
  alto sotto pull-up (transitions=1, rampa RC pulita, poi stabile HIGH) —
  nessuna anomalia, semplicemente nessuna risposta al protocollo/timing tentato

**Risultati — chip clone (pad grandi Green/Orange):**
- Nessuna risposta OneWire
- Raw probe: nessuna attività
- **Anomalia elettrica**: la linea non supera mai ~0.6-0.7V, indipendentemente da:
  - forza del pull-up (provato da 470Ω a resistori multipli in parallelo, fino
    a ~117Ω equivalenti, corrente fino a ~28mA)
  - polarità (scambio Green/Orange non cambia il risultato)
  - stato dei pad piccoli (aggiunto pull-down 470Ω su Yellow/Red per escludere
    interferenza da pin fluttuanti — nessun cambiamento)
- La quasi-costanza della tensione (0.6→0.7V) al variare della corrente è
  **firma tipica di un diodo reale in conduzione**, non di un ingresso ad alta
  impedenza che semplicemente "non risponde"

**Interpretazione conclusiva sul clone:** dato che l'utente aveva già osservato
che questo clone specifico non funzionava sulla stampante reale (prima di
qualunque test di laboratorio), l'ipotesi più semplice coerente con tutti i dati
è che **questo esemplare di clone sia elettricamente danneggiato** (diodo di
protezione interno guasto/permanentemente in conduzione — guasto comune da ESD
o sovratensione), non un mistero di protocollo. Il comportamento elettrico
"pulito" dell'originale (nessun blocco anomalo di tensione) supporta questa
distinzione.

### 4. Tentativo identificazione pinout ICSP sul clone

Ipotesi iniziale: pad piccoli (Yellow/Red) = ICSP PIC (PGC/PGD), chip target
PIC12F683 (menzionato in un forum come chip usato in moduli simili).

**Verifica specifica trovata (documentazione ufficiale Microchip):** l'ingresso
in modalità Low-Voltage Programming per PIC12F683 richiede **MCLR portato a
livello basso** oltre a VDD, VSS, PGC, PGD — cioè **5 collegamenti**, non 4.

**Conclusione:** con solo 4 pad totali disponibili sul clone, l'ICSP PIC
standard (almeno nello schema documentato per 12F683) **non è applicabile
direttamente** — o manca un punto di accesso (MCLR non esposto), o il chip non
è realmente un PIC12F683, o usa uno schema di entry diverso non richiedente
MCLR dedicato.

**Candidati alternativi per interfaccia a 4 fili (VDD/GND + 2 segnali, senza
reset dedicato) non ancora testati:**
- I2C debug bus (VDD, VSS, SDA, SCL)
- SWD — se il clone monta un MCU ARM Cortex-M0/M0+ invece di un PIC
- UART bootloader (VDD, GND, TX, RX)

## Ricerche pubbliche — riepilogo risultati

- **Nessuna documentazione tecnica pubblica** trovata per il chip HP 135A/135X
  (pinout, protocollo, tensione operativa) — confermato assente in: forum
  elettronica (elektroda.com), GitHub (repo `milahu/reset-toner-chip`), brevetti
  HP, sito produttore clone (Apex Microelectronics)
- Nome tecnico interno HP per questi chip: **"E-label"**
- Manuale service HP LaserJet Pro M402/M403 (modello imparentato, non identico):
  - Rail logico dominante sulla scheda controller: **+3.3V** (nessun +5V per
    circuiti logici) → rinforza ipotesi 3.3V per il chip
  - Dichiara che il **livello toner è rilevato otticamente** dal controller
    stampante — **smentito empiricamente dall'utente su M234**: test con
    tamburo/corpo diverso ma chip originale riciclato da cartuccia esaurita ha
    fatto sì che la stampante segnalasse toner scarso → sul modello M234 il
    dato riportato **segue il chip**, non una misura ottica indipendente
  - Codici errore rilevanti: 10.0xx/10.1xx (errore/assenza memoria chip),
    10.3xx (cartuccia non autorizzata/non HP/contraffatta), 10.35x
    (supply incompatibile) — utile per diagnosticare in futuro il comportamento
    reale della stampante con un dato chip
- **HP Dynamic Security**: sistema di autenticazione crittografica
  challenge-response tra stampante e chip cartuccia, introdotto/rafforzato via
  aggiornamenti firmware, oggetto di controversie legali (FTC, right-to-repair).
  Chiave segreta memorizzata in flash interna del chip, non estraibile via
  semplice dump memoria per design.
- Vettore di attacco alternativo noto (lato sistema, non chip): **downgrade del
  firmware della stampante** a versione precedente l'enforcement di Dynamic
  Security (es. tool "chiplessprinter.com" per HP 135A/135W/135r) — bypassa il
  problema senza toccare il chip fisico. Rilevante per la stampante di questo
  esame, ma non ancora verificato se applicabile (firmware mai aggiornato
  manualmente, quindi già su versione di fabbrica).

## Stato attuale / domande aperte

1. **Chip originale**: protocollo di comunicazione reale ancora sconosciuto.
   Elettricamente "sano" ma silenzioso a tutti i tentativi (1-Wire a varie
   tensioni/timing). Blocco principale: nessuna documentazione pubblica,
   nessun modo pratico di catturare traffico reale senza accesso a una
   stampante.
2. **Chip clone**: sospetto danneggiato (comportamento diodo anomalo
   persistente sui pad grandi). Pad piccoli (ICSP presunto) non ancora testati
   elettricamente/funzionalmente in dettaglio; ipotesi PIC12F683 ICSP standard
   indebolita dalla mancanza di un quinto collegamento (MCLR).
3. Non è stato possibile ottenere il codice di errore esatto mostrato dalla
   stampante con il chip clone installato (utile per distinguere: errore di
   comunicazione vs modello/regione sbagliati vs fallimento crypto/auth).

## Prossimi passi possibili (da decidere)

- Accesso (anche breve, supervisionato) a una stampante compatibile con logic
  analyzer per catturare un ciclo di comunicazione reale — resta il modo più
  affidabile per sbloccare la comprensione del protocollo dell'originale
- Test I2C scan (via CH341A o Arduino) sui pad piccoli del clone, come
  alternativa a ICSP PIC
- Valutare se procurarsi un secondo chip clone "fresco"/funzionante come
  riferimento elettrico pulito, per distinguere danno vs protocollo sconosciuto
- Considerare la via del downgrade firmware stampante come vettore alternativo
  per l'obiettivo pratico (reset contatore), da verificare se applicabile a
  questo specifico modello/versione

## Sessione successiva — test timing esteso e ipotesi modulazione di corrente

### Test "slow-wake" (soak lungo prima del reset)

Ipotesi: il chip (se MCU-based) potrebbe aver bisogno di più tempo di quanto
testato finora (solo µs) per completare un vero boot firmware prima di
rispondere. Nuovo sketch (`toner_slow_wake.ino`): tiene la linea rilasciata
(pull-up) per un tempo di "soak" variabile (10/50/100/300ms) prima di inviare
il reset, poi campiona la risposta per una finestra molto più lunga (20ms
invece di 700µs).

**Risultato sul chip originale:** nessuna attività rilevata a nessun tempo di
soak testato (10-300ms). L'ipotesi "serve solo più tempo" è indebolita.

### Ipotesi non testata: comunicazione a modulazione di corrente

Ragionamento: con soli 2 contatti e nessuna antenna, le uniche categorie di
comunicazione fisicamente possibili sono (a) protocollo a livelli di tensione
digitali (testato, negativo), (b) modulazione di corrente sulla stessa linea
di alimentazione (il chip varia quanta corrente assorbe, la tensione resta
quasi costante — `digitalRead()` non la rileverebbe), (c) valore
resistivo/analogico statico (già escluso, il chip è un IC attivo).

Progettato schema con resistore di sense (470Ω) in serie tra pull-up e nodo
dati, con lettura differenziale via due ingressi analogici (A0/A1) per
calcolare la corrente istantanea (`I = (V_A1 - V_A0) / R_sense`) — **non
ancora implementato/testato**, resta il test aperto da tentare per primo alla
ripresa del lavoro.

### Episodio di falso positivo (lezione metodologica)

Durante il debug del cablaggio del test slow-wake, un errore di cablaggio poi
corretto ha temporaneamente prodotto un risultato che sembrava attività reale
del chip: 150-212 transizioni per ciclo di campionamento (contro 0 di tutti i
test precedenti), con tempi di prima transizione rapidi e consistenti
(15-56µs). Sul momento sembrava una scoperta.

**Verifica successiva ha smentito il dato**: ripetendo lo stesso test (stesso
sketch, dopo scollegare/ricollegare alimentazione e spostare il cablaggio su
una zona diversa della breadboard) il risultato è tornato a **zero
transizioni**, in modo stabile e ripetibile su più tentativi/posizioni.
Tensione misurata direttamente ai capi del chip in quella finestra: ~1.9-1.95V
stabile (valore intermedio, in zona ambigua per soglie logiche a 3.3V — ma
NON oscillante).

**Conclusione:** il picco di attività registrato una volta era quasi
certamente un **artefatto da contatto breadboard instabile** (chatter/rumore
da connessione imperfetta), non una risposta reale del chip. Non
riproducibile in condizioni altrimenti identiche. Da tenere a mente: la
breadboard usata ha mostrato più volte problemi di contatto/errori di
cablaggio nel corso della sessione — ogni risultato "positivo" isolato va
verificato con un secondo tentativo indipendente (idealmente su cablaggio
fisicamente diverso) prima di essere considerato dato reale.

### Test modulazione di corrente — eseguito, con controlli

Implementato circuito con resistore di sense (470Ω) in serie tra pull-up e
nodo dati, lettura differenziale via due ingressi analogici (A0 dopo il sense,
A1 prima) per stimare la corrente istantanea verso il chip
(`I = (V_A1-V_A0)/R_sense`).

**Primo tentativo (chip collegato):** range di corrente ~3.6mA per ciclo di
campionamento (10ms), apparentemente consistente su ~12 cicli consecutivi —
sembrava un segnale reale.

**Controllo 1 — A0/A1 pontati insieme (bypass del resistore di sense):**
range crolla a ~0.02-0.1mA dopo un breve transitorio di assestamento —
esclude che il primo risultato fosse puro rumore/offset tra i due canali ADC.

**Controllo 2 — chip scollegato, sostituito con resistore dummy (470Ω) verso
massa:** range quasi nullo (~0.01-0.04mA), corrente media perfettamente
stabile (~1.915mA) — comportamento atteso da un carico passivo puro.

**Controllo 3 — chip ricollegato, stesso identico test ripetuto:** range
torna a **~0.02-0.05mA**, indistinguibile dal controllo con resistore dummy.
**Il valore iniziale di ~3.6mA NON si riproduce.**

**Conclusione:** come il falso positivo delle transizioni digitali
(150+ eventi, poi non riprodotto), anche questo risultato era quasi
certamente un transitorio/artefatto legato al momento della modifica del
cablaggio, non un comportamento stabile del chip. Con controlli propri
(dummy load vs chip, ripetuto), **nessuna evidenza di modulazione di corrente
reale** — il chip si comporta, nella finestra temporale testata, in modo
elettricamente indistinguibile da un carico resistivo statico.

### Stato attuale — riepilogo ipotesi escluse sul chip originale

Con metodologia e controlli propri, sono state escluse:
- Comunicazione RFID/NFC (test smartphone negativo, nessuna antenna visibile)
- Protocollo digitale a livelli di tensione stile 1-Wire (varie tensioni
  3.3V/5V-equivalenti, timing da 60µs a 1000µs, soak fino a 300ms, entrambe
  le polarità)
- Modulazione di corrente sulla linea di alimentazione (con controlli
  positivo/negativo)
- Elemento puramente resistivo/passivo (il chip è un vero IC attivo, diodi
  ESD confermati)

Nessuna delle categorie di comunicazione fisicamente plausibili con 2 soli
contatti e nessuna antenna ha prodotto un segnale ripetibile. Il lavoro
sperimentale sul chip originale, con la strumentazione disponibile (Arduino,
multimetro, breadboard), ha raggiunto un punto di stallo metodologicamente
solido: risultato negativo rigoroso, non conclusivo sul "come comunica" ma
utile a restringere il campo (o richiede capture di traffico reale da una
stampante, o richiede strumentazione/tecnica non ancora disponibile in questo
lavoro).

## Terzo falso positivo — ipotesi "comunica solo all'accensione" (smascherato)

### Setup

Nuovo sketch (`toner_cold_boot.ino`): ascolto passivo puro in loop continuo
(nessun impulso di reset mai inviato, solo `digitalRead` continuo), finestre
successive di 200-600ms. Razionale: se il chip comunica solo una volta subito
dopo un vero power-on da zero (mai testato finora — l'alimentazione era
sempre rimasta accesa tra un tentativo e l'altro), un ascolto passivo attivato
nel momento esatto in cui il chip viene fisicamente ricollegato (dopo essere
stato scollegato, quindi scarico) potrebbe catturarlo.

### Risultato iniziale — sembrava una scoperta

Con chip scollegato poi ricollegato mentre lo sketch era già in ascolto:
catturata attività strutturata — raffiche di transizioni rapide (5-20µs tra
un cambio e l'altro) separate da pause di alcuni millisecondi, per un totale
di 200-300+ms di attività continua dopo il collegamento, poi silenzio stabile
e prolungato. **Riprodotto identico su due connessioni fisiche indipendenti**
(non un singolo evento isolato).

Test con **polarità invertita**: attività molto più rada e priva di
struttura (solo ~11 eventi per raffica, spaziatura di decine-centinaia di ms,
non µs) — asimmetria che sembrava confermare una polarizzazione reale del
chip.

### Smascheramento — controllo decisivo

Ripetendo la stessa identica procedura (scollega/aspetta/ricollega) ma
**senza il chip collegato affatto** (solo il filo DATA lasciato flottante
lato Arduino, nessun collegamento al chip): **lo stesso identico pattern è
ricomparso**, con la stessa struttura a raffiche e gli stessi tempi.

Analisi dei tempi tra raffiche nella cattura più lunga e regolare: gap
alternati di **~11.6-12.9ms** e **~6.8-8.2ms**, la cui somma è costantemente
**~19.7-19.9ms** — periodicità di **circa 50Hz**, esattamente la frequenza di
rete elettrica italiana/europea.

**Conclusione:** l'intera struttura a raffiche osservata (in entrambi i test
precedenti, chip collegato in un verso o nell'altro) era **ronzio di rete a
50Hz captato dalla linea flottante/alta impedenza**, non comunicazione del
chip. Una linea con pull-up debole e livelli logici marginali (coerente con
le tensioni ambigue viste più volte nella sessione: 0.6-0.7V, 1.9V) è
particolarmente suscettibile a questo tipo di interferenza ambientale,
specialmente su una breadboard priva di schermatura vicino a strumentazione
da banco alimentata da rete.

### Lezione metodologica

Tre falsi positivi in questa sessione (transizioni digitali dopo fix
cablaggio, range di corrente dopo riconnessione, ora questo pattern a
raffiche "post-accensione") condividono lo stesso rischio: un risultato che
appare "strutturato" o "consistente" non è di per sé prova di segnale reale.
Ogni risultato positivo va sottoposto a controlli specifici prima di essere
accettato:
- Ripetibilità a distanza di tempo, senza toccare il circuito
- Controllo negativo (stesso test, ma con il componente sotto esame rimosso/
  sostituito da un carico noto)
- Se il pattern è periodico, verificarne la frequenza contro sorgenti di
  rumore ambientale note (rete elettrica 50Hz, clock di altri dispositivi,
  ecc.)

Con questi controlli, **anche l'ipotesi "comunica solo all'accensione" risulta
al momento non supportata** da dati reali — nessuna attività attribuibile con
certezza al chip è stata osservata in nessuna condizione testata finora.

### Controlli aggiuntivi — perché la polarità sembrava contare

Osservazione dell'utente: se è solo rumore, perché la polarità invertita
dava molti meno eventi rispetto alla polarità "corretta"? Spiegazione
verificata: il rumore ambientale è fisicamente identico in entrambi i casi,
ma il diodo interno del chip (già caratterizzato nei test precedenti) ancora
la tensione di riposo della linea in un punto diverso a seconda
dell'orientamento — vicino alla soglia logica (~0.6-1.9V, zona ambigua) in un
verso, lontano dalla soglia (livello stabile) nell'altro. Lo stesso ripple di
rumore, sommato a un livello già vicino alla soglia, la fa attraversare
ripetutamente (molti eventi registrati); sommato a un livello lontano dalla
soglia, quasi mai (pochi eventi). Nessuna contraddizione con l'ipotesi
rumore — anzi, la conferma con un meccanismo coerente.

**Controllo finale:** stesso cablaggio "invertito" ma con il chip
completamente scollegato (solo filo flottante). Risultato: pattern ancora
diverso (gap quasi costanti ~10ms, periodicità ~100Hz — armonica doppia della
rete elettrica, plausibile ripple da alimentatori switching/trasformatori
nelle vicinanze), eventi più radi. Conferma ulteriore: la "firma" del rumore
captato dipende dal filo fisico specifico usato come nodo flottante (diversa
geometria/lunghezza = diversa sensibilità come antenna), ma **il chip è
assente in questo test** — quindi qualunque pattern osservato, in qualunque
configurazione, resta pienamente spiegabile da interferenza ambientale.

### Conclusione definitiva su questa pista

Testate tutte le combinazioni ragionevoli (chip presente/assente, polarità
dritta/invertita) — il rumore ambientale (rete elettrica, 50Hz e sua
armonica 100Hz) spiega integralmente ogni pattern osservato, inclusi quelli
che inizialmente sembravano strutturati/promettenti. Pista chiusa: nessuna
prova di comunicazione reale del chip originale ottenuta con questo metodo,
nonostante test approfonditi e ripetuti con controlli propri.

### Ultimo giro di test — pull-up massimo e 5V

Per completezza, testate anche le combinazioni rimaste:
- Pull-up massimo disponibile a 3.3V (4× 470Ω in parallelo, ~117Ω, ~28mA) —
  entrambe le polarità: linea bloccata bassa (stesso comportamento a diodo
  già caratterizzato), zero transizioni
- 5V con resistore di protezione (470Ω) — entrambe le polarità: identico,
  linea bloccata bassa, zero transizioni

**Risultato:** in nessuna combinazione di tensione (3.3V debole/forte, 5V),
polarità, timing o soak testata nel corso della sessione si è ottenuta una
risposta di protocollo dal chip originale. L'unico comportamento osservato
resta o una linea pulita che sale regolarmente sotto pull-up debole
(nessuna anomalia, nessuna attività), oppure un blocco a diodo interno sotto
pull-up più forte (qualunque polarità/tensione) — mai attività dinamica
attribuibile con certezza al chip stesso.

### Valutazione finale — perché l'approccio elettrico "a tentoni" non basta

Fisicamente, con 2 soli contatti e nessuna antenna, il chip comunica
necessariamente con un protocollo a livelli di tensione — le altre categorie
(modulazione di corrente, RFID, elemento passivo) sono state escluse con
prove proprie. La mancata risposta non è quindi prova che il chip non
comunichi così, ma indica che lo stimolo generato con resistori passivi e
timing generato via software (Arduino) non replica con sufficiente
precisione (slew rate, tolleranze di timing strette, pilotaggio a bassa
impedenza) quello che un vero controller stampante genera. Il gap tra
"fisicamente possibile" e "abbiamo trovato la ricetta esatta" resta aperto e,
senza documentazione pubblica del protocollo, richiede o un capture di
traffico reale da stampante, o l'analisi del firmware ufficiale della
stampante (dove il codice che genera questo stesso segnale è scritto in
chiaro) come prossima pista, prima di poter concludere in modo definitivo.
