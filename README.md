# relay0

# Minimal UDP Rendezvous Protocol

## Obiettivo

Definire una primitiva di rete minimale che permetta a due peer che conoscono lo stesso token di:

1. incontrarsi attraverso un relay pubblico;
2. ottenere reciprocamente il proprio endpoint pubblico IP:porta;
3. tentare una connessione P2P diretta;
4. eliminare immediatamente ogni stato dal relay.

Il relay non trasporta il traffico applicativo e non conserva dati in modo persistente.

L'idea fondamentale è:

```text
token → meet
```

Non vogliamo costruire:

- identità globali;
- account;
- filesystem;
- message broker;
- HTTP API;
- WebSocket;
- database;
- routing alternativo.

Vogliamo solo un rendezvous effimero.

---

## Architettura

```text
Peer A ───────────┐
                  │
                  ▼
            Rendezvous Relay
                  ▲
                  │
Peer B ───────────┘

            endpoint exchange

Peer A ◄──── endpoint B
Peer B ◄──── endpoint A

                 ↓
               forget

Peer A ═════════════════════ Peer B
               P2P
```

Il relay funziona solo come punto di incontro.

---

## Livello di rete

Il protocollo gira direttamente sopra UDP:

```text
Application
-------------------------
Rendezvous Protocol
-------------------------
UDP
-------------------------
IP
-------------------------
Ethernet / Wi-Fi / ...
```

Niente HTTP.

Niente TLS obbligatorio.

Niente JSON.

Niente WebSocket.

UDP è probabilmente il livello più basso praticamente utilizzabile su Internet reale, perché protocolli IP custom sarebbero frequentemente bloccati da NAT e firewall.

---

## Rendezvous ID

I due peer condividono un token casuale:

```text
room_id = 128 o 256 bit
```

Esempio concettuale:

```text
6d7a89d0e5154f219b3f4c7a8e41...
```

Il token deve essere sufficientemente imprevedibile.

Il relay non ha bisogno di sapere chi sono i peer.

Conosce solamente:

```text
room_id
source_ip
source_port
timestamp
```

Il token stesso può essere considerato una capability:

```text
chi conosce il token può partecipare al rendezvous
```

In una versione successiva si potranno aggiungere firme, autenticazione e verifica del relay, ma non fanno parte del nucleo iniziale.

---

## Primo peer

Peer A invia:

```text
REGISTER
room_id = X
```

Il relay riceve il pacchetto UDP e osserva direttamente il suo endpoint pubblico:

```text
151.32.10.4:51324
```

Il relay crea temporaneamente:

```text
X → 151.32.10.4:51324
```

e può rispondere:

```text
WAIT
```

---

## Secondo peer

Peer B invia:

```text
REGISTER
room_id = X
```

Il relay osserva:

```text
93.44.21.7:62001
```

Adesso il relay possiede temporaneamente:

```text
X → A
X → B
```

e invia:

```text
A ← PEER 93.44.21.7:62001

B ← PEER 151.32.10.4:51324
```

Subito dopo:

```text
delete X
```

Il relay ha terminato il proprio compito.

---

## Comunicazione diretta

Dopo aver ricevuto l'endpoint dell'altro peer:

```text
A → B
B → A
```

entrambi iniziano immediatamente a inviare pacchetti UDP.

Questo permette di tentare UDP hole punching.

Il rendezvous relay non partecipa più alla comunicazione.

---

## Stato del relay

Il relay usa soltanto RAM.

Una entry concettuale potrebbe essere:

```c
struct rendezvous_entry {
    uint8_t room_id[32];
    sockaddr_storage peer;
    uint64_t expires_at;
};
```

Il TTL deve essere breve.

Esempio:

```text
5 secondi
10 secondi
30 secondi
```

Se il secondo peer non arriva entro il TTL:

```text
delete X
```

Non è necessario alcun database.

Non è prevista persistenza.

Non esiste cronologia delle sessioni.

---

## Semantica

Il servizio può essere descritto come:

```text
R : Token × Endpoint → Endpoint ∪ WAIT
```

Prima registrazione:

```text
R(X, A) = WAIT
```

Seconda registrazione:

```text
R(X, B) = A
```

e contemporaneamente:

```text
notify(A, B)
```

poi:

```text
state[X] = ∅
```

Questa è l'intera semantica fondamentale.

---

## Protocollo binario

Il pacchetto dovrebbe essere estremamente piccolo.

Formato iniziale possibile:

```text
+--------+---------+----------------+
| magic  | opcode  | room_id        |
| 2 byte | 1 byte  | 32 byte        |
+--------+---------+----------------+
```

Totale:

```text
35 byte
```

Esempio:

```text
magic  = 0x5256
opcode = 0x01
```

dove `RV` significa Rendezvous.

---

## Messaggi

Prima versione:

```text
REGISTER
PEER
```

Eventualmente:

```text
WAIT
ERROR
PING
```

ma il protocollo base dovrebbe cercare di restare il più piccolo possibile.

### REGISTER

```text
REGISTER {
    room_id
}
```

