# meshcore_USB_control
====================

This is an embryonic project that I hope will develop into a complete node-red interface to a Meschcore USB companion.
The device used as a first approach is the\
Heltec V3 LoRa32.\
See the following address:\
[https://meshcore.co.uk/get.html]

## minimal reiquirements\
I installed Node-Red on a VMware Linux virtual machine (Ubuntu in my case)\
It doesn't require any special computing power, so in my opinion, this is an ideal solution for avoiding adding numerous applications to the host system and keeping the development environment isolated. Furthermore, everyone is well aware of the advantages of working in a virtual machine: _snapshots, copy-and-paste, move, etc._\
Naturally, you'll need a Heltec V3 and a USB cable to connect it.
In Linux, a standard user cannot access the USB port unless they're part of the "dialout" group.\
So, run this command to ensure there are no problems accessing the USB port.\
`usermod -a -G dialout $USER`\
When you insert the Heltec V3 device into the USB port, it automatically becomes visible to the host system.\
To transfer access to the virtual guest, you need to connect to it from the virtual machine menu.\
`VM - Removable Devices - Silicon CP2102 - Connect`
Now, if you haven't already done so, you need to install node-red by following the official guide at this address:
[https://nodered.org/docs/getting-started/local]
This is the message that node-red writes to my terminal when launching its instance:\

29 Jan 18:40:20 - [info] Node-RED version: v4.1.3\
29 Jan 18:40:20 - [info] Node.js version: v18.19.1\
29 Jan 18:40:20 - [info] Linux 6.14.0-37-generic x64 LE\
29 Jan 18:40:21 - [info] Loading palette nodes\
29 Jan 18:40:22 - [info] Dashboard version 3.6.6 started at /ui\
29 Jan 18:40:22 - [info] Settings file : ~/.node-red/settings.js\
29 Jan 18:40:22 - [info] Context store : 'default' [module=memory]\
29 Jan 18:40:22 - [info] User directory : ~/.node-red\
29 Jan 18:40:22 - [info] Projects directory: ^/.node-red/projects\
29 Jan 18:40:22 - [info] Server now running at http://127.0.0.1:1880/\
29 Jan 18:40:22 - [info] Active project : meshcore_USB_control\
29 Jan 18:40:22 - [info] Flows file: ~/.node-red/projects/meshcore_USB_control/flows.json\
29 Jan 18:40:22 - [info] Starting flows\
29 Jan 18:40:22 - [info] Started flows\
29 Jan 18:40:24 - [info] [serialconfig:ec7bf5fc2e81df2b] serial port /dev/ttyUSB0 opened at 115200 baud 8N1\

The last line refers to the opening of the /dev/ttyUSB0 port with baud rate 115200 and parameters N81\
This happens when a serial communication node is present in the default node-red flow.\
To use a serial communication node in node-red, you must install nodered-node-serialport.\
This operation is very easy, just access the menu\
`Manage palette - Install (tab) - search modules - nodered-node-serialport`\
then install the module.\
From now on, since I share the flows that I keep updated on Git, I don't think there will be any need to describe the communication modules settings anymore; just open them to see what's inside.

## flux description\
For now, there are only two flows.\
In the first (FLOW1), the working one, the messages sent to the Heltec module via USB will be assembled and tested. The responce messages the device retransmits to the USB port are also read in the same flow.\
The second flow contains the nodes communicating with the USB port.\
This separation is necessary to deploy only the modified flows, excluding those operating on the USB ports.\
Otherwise, the serial port would crash, and it would be necessary to close and run again node-red.\

## Implemented commands
Currently, the implemented messages are few and essential. They are only those useful for verifying the correct transmission and reception of responses.\
_query_\
_reboot_\
_start_\
_get\_time_\
_self\_advert_\
_set\_name_\
Pressing the inject button on the node corresponding to the command will generate the sequence for transmitting the command to the USB port. At the same time, if the device responds with a string, it will be captured and transmitted in the debug window.

## Future steps
As I've written several times, this is just a starting point for creating a node-red interface to a companion device.
One of the next steps will be to create a global memory to store important meshcore protocol variables, such as node data captured from issued advertisements and the meshcore nodes to use for ACC hops.\
Furthermore, the documentation of the _function_ nodes will be improved and maximized.

## Aggiornamento 1 AI (Ita)
# Documentazione Tecnica: Nodo TX (Encoder Protocollo)

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
# 1. Costanti del Protocollo\
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

