# Proiectul Nr.1 - Agent de mesagerie. Invocarea la distanță.

Soluție .NET cu 4 proiecte, deschide `MessageBrokerLab.sln` în Rider.

## Partea 1 - Socketuri (`Common`, `Broker`, `Sender`, `Receiver`)

Arhitectură **Publisher/Subscriber pe bază de subiect (topic)**, peste TCP,
mesaje serializate JSON (`Common/MessageProtocol.cs`, framing cu prefix de
lungime).

- **Protocol de transport**: TCP (nu UDP) - am ales TCP pentru că broker-ul și
  receiverii trebuie să primească mesajele complete, în ordine, fără
  pierderi; UDP ar necesita reconstrucție/reordonare la nivel de aplicație,
  overhead nejustificat pentru scopul acestui laborator.
- **Broker** (`Broker/Program.cs`) ascultă pe portul `5050`. Fiecare conexiune
  e tratată pe un `Task` separat -> procesare concurentă (cerința 2.2).
  - Dacă prima conexiune trimite un mesaj `Command`/`SUBSCRIBE`, e
    înregistrată ca **subscriber** pe subiectul indicat.
  - Altfel e tratată ca **publisher**: fiecare mesaj trimis e pus în coadă.
- **Coadă separată per topic**: `ConcurrentQueue<Message>` pentru fiecare
  topic (queue1 -> topic1, queue2 -> topic2, ...), colecție thread-safe .NET.
- **Persistența mesajelor** (`Broker/PersistentStore.cs`), XML pe disc sub
  `Broker/bin/Debug/net10.0/storage/`:
  - `history/<topic>/` - istoricul complet al topicului, NU se șterge. Un
    subscriber care se abonează mai tîrziu primește toată conversația de
    pînă atunci.
  - `pending/<topic>/` - mesaje încă nelivrate; se șterg după livrare. La
    repornirea Brokerului sînt restaurate (recuperare după cădere).
  - `deadletter/<topic>/` - **Dead Letter Queue**: mesaje care nu au putut fi
    livrate nici după 3 încercări (`BrokerHub.MaxDeliveryAttempts`).
- **Politica de reîncercare**: dacă livrarea eșuează, mesajul revine în coadă;
  după 3 încercări eșuate ajunge în DLQ și nu mai blochează coada. Dacă pur
  și simplu nu există niciun subscriber, mesajul așteaptă fără a consuma
  încercări.
- **Rutare** (`Broker/DeliveryWorker.cs`): un "work job" de fundal rulează
  la fiecare secundă și livrează concurent mesajele stocate pe fiecare
  subiect către toți subscriberii activi ai acelui subiect. Dacă nu există
  niciun subscriber, mesajele rămîn stocate pînă apare unul.
- **Tratarea excepțiilor**: date corupte (non-JSON) primite de la un
  Sender sau livrarea eșuată către un subscriber deconectat sînt prinse și
  logate, fără ca Brokerul să cadă.

### Ordinea de pornire (Partea 1)

1. `Broker`
2. `Receiver` - la pornire îți cere subiectul (topic) la care te abonezi
3. `Sender` - la pornire îți cere subiectul pe care publici

Poți porni oricîți Sender/Receiver vrei, pe subiecte diferite sau la fel.

### Rulare pe 3 calculatoare diferite (Broker / Sender / Receiver separate)

Broker-ul ascultă deja pe toate interfețele de rețea (`0.0.0.0`), nu doar
pe `127.0.0.1`, iar Sender/Receiver acceptă adresa Broker-ului ca argument
de linie de comandă - deci se poate rula fiecare rol pe alt calculator,
atîta timp cît toate 3 sînt în aceeași rețea (ex. același Wi-Fi/LAN).

1. Pe calculatorul care rulează **Broker**-ul:
   ```
   cd Broker
   dotnet run
   ```
   La pornire, Broker-ul afișează adresa IP locală (ex. `192.168.1.23`) -
   asta e adresa pe care o dau celelalte 2 calculatoare mai jos.
2. Pe calculatorul cu **Sender**:
   ```
   cd Sender
   dotnet run -- 192.168.1.23 5050
   ```
3. Pe calculatorul cu **Receiver**:
   ```
   cd Receiver
   dotnet run -- 192.168.1.23 5050
   ```

Înlocuiește `192.168.1.23` cu IP-ul afișat de Broker pe calculatorul lui.
Dacă Sender/Receiver rulează pe același calculator ca Broker-ul, poți
folosi `dotnet run` fără argumente (implicit `127.0.0.1:5050`).

Cerințe pentru ca legătura să funcționeze între calculatoare diferite:
- toate 3 calculatoarele trebuie să fie în aceeași rețea locală;
- portul `5050` trebuie permis de firewall-ul calculatorului cu Broker-ul
  (pe Mac: System Settings -> Network -> Firewall; pe Windows: Windows
  Defender Firewall -> Allow an app);
- rețelele de tip "public"/"guest" (ex. hotspot de telefon cu izolare de
  clienți, eduroam la unele universități) pot bloca traficul direct între
  calculatoare - în acest caz încearcă un hotspot personal fără izolare,
  sau conectează toate 3 la același router.


## Structura proiectului

```
MessageBrokerLab.sln
Common/            - Message, MessageProtocol
Broker/            - Partea 1: Broker (Pub/Sub, storage, worker)
Sender/            - Partea 1: Sender (Publisher)
Receiver/          - Partea 1: Receiver (Subscriber)
```
