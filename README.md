A Spring Boot backend that demonstrates offline UPI payments routed through a Bluetooth-style mesh network. 
You're in a basement with zero connectivity. You send your friend ₹500. Your phone encrypts the payment,
broadcasts it to nearby phones, and the packet hops device-to-device until some phone walks outside, gets 4G,
and silently uploads it to this backend. The backend decrypts, deduplicates, and settles.

The demo flow (step by step)
The dashboard has four buttons that walk through the full pipeline. The intended sequence:

Step 1 — Compose a payment
Choose sender, receiver, amount, PIN. Click "📤 Inject into Mesh".

What actually happens on the backend:

The server pretends to be the sender's phone.
It builds a PaymentInstruction with a unique nonce and current timestamp.
It encrypts that with the server's RSA public key (using hybrid encryption — see below).
It wraps the ciphertext in a MeshPacket with a TTL of 5.
It hands the packet to phone-alice, an offline virtual device.
You'll see phone-alice now holds 1 packet.

Step 2 — Run gossip rounds
Click "🔄 Run Gossip Round". Then click it again.

Each round, every device that holds a packet broadcasts it to every other device within "Bluetooth range" (which, in our simulator, means everyone). TTL decrements per hop.

After 1 round: every device holds the packet. After 2 rounds: still every device — TTL is just lower.

In the real system this would happen organically as people walk past each other in the basement.

Step 3 — Bridge node walks outside
Click "📡 Bridges Upload to Backend".

phone-bridge is the only device with hasInternet=true. The dashboard simulates that phone walking outside and getting 4G. It POSTs every packet it holds to /api/bridge/ingest.

The backend pipeline runs:

Hash the ciphertext (SHA-256).
Try to claim the hash in the idempotency cache.
If claimed: decrypt with the server's RSA private key.
Verify freshness (signedAt within 24 hours).
Run the debit/credit in a single DB transaction.
Watch the Account Balances table — money has moved. Watch the Transaction Ledger — a new row appears.

Step 4 — Demonstrate idempotency (the killer feature)
Reset the mesh. Inject a single packet. Run gossip 2 times. Now all 5 devices hold the same packet, including multiple bridges in a more complex setup.

To really see idempotency in action, modify MeshSimulatorService.java to seed multiple bridge devices, or just:

Click "Inject" once.
Click "Gossip" twice.
Click "Flush Bridges" — only phone-bridge is a bridge in the default seed, so just one upload happens.
To exercise the concurrent duplicate case properly, run the test:

mvnw.cmd test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce

This test creates one packet, fires 3 threads at BridgeIngestionService.ingest() simultaneously, 
and verifies that exactly one settles,
two are dropped as duplicates, and the sender is debited exactly once.

┌─────────────────────────────────────────────────────────────────────────┐
│                         SENDER PHONE (offline)                          │
│  PaymentInstruction { sender, receiver, amount, pinHash, nonce, time }  │
│              │                                                          │
│              ▼ encrypt with server's RSA public key                     │
│   MeshPacket { packetId, ttl, createdAt, ciphertext }                   │
└──────────────────────────────────────┬──────────────────────────────────┘
                                       │ Bluetooth gossip
                                       ▼
        ┌─────────┐  hop   ┌─────────┐  hop   ┌─────────┐
        │stranger1│ ─────▶ │stranger2│ ─────▶ │ bridge  │ ◀── walks outside
        └─────────┘        └─────────┘        └────┬────┘     gets 4G
                                                   │
                                                   ▼ HTTPS POST
┌─────────────────────────────────────────────────────────────────────────┐
│                     SPRING BOOT BACKEND (this project)                  │
│                                                                         │
│  /api/bridge/ingest                                                     │
│       │                                                                 │
│       ▼                                                                 │
│  [1] hash ciphertext (SHA-256)                                          │
│       │                                                                 │
│       ▼                                                                 │
│  [2] IdempotencyService.claim(hash)  ◀── atomic putIfAbsent (≈ Redis    │
│       │                                  SETNX). Duplicates rejected    │
│       │                                  here, before any work.         │
│       ▼                                                                 │
│  [3] HybridCryptoService.decrypt(ciphertext)                            │
│       │       (RSA-OAEP unwraps AES key, AES-GCM decrypts payload       │
│       │        AND verifies the auth tag — tampering = exception)       │
│       ▼                                                                 │
│  [4] Freshness check: signedAt within last 24h                          │
│       │                                                                 │
│       ▼                                                                 │
│  [5] SettlementService.settle()                                         │
│       @Transactional: debit sender, credit receiver, write ledger       │
│       @Version on Account = optimistic locking (defense in depth)       │
└─────────────────────────────────────────────────────────────────────────┘
he three hard problems and how they're solved
Problem 1: Untrusted intermediates
A random stranger's phone is carrying your transaction. How do you stop them from reading the amount or changing it?

Solution: Hybrid encryption (RSA-OAEP + AES-GCM).

The sender encrypts the payload with the server's public key. Only the server holds the private key, so intermediates see opaque ciphertext.

But RSA can only encrypt small data (~245 bytes for a 2048-bit key), and our payload is JSON that could exceed that. So we use the standard hybrid pattern:

Generate a fresh AES-256 key for this packet.
Encrypt the JSON with AES-256-GCM (fast + authenticated).
Encrypt just the AES key with RSA-OAEP.
Concatenate: [256 bytes RSA-encrypted AES key][12 bytes IV][AES ciphertext + 16-byte GCM tag].
Why GCM specifically? It's authenticated encryption. If an intermediate flips one bit anywhere in the ciphertext, decryption throws an exception — the GCM tag won't verify. The server cannot be tricked into processing tampered data.