L'endpoint non viene dichiarato dal client.

Il relay usa:

```text
source IP
source UDP port
```

del datagramma ricevuto.

Questo è importante perché consente al relay di osservare direttamente la mappatura NAT.

### PEER

Risposta:

```text
PEER {
    address
    port
}
```

Il relay comunica a ciascun peer l'endpoint osservato dell'altro.

---

## NAT

Supponiamo:

```text
Peer A locale
192.168.1.20:4000
```

Il NAT lo trasforma in:

```text
151.32.10.4:51324
```

Il relay vede:

```text
151.32.10.4:51324
```

e comunica proprio questo endpoint a B.

Analogamente per B.

Il rendezvous svolge quindi implicitamente anche una funzione simile a una parte di STUN:

```text
client → relay
relay osserva source endpoint
```

ma in più associa due peer che condividono lo stesso token.

---

## Ritrasmissioni

UDP non garantisce consegna.

Il client può quindi usare una strategia semplice:

```text
REGISTER
wait 250 ms

REGISTER
wait 500 ms

REGISTER
wait 1 s
```

Il relay deve trattare `REGISTER` in maniera idempotente.

Una ritrasmissione dello stesso peer non deve creare un nuovo partecipante.

---

## Identificazione dello stesso peer

Per la prima implementazione si può considerare uguale:

```text
room_id + source IP + source UDP port
```

Quindi:

```text
REGISTER X da A
REGISTER X da A
```

è una semplice ritrasmissione.

Mentre:

```text
REGISTER X da B
```

produce il match.

---

## Collisioni e numero di peer

Il protocollo iniziale è pairwise.

Ogni `room_id` associa esattamente due peer.

```text
A + B → MATCH
```

Il terzo peer che utilizza lo stesso token dovrebbe ricevere:

```text
ERROR
```

oppure iniziare una nuova sessione solo dopo che la precedente è stata eliminata.

La semantica più pulita è:

```text
room_id → massimo due partecipanti
```

---

## Privacy

Il relay conosce soltanto temporaneamente:

```text
room_id
endpoint A
endpoint B
```

Non conosce necessariamente:

```text
username
email
device identity
account
nome host
```

Il `room_id` potrebbe inoltre essere derivato da un segreto condiviso:

```text
room_id = H(shared_secret || time_window)
```

Per esempio:

```text
H(secret || 2026-10-01T16:05)
```

A e B possono così calcolare autonomamente lo stesso rendezvous ID senza inviare il segreto originale.

Questo può essere aggiunto a livello client senza modificare il relay.

---

## Affidabilità del relay

Il protocollo dovrà prevedere in seguito un meccanismo di verifica del relay.

Obiettivo futuro:

```text
il client deve poter distinguere
un relay ufficiale/non modificato
da una build custom o manomessa
```

La struttura del protocollo dovrebbe quindi riservare spazio per:

```text
relay identity
build identity
firma
eventuale attestation
```

ma questa parte non viene approfondita nella prima versione.

Per ora il modello è:

```text
client sceglie un relay
client decide se fidarsi di quel relay
```

---

## Cosa NON fa il relay

Il relay non deve:

```text
proxyare traffico
conservare file
conservare messaggi
creare utenti
autenticare account
risolvere nomi
mantenere sessioni persistenti
fornire API HTTP
```

Fa soltanto:

```text
receive endpoint A
remember briefly
receive endpoint B
exchange endpoints
forget
```

---

## Proprietà desiderate

Il protocollo deve essere:

```text
minimal
UDP-native
binary
ephemeral
pairwise
stateless nel lungo periodo
RAM-only
senza account
senza database
senza dipendenze applicative
facile da implementare
facile da implementare anche embedded
```

---

## Implementazione concettuale del relay

Pseudo-codice:

```text
on REGISTER(room_id, source):

    if room_id does not exist:
        table[room_id] = {
            peer: source,
            expires: now + TTL
        }

        return WAIT


    entry = table[room_id]

    if entry.peer == source:
        return WAIT


    peerA = entry.peer
    peerB = source

    send PEER(peerB) to peerA
    send PEER(peerA) to peerB

    delete table[room_id]
```

Cleanup:

```text
for each entry:
    if entry.expires < now:
        delete entry
```

---

## Nome concettuale

Possibili definizioni:

```text
Minimal UDP Rendezvous Protocol
```

oppure:

```text
Ephemeral Pairwise Rendezvous Protocol
```

Il server potrebbe essere chiamato:

```text
Rendezvous Relay
```

oppure, forse più correttamente:

```text
Rendezvous Point
Rendezvous Switch
```

Il termine `relay` è leggermente improprio perché il server non inoltra il traffico P2P vero e proprio.

---

## Definizione sintetica

> Un servizio UDP effimero che associa temporaneamente due endpoint attraverso un capability token condiviso, comunica a ciascun peer l'endpoint osservato dell'altro e cancella immediatamente l'associazione.

Oppure, nella forma più corta:

```text
token → meet → exchange endpoints → forget
```

Questa deve rimanere la proprietà centrale del protocollo.
