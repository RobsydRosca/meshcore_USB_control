## Aggiornamento 1 (Ita)
# Documentazione: Nodo TX. Generata via AI (Gemini)

## Descrizione Generale
Questo script è progettato per essere utilizzato all'interno di un nodo **Function** di Node-RED (denominato "TX" o "Encoder").
Il suo scopo è agire come un **traduttore polimorfico**: accetta comandi in formati semplici (numeri, stringhe, oggetti JSON) e li converte nel protocollo binario rigoroso richiesto dal dispositivo target via seriale.

## Codice Sorgente

Copia il seguente codice nel corpo del nodo `Function`:

```javascript
// --- CONFIGURAZIONE COSTANTI ---
const HEADER = 0x3C;          // Byte di sincronizzazione (Start of Frame)
const OPCODE_TEXT_CMD = 0x13; // 19 dec: OpCode per tunneling comandi testuali (CLI)

// Variabili di lavoro
let opCode;
let dataBuffer;

// --- 1. LOGICA DI SELEZIONE COMANDO (Input Handling) ---

// CASO A: Ingresso Numerico -> È un OpCode diretto
// Esempio: msg.payload = 22 (Query), msg.payload = 1 (Start)
if (typeof msg.payload === 'number') {
    opCode = msg.payload;
    dataBuffer = Buffer.alloc(0); // Nessun payload dati aggiuntivo, solo l'OpCode
}

// CASO B: Ingresso Stringa -> È un comando CLI testuale
// Esempio: msg.payload = "reboot"
else if (typeof msg.payload === 'string') {
    opCode = OPCODE_TEXT_CMD;          // Forza l'uso dell'OpCode 0x13 (Tunnel Testo)
    dataBuffer = Buffer.from(msg.payload); // Converte la stringa in Buffer ASCII
}

// CASO C: Ingresso Oggetto -> Comando Complesso / Parametrico
// Esempio: { op: 32, data: [0xFF] } oppure { op: 65, name: "Node1" }
else if (typeof msg.payload === 'object' && msg.payload.op) {
    opCode = msg.payload.op;
    
    if (msg.payload.name) {
        // Se c'è una proprietà "name", la convertiamo in Buffer ASCII
        dataBuffer = Buffer.from(msg.payload.name);
    } else {
        // Altrimenti usiamo l'array "data" raw (o vuoto se assente)
        dataBuffer = Buffer.from(msg.payload.data || []);
    }
}

// Gestione Errori: Input non riconosciuto
else {
    node.error("Tipo di payload non supportato: " + typeof msg.payload);
    return null;
}

// --- 2. COSTRUZIONE DEL PACCHETTO (Packet Assembly) ---

// A. Calcola la lunghezza totale del payload protocollo
// NOTA: In questo protocollo, la lunghezza include l'OpCode (1 byte) + i Dati
const payloadLength = 1 + dataBuffer.length;

// B. Crea il Buffer della lunghezza (2 bytes, Little Endian)
const lenBuffer = Buffer.alloc(2);
lenBuffer.writeUInt16LE(payloadLength, 0);

// C. Assembla il pacchetto finale concatenando i buffer
// Struttura: [HEADER] [LEN_LOW] [LEN_HIGH] [OPCODE] [DATA...]
msg.payload = Buffer.concat([
    Buffer.from([HEADER]),       // 0x3C
    lenBuffer,                   // Lunghezza (2 bytes)
    Buffer.from([opCode]),       // L'OpCode effettivo (1 byte)
    dataBuffer                   // I dati (N bytes)
]);

// --- 3. DEBUG E OUTPUT ---
// Log visivo per debug rapido nella console di Node-RED
node.warn(`TX -> OpCode: ${opCode} (0x${opCode.toString(16).toUpperCase()}), Len: ${payloadLength}`);

return msg;
```

## Dettagli Funzionamento
# 1. Costanti del Protocollo
HEADER (0x3C): Byte di sincronizzazione. Ogni pacchetto valido deve iniziare con questo byte per essere riconosciuto dal firmware.\
OPCODE_TEXT_CMD (0x13): Codice operativo speciale (decimale 19). Funge da "busta" (wrapper) per inviare stringhe ASCII al parser CLI del dispositivo.\

# 2. Gestione degli Input (Input Handling)
Lo script adatta il comportamento in base al tipo di dati ricevuto (msg.payload):\
Input Numerico (Number):\
Viene interpretato direttamente come OpCode.\
Esempio: Inviare 22 genera una richiesta Device Query.\
Nessun dato aggiuntivo viene allegato.\
Input Stringa (String):\
Viene trattato come comando testuale CLI.\
L'OpCode viene forzato a 0x13.\
La stringa viene convertita in Buffer ASCII e allegata come payload.\
Input Oggetto (Object):\
Per comandi avanzati che richiedono parametri specifici.\
Struttura attesa: { op: <number>, data: <array> } oppure { op: <number>, name: <string> }.

# 3. Assemblaggio del Pacchetto (Packet Assembly)
Il protocollo richiede una struttura binaria precisa:\

## Dettagli di Funzionamento

### 1. Costanti del Protocollo

* **`HEADER` (`0x3C`):** È il "byte di start". Ogni pacchetto valido deve iniziare con questo valore esadecimale affinché il firmware del ricevitore inizi a processarlo.
* **`OPCODE_TEXT_CMD` (`0x13`):** (Decimale 19). È un OpCode speciale che funge da "busta" (wrapper). Dice al dispositivo: *"Ciò che segue non è un dato binario, ma una stringa di testo da passare alla Command Line Interface (CLI)"*.

### 2. Gestione degli Input (Polimorfismo)

Il nodo adatta automaticamente il comportamento in base al tipo di dati (`typeof`) ricevuto in ingresso:

* **Numero (`Number`):** Interpretato come **Comando Diretto**.
    * Utile per comandi di sistema veloci come *Ping*, *Status*, *Query*.
    * Non ha payload dati, solo l'istruzione.
* **Stringa (`String`):** Interpretata come **Comando CLI**.
    * Viene automaticamente incapsulata nell'OpCode `0x13`.
    * Utile per comandi come `"nodes"`, `"mesh info"`, `"reboot"`.
* **Oggetto (`Object`):** Interpretato come **Comando Avanzato**.
    * Permette di specificare manualmente sia l'OpCode che i Dati.
    * Supporta parametri binari (array di byte) o stringhe (nomi/ID).

### 3. Assemblaggio Binario

Il protocollo richiede una struttura specifica dei byte. Lo script gestisce due aspetti critici spesso causa di errori:

1.  **Calcolo della Lunghezza:** La lunghezza dichiarata nel pacchetto deve includere la dimensione dei dati **PIÙ 1 byte** (che rappresenta l'OpCode stesso).
2.  **Endianness:** La lunghezza (che è un numero a 16 bit) viene scritta in formato **Little Endian** (il byte meno significativo prima), standard per la maggior parte dei microcontrollori (ESP32, ARM).

### Tabella Esempi di Trasmissione

La seguente tabella illustra come diversi input (`msg.payload`) vengono trasformati nel pacchetto esadecimale finale (`msg.payload` in uscita).

| Tipo Input | Input (msg.payload) | OpCode | Pacchetto Generato (Hex) | Note |
| :--- | :--- | :--- | :--- | :--- |
| **Number** | `22` | `0x16` | `3C 01 00 16` | Comando diretto (Query). Len=1. |
| **String** | `"reboot"` | `0x13` | `3C 07 00 13 ...` | Stringa incapsulata. |
| **Object** | `{op:32, ...}` | `0x20` | `3C 02 00 20 ...` | Comando custom. |