This is the same scheme TLS uses. See HybridCryptoService.java.

Problem 2: The duplicate-storm
Three bridge nodes hold the same packet. They all walk outside at the same instant. They all POST to /api/bridge/ingest within milliseconds of each other. If you naively process all three, the sender is debited ₹1500 instead of ₹500.

Solution: Atomic compare-and-set on the ciphertext hash.

The very first thing the server does on receiving a packet is compute SHA-256(ciphertext) and try to "claim" that hash:

// IdempotencyService.java
Instant prev = seen.putIfAbsent(packetHash, now);
return prev == null;  // true = first claimer, false = duplicate
ConcurrentHashMap.putIfAbsent is atomic. Even if 100 threads call it at the exact same nanosecond, exactly one returns null (the first claimer) and the rest return the existing entry. Only the first claimer proceeds to decrypt and settle. The rest are short-circuited as DUPLICATE_DROPPED.

Why hash the ciphertext, not the packetId or the cleartext?

packetId can be rewritten by a malicious intermediate. Two copies of the same payment could have different packetIds. Bad key.
The cleartext requires decryption first. We want to dedupe before spending CPU on RSA.
The ciphertext is authenticated by GCM, so any tampering is detectable on decrypt. Two legitimate deliveries of the same payment have byte-identical ciphertexts (AES is deterministic for a given key+IV+plaintext, and the same packet means the same key+IV+plaintext).
In production this ConcurrentHashMap becomes Redis: SET key NX EX 86400. Same semantics, distributed across replicas.

There's also a defense-in-depth fallback: transactions.packet_hash has a unique index. If the cache layer ever fails and two settlements somehow try to write the same hash, the database rejects the second one.

Problem 3: Replay attacks
An attacker who captured a ciphertext weeks ago could replay it whenever convenient.

Solution: Two layers.

Inside the encrypted payload, the sender includes signedAt (epoch millis). The server rejects any packet older than 24 hours. The attacker can't change signedAt without breaking the GCM tag.
Inside the encrypted payload, the sender includes a nonce (UUID). Even if Alice legitimately sends Bob ₹100 twice, the nonces differ → ciphertexts differ → hashes differ → both settle. But a replay of one specific signed packet is byte-identical, so the idempotency cache catches it.
See BridgeIngestionService.java for the freshness check.

File-by-file walkthrough
upi-offline-mesh/
├── pom.xml                                  Maven build, Spring Boot 3.3, Java 17
├── mvnw, mvnw.cmd                           Maven wrapper (no install needed)
├── README.md                                this file
└── src/main/
    ├── resources/
    │   ├── application.properties           H2 in-memory DB, port 8080, TTLs
    │   └── templates/dashboard.html         The interactive demo UI
    └── java/com/demo/upimesh/
        ├── UpiMeshApplication.java          Spring Boot main class
        │
        ├── model/                           ── Domain layer
        │   ├── Account.java                 JPA entity. @Version = optimistic lock
        │   ├── AccountRepository.java       Spring Data JPA
        │   ├── Transaction.java             Settled-tx ledger. unique idx on packetHash
        │   ├── TransactionRepository.java   Spring Data JPA
        │   ├── MeshPacket.java              Wire format. Outer fields readable, ciphertext opaque
        │   └── PaymentInstruction.java      Decrypted payload (sender/receiver/amount/nonce/time)
        │
        ├── crypto/                          ── Cryptography layer
        │   ├── ServerKeyHolder.java         Generates RSA-2048 keypair on startup
        │   └── HybridCryptoService.java     RSA-OAEP + AES-256-GCM encrypt/decrypt + ciphertext hash
        │
        ├── service/                         ── Business logic
        │   ├── DemoService.java             Seeds accounts, simulates a sender phone
        │   ├── VirtualDevice.java           One simulated phone in the mesh
        │   ├── MeshSimulatorService.java    Gossip protocol across virtual devices
        │   ├── IdempotencyService.java      ConcurrentHashMap = JVM-local Redis SETNX
        │   ├── SettlementService.java       @Transactional debit + credit + ledger insert
        │   └── BridgeIngestionService.java  THE pipeline: hash → claim → decrypt → freshness → settle
        │
        ├── controller/                      ── HTTP layer
        │   ├── ApiController.java           All REST endpoints
        │   └── DashboardController.java     Serves the dashboard HTML at /
        │
        └── config/
            └── AppConfig.java               @EnableScheduling for cache eviction

src/test/java/com/demo/upimesh/
└── IdempotencyConcurrencyTest.java          The 3-bridges-at-once test + tamper test
API reference
Method	Path	What it does
GET	/	Dashboard HTML
GET	/api/server-key	Server's RSA public key (base64)
GET	/api/accounts	All accounts and balances
GET	/api/transactions	Last 20 transactions
GET	/api/mesh/state	Current state of every virtual device
POST	/api/demo/send	Simulate sender phone — encrypt + inject packet
POST	/api/mesh/gossip	Run one round of gossip across the mesh
POST	/api/mesh/flush	Bridges with internet upload to backend (parallel)
POST	/api/mesh/reset	Clear mesh + idempotency cache
POST	/api/bridge/ingest	The production endpoint. Real bridges POST here
GET	/h2-console	Browse the in-memory database
