# Week 1 CN Notes (OSI, TCP IP, TCP vs UDP)

# Week 1 CN Notes OSI, TCP IP, TCP vs UDP (for Data Engineers)

# Week 1 CN Notes: OSI, TCP/IP, TCP vs UDP

Computer Networks for Data Engineers. Roadmap Part 5, Week 1 (1 hr/day). Covers the 3 Week 1 topics in full, with DE context, numericals, commands, interview Q&A and a 7-day plan.

## 0. Why a Data Engineer needs this

Almost everything you build moves data over a network:

- Python or Spark connecting to MySQL/Postgres/Redshift (JDBC/ODBC) uses TCP (ports 3306, 5432, 5439).
- Reading and writing S3/GCS/ADLS is HTTPS (TCP port 443).
- Kafka producers and consumers talk to brokers over TCP (port 9092).
- Spark driver and executors talk by RPC over TCP, and shuffles move data between nodes over the network.
- Airflow UI (8080), Spark UI (4040), Jupyter (8888) are just TCP ports.
- ‘Connection timed out’, ‘Connection refused’, ‘Name or service not known’ are network errors you will debug weekly.

Goal of this week: explain what happens from the moment your code says connect() to the moment bytes arrive, and debug it when it fails.

---

## 1. OSI model (7 layers)

OSI = Open Systems Interconnection. A reference model that splits network communication into 7 layers. Each layer only talks to the layer above and below it, and adds or removes its own header.

**Layer 7: Application** (the app talks)

- Job: the program you use asks for something over the network.
- Example: your browser sends an HTTP request, pandas calls an API, DBeaver runs a SQL query.
- Protocols: HTTP, HTTPS, DNS, FTP, SMTP, SSH.
- Parcel analogy: you write the letter.

**Layer 6: Presentation** (translate and secure)

- Job: puts data in a format both sides understand, and encrypts or compresses it.
- Example: TLS encrypts your data (the “S” in HTTPS), UTF-8 text encoding, gzip compression.
- Parcel analogy: you translate the letter into a common language and lock it in a safe.

**Layer 5: Session** (manage the conversation)

- Job: starts, keeps alive and ends the conversation between two machines.
- Example: a database login session stays open while you run many queries, then closes.
- Parcel analogy: the phone call that says “I’m about to send you something… done, thanks.”

**Layer 4: Transport** (right program, safely)

- Job: delivers data to the correct *program* on the other machine using **port numbers**, and decides how reliable delivery is (TCP or UDP). It splits big data into **segments**.
- Example: port 5432 means PostgreSQL, port 443 means HTTPS.
- Parcel analogy: splitting the contents into numbered boxes, and writing the apartment number on each.

**Layer 3: Network** (right machine, across networks)

- Job: finds the path to the destination machine using **IP addresses**. Data is now called a **packet**. Routers work here.
- Example: 192.168.1.10, ping.
- Parcel analogy: the street address, and the post offices choosing the route.

**Layer 2: Data Link** (next hop on the same network)

- Job: moves data between two devices directly connected on the same local network using **MAC addresses**. Data is now a **frame**. Switches work here.
- Example: your laptop to your Wi-Fi router.
- Parcel analogy: the delivery van driving one leg of the route.

**Layer 1: Physical** (actual signals)

- Job: sends raw 1s and 0s as electrical pulses, light or radio waves.
- Example: Ethernet cable, fibre, Wi-Fi radio.
- Parcel analogy: the road itself.

Memory trick (top to bottom): All People Seem To Need Data Processing. Bottom to top: Please Do Not Throw Sausage Pizza Away.

| # | Layer | Job | PDU (data unit) | Examples / protocols | Devices |
| --- | --- | --- | --- | --- | --- |
| 7 | Application | Interface for apps and services | Data | HTTP, HTTPS, FTP, SMTP, DNS, SSH | L7 load balancer, WAF, proxy |
| 6 | Presentation | Format, encryption, compression | Data | TLS/SSL, JSON/UTF-8 encoding, gzip | - |
| 5 | Session | Open, maintain, close sessions | Data | Sessions, RPC, sockets API | - |
| 4 | Transport | End-to-end delivery, ports, reliability | Segment (TCP) / Datagram (UDP) | TCP, UDP | Firewall, L4 load balancer |
| 3 | Network | Logical addressing and routing across networks | Packet | IP, ICMP, routing | Router |
| 2 | Data Link | Delivery between devices on the same network (MAC) | Frame | Ethernet, Wi-Fi, ARP | Switch, NIC |
| 1 | Physical | Raw bits on the medium | Bits | Cables, fibre, radio signals | Hub, repeater |

### IMPORTANT:-

Two points to remember:

- Going down the stack, each layer adds its **header** (encapsulation). Going up, each layer removes it.
- **MAC changes at every hop; IP stays the same end to end.**

### Addressing at each layer

| Layer | Address | Example |
| --- | --- | --- |
| L2 | MAC (48-bit) | A4:5E:60:C1:22:9B |
| L3 | IP (IPv4 32-bit, IPv6 128-bit) | 192.168.1.10 |
| L4 | Port (16-bit) | 5432 |
| L7 | Name / URL | db.company.com, [https://api.x.com/v1](https://api.x.com/v1) |

### Encapsulation and decapsulation

Sending: data goes DOWN the stack and each layer wraps it with its header.

```
App data  ->  [TCP hdr | data] (segment)  ->  [IP hdr | segment] (packet)  ->  [Eth hdr | packet | Eth trailer] (frame)  ->  bits
```

Receiving: the reverse (decapsulation). Each layer strips its header and passes the rest up.

Which header gets rewritten at each router hop? **L2 MAC addresses change every hop; L3 IP addresses stay the same end to end** (ignoring NAT). This is a very common interview question.

### Which layer does it? (Quick map)

| Thing | Layer |
| --- | --- |
| Cable, Wi-Fi signal | 1 |
| Switch, MAC address, ARP, Ethernet | 2 |
| Router, IP address, ICMP (ping) | 3 |
| TCP, UDP, port numbers | 4 |
| TLS/SSL, encryption | 6 (or 4-7 boundary) |
| HTTP, DNS, FTP, SMTP, SSH | 7 |
| Load balancer (ALB) | 7; NLB is 4 |
| Security group / firewall rule on port | 3-4 |

---

## 2. TCP/IP model (4 layers) and where HTTP, TCP and IP sit

OSI is the teaching model. **TCP/IP is the model the internet actually uses.** It collapses OSI into 4 layers.

| TCP/IP layer | Maps to OSI | Protocols | Unit |
| --- | --- | --- | --- |
| Application | 5, 6, 7 | HTTP, HTTPS, DNS, FTP, SMTP, SSH, TLS | Data / message |
| Transport | 4 | TCP, UDP | Segment / datagram |
| Internet | 3 | IP (IPv4/IPv6), ICMP, ARP-ish | Packet |
| Network Access (Link) | 1, 2 | Ethernet, Wi-Fi | Frame / bits |

**Where they sit:**

- **HTTP** = Application layer (TCP/IP L4-top, OSI L7). HTTP/1.1 and HTTP/2 run over TCP. HTTP/3 runs over QUIC, which runs over **UDP**.
- **TCP** = Transport layer (OSI L4).
- **IP** = Internet layer (OSI L3).

### What happens when you run

`requests.get('https://api.example.com/data')`

1. **Find the address.** You gave a name (`api.example.com`), but computers need a number. A lookup service (DNS) converts the name into an IP address, like looking up a phone number.
**DNS** lookup: api.example.com becomes an IP (UDP port 53 usually).
2. **Say hello.** Your computer and the server exchange 3 quick messages to confirm they can hear each other (the TCP handshake).
**TCP 3-way handshake** to that IP on port 443.
3. **Lock the line.** They agree on a secret code so nobody in between can read the data (TLS, the “S” in HTTPS).
**TLS handshake**: agree on encryption, verify the certificate.
4. **Send your request.** Your “give me /data” message goes out (HTTP).
**HTTP request** sent (encrypted) over the TCP connection.
5. **Travel.** The message is cut into small pieces and passed through many routers until it reaches the server.
Data is split into segments, wrapped in IP packets, wrapped in frames, sent hop by hop through routers.
6. **Get the reply.** The server sends data back, and your computer puts the pieces back in order.
Server replies; the stack reassembles the bytes in order for your app.
7. **Hang up or stay connected.** The connection either closes or stays open for the next request.
Connection closed (FIN) or kept alive for reuse.

### IP essentials (the Internet layer)

### The basics you need about IP addresses and ports

- **IPv4 address:** 32 bits, written as 4 numbers from 0 to 255, like 192.168.1.10. That gives about 4.3 billion addresses. **IPv6** has 128 bits and was created because IPv4 is running out.
- **Private IP ranges** (used inside offices, homes and cloud networks, not reachable from the internet): 10.x.x.x, 172.16.x.x to 172.31.x.x, 192.168.x.x. Your database server is often on one of these.
- A port is a logical identifier used to distinguish different applications or services on a device, allowing network traffic to reach the correct program.
When a device communicates over a network, data packets are sent to its IP address. Each packet also includes a port number, which tells the operating system which application or service should receive that data.

- **Port number:** 16 bits, so from 0 to 65535. It identifies the program on a machine.
    - 0 to 1023: well-known ports (HTTP 80, HTTPS 443, SSH 22).
    - 1024 to 49151: registered ports (MySQL 3306, PostgreSQL 5432).
    - 49152 to 65535: temporary ports your machine picks for itself when it connects out.

- **TCP and UDP protocols use ports differently:** TCP ensures reliable delivery, while UDP is faster but without guaranteed delivery.

- **Socket:** an IP address plus a port, like 10.0.1.5:5432. A TCP connection is identified by four things together: your IP, your port, the server IP, the server port. That is why one server port can serve thousands of clients at once.
- **IP + Port + Protocol(TCP/UDP) = Socket:** This combination ensures that network traffic is delivered precisely to the right process on the right device.

- **MTU:** the biggest chunk a network link can carry in one go. On Ethernet it is **1500 bytes**. Big data is cut to fit inside this limit. You will use this number in the problems.

**Ports every DE should know:**

| Port | Service | Port | Service |
| --- | --- | --- | --- |
| 22 | SSH / SFTP | 5432 | PostgreSQL |
| 53 | DNS | 5439 | Redshift |
| 80 | HTTP | 6379 | Redis |
| 443 | HTTPS | 8080 | Airflow UI / common web alt |
| 3306 | MySQL | 9092 | Kafka broker |
| 1433 | SQL Server | 2181 | ZooKeeper |
| 1521 | Oracle | 27017 | MongoDB |
| 4040 | Spark UI | 8888 | Jupyter |
| 9200 | Elasticsearch | 7077 | Spark master |

---

## 3. TCP vs UDP (the main topic)

Both live at the **Transport layer** and both use ports to reach the right program. The difference is the promise they make. TCP promises careful, correct delivery. UDP promises nothing and is faster.

### TCP (Transmission Control Protocol)

Think of TCP as a **phone call**: you connect first, you confirm the other side hears you, and you repeat anything that was missed.

**1. It connects first (3-way handshake).**

```markdown
Client                          Server
  |  --- SYN (seq = x) ------->   |   Hello, can we talk? My starting number is x.
  |  <-- SYN-ACK (seq = y,        |   Yes. Got your x. My starting number is y.
  |        ack = x + 1) -------   |
  |  --- ACK (ack = y + 1) --->   |   Got your y. Connection is open.
```

- **SYN** = “synchronize”. It means “I want to start a connection.”
- **ACK** = “acknowledge”. It means “I got your message.”
- **SYN-ACK** = both at once: “I got your request, and I’m ready too.”
- **seq** = sequence number, the number a side starts counting its data from. Each side picks its own (x for the client, y for the server).
- **ack = x + 1** means “I got your x, and the next number I expect from you is x + 1.”

Three steps (not two) make sure **both** sides know the other can send and receive, and agree on starting numbers. Cost: **1 round trip (RTT)** before any data moves.

**2. It is reliable.** Every segment has a **sequence number**. The receiver sends back an **ACK** saying what it got. If an ACK does not come back in time, the sender sends that segment again (**retransmission**). A **checksum** in the header catches corrupted data.

**3. It keeps order.** Sequence numbers let the receiver put segments back in the right order and throw away duplicates, even if they arrive jumbled.

**4. Flow control (protect the receiver).** The receiver tells the sender how much it can hold right now (the **window size**). A fast sender does not flood a slow receiver.

**5. Congestion control (protect the network).** TCP starts sending slowly (**slow start**), speeds up while all is fine, and slows down when it detects loss. This stops the network from collapsing.

**6. It closes politely (4 steps).** FIN, ACK, FIN, ACK. The side that closes first waits in a state called **TIME_WAIT** for about a minute so late packets do not confuse a new connection.

**7. It sends a stream of bytes.** TCP has no idea where one message ends and the next begins. If you send three messages, the receiver may get them as one chunk. Applications add their own markers (like a newline or a length) to tell messages apart.

**TCP header:** at least **20 bytes**. It holds source port, destination port, sequence number, acknowledgement number, flags (SYN, ACK, FIN, RST), window size and checksum.

### UDP (User Datagram Protocol)

Think of UDP as **posting a postcard**: you write the address, drop it in the box and walk away. No call, no confirmation, no resend.

- **No handshake.** It just sends.
- **No guarantee.** A datagram can be lost, duplicated, or arrive out of order. The program must handle that if it cares.
- **No flow or congestion control.**
- **Each datagram stands alone** and keeps its boundaries (one send equals one receive).
- **Tiny header: 8 bytes only.** It has just 4 fields: source port, destination port, length, checksum (2 bytes each).

### Side by side

| Feature | TCP | UDP |
| --- | --- | --- |
| Connection | Yes (handshake) | No |
| Reliability | Guaranteed delivery, retransmit | Best effort |
| Ordering | Yes | No |
| Flow and congestion control | Yes | No |
| Header size | 20-60 bytes | 8 bytes |
| Speed / overhead | Slower, heavier | Faster, lighter |
| Data unit | Byte stream (no message boundaries) | Datagrams (boundaries kept) |
| Use when | Correctness matters | Latency matters more than perfection |
| Examples | HTTP/HTTPS, SSH, SMTP, FTP, DB protocols, Kafka | DNS queries, DHCP, video/voice streaming, online games, NTP, QUIC/HTTP3, syslog, SNMP |

### Why databases use TCP

A SQL query, a transaction commit or a row batch must arrive **complete, in order, exactly once**. A missing byte means a corrupt result or a wrong ledger. TCP gives that for free, so database drivers (MySQL, Postgres, JDBC) and Kafka’s protocol all sit on TCP.

### Why streaming/video uses UDP

A late frame is useless. Waiting to retransmit a lost packet would freeze the video; it is better to skip it and show the next frame. Live calls and games care about **low latency**, and tolerate small loss. UDP also avoids TCP’s head-of-line blocking (one lost segment blocking everything behind it).

### Trick questions

- **DNS uses UDP 53 for normal queries, but TCP 53** for large responses (over 512 bytes classically) and zone transfers.
- **HTTP/3 uses UDP** (via QUIC), which rebuilds reliability in user space and avoids TCP head-of-line blocking.
- **‘Streaming’ in DE (Kafka, Spark Structured Streaming) is NOT UDP.** It means continuous data processing and still uses TCP underneath. Do not confuse video streaming with data streaming.
- TCP is a **byte stream**: two `send()` calls may arrive as one `recv()` (no message boundaries). Applications add framing (length prefix, newline).

---

## 4. Data Engineering scenarios (apply the theory)

| Scenario | What is happening | Layer / concept |
| --- | --- | --- |
| `psycopg2.connect()` hangs then ‘timed out’ | Packets are silently dropped: security group / firewall / NACL blocks 5432, or wrong route | L3-L4, firewall |
| ‘Connection refused’ | Host reachable but nothing listening on that port (DB down, wrong port), server sent RST | L4 |
| ‘Name or service not known’ | DNS cannot resolve the hostname | L7 DNS |
| Cross-region S3 copy is slow on 10 Gbps link | High RTT with window-limited TCP (BDP) | L4 |
| Millions of tiny S3 requests slow | Latency bound, per-request overhead (RTT + TLS + HTTP) | Latency vs throughput |
| Spark job slow in shuffle | Network throughput between executors, cross-AZ traffic | L4 throughput |
| App opens/closes DB connection per query | Handshake cost, TIME_WAIT pile-up, port exhaustion | TCP handshake |
| Server runs out of file descriptors / ephemeral ports | Too many connections or leaked sockets (CLOSE_WAIT) | Sockets |
| Kafka consumer keeps disconnecting | Idle connections dropped by firewall/NAT; tune keep-alive and timeouts | TCP keep-alive |
| Live dashboard needs sub-second updates | Choose push (WebSocket / streaming) over polling | L7 |

**Timed out vs refused (high-frequency interview + real-life question):**

- **Refused** = you reached the machine; nobody home on that port. Check the service and port.
- **Timed out** = you never got an answer. Check security groups, firewall, route tables, private vs public subnet, VPN.

---

## 5. Hands-on commands (practise these this week)

```bash
ping google.com                 # ICMP, checks reachability + RTT (L3)
traceroute google.com           # shows router hops (uses TTL); 'tracert' on Windows
nslookup google.com             # DNS lookup  (also: dig google.com)
curl -v https://example.com     # shows DNS, TCP connect, TLS handshake, HTTP request/response
ss -tulpn                       # listening TCP/UDP ports + owning process (Linux)
netstat -an                     # all connections + states (ESTABLISHED, TIME_WAIT...)
nc -zv hostname 5432            # test whether a TCP port is open (nc = netcat)
telnet hostname 5432            # older way to test a TCP port
ifconfig / ip addr              # your IP and MAC addresses
arp -a                          # IP to MAC table (L2)
```

Exercises: run `curl -v https://example.com` and label each line with its layer. Run `ss -tan` and find ESTABLISHED and LISTEN. Use `nc -zv localhost 3306` before and after starting MySQL.

**Optional: Wireshark.** Capture a `curl http://example.com`, filter on `tcp`, and find the SYN, SYN-ACK, ACK packets and the sequence numbers.

---

## 6. Python: see TCP vs UDP yourself

TCP server and client:

```python
# tcp_server.py
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)   # SOCK_STREAM = TCP
s.bind(('127.0.0.1', 5050))
s.listen()
conn, addr = s.accept()          # 3-way handshake completes here
print('connected:', addr)
print(conn.recv(1024).decode())  # TCP is a byte stream
conn.sendall(b'hello from server')
conn.close()

# tcp_client.py
import socket
c = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
c.connect(('127.0.0.1', 5050))   # SYN, SYN-ACK, ACK
c.sendall(b'hello from client')
print(c.recv(1024).decode())
c.close()
```

UDP server and client (no connect, no accept):

```python
# udp_server.py
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)    # SOCK_DGRAM = UDP
s.bind(('127.0.0.1', 5051))
data, addr = s.recvfrom(1024)    # one datagram, boundaries preserved
print(data.decode(), 'from', addr)

# udp_client.py
import socket
c = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
c.sendto(b'hello udp', ('127.0.0.1', 5051))   # fire and forget, no guarantee
```

Things to notice: UDP has no `listen`/`accept`/`connect` handshake; if the server is not running the UDP client still ‘succeeds’ silently, while the TCP client raises ConnectionRefusedError.

Challenge: make the TCP client send 3 separate `sendall` calls and see how many `recv` calls the server needs (it can be fewer: byte stream, no boundaries).

---

## 7. Practice problems (with answers)

Try each one on paper first, then read the answer.

**Problem 1. Which layer?** Give the OSI layer number for: (a) router, (b) switch, (c) HTTP, (d) TLS encryption, (e) port numbers, (f) MAC address, (g) ping.

- **Answer:** (a) 3, (b) 2, (c) 7, (d) 6, (e) 4, (f) 2, (g) 3 (ping uses ICMP, which belongs to the network layer).

**Problem 2. Data units.** What is the data unit called at Layer 4 (TCP), Layer 3 and Layer 2?

- **Answer:** Layer 4 = segment (TCP) or datagram (UDP), Layer 3 = packet, Layer 2 = frame.

**Problem 3. Header overhead on TCP.** You send 1000 bytes of data over TCP/IP. The IP header is 20 bytes and the TCP header is 20 bytes. What share of what you send is real data? Then repeat with only 10 bytes of data.

- **Answer:** Headers = 20 + 20 = 40 bytes.
- 1000 bytes: 1000 / 1040 = **96.15%** is data.
- 10 bytes: 10 / 50 = **20%** is data, and 80% is wasted on headers.
- **Lesson:** tiny messages waste a lot on headers. This is one reason data engineers send data in batches, not one row at a time.

**Problem 4. Header overhead on UDP.** Same 1000 bytes, but over UDP (8-byte header) and IP (20 bytes). What share is data?

- **Answer:** Headers = 20 + 8 = 28 bytes. 1000 / 1028 = **97.28%**. UDP is a little lighter than TCP.

**Problem 5. MSS and number of segments.** MTU is 1500 bytes. The MSS (the biggest data chunk per TCP segment) is the MTU minus the IP header and TCP header. (a) Find the MSS. (b) How many segments to send a 10 MB file (10,000,000 bytes)? (c) How many for 1,000,000 bytes?

- **Answer (a):** MSS = 1500 - 20 - 20 = **1460 bytes**.
- **Answer (b):** 10,000,000 / 1460 = 6849.3, so round up to **6850 segments**. The last segment carries only 460 bytes (6849 full segments carry 9,999,540 bytes).
- **Answer (c):** 1,000,000 / 1460 = 684.9, so **685 segments**.

**Problem 6. Handshake and sequence numbers.** The client starts with number 1000 and the server with 5000. After the handshake, the client sends 300 bytes, and the server answers with 500 bytes. Write each packet.

- **Answer:**
    1. Client to server: SYN, seq = 1000.
    2. Server to client: SYN-ACK, seq = 5000, ack = 1001.
    3. Client to server: ACK, seq = 1001, ack = 5001. Connection is open.
    4. Client sends 300 bytes: seq = 1001. The next byte it will send is 1001 + 300 = 1301.
    5. Server acknowledges: ack = **1301**.
    6. Server sends 500 bytes: seq = 5001. Client acknowledges with ack = 5001 + 500 = **5501**.
- **Rule:** the ACK number is always the next byte number you expect. A SYN counts as 1 byte, which is why 1000 becomes 1001.

**Problem 7. The cost of opening connections.** The round trip time (RTT) to a remote database is 40 ms. A script runs 500 queries and opens a new TCP connection for every query. How much time is spent only on TCP handshakes? Compare it with one reused connection.

- **Answer:** A TCP handshake costs 1 RTT = 40 ms. New connection each time: 500 x 40 ms = 20,000 ms = **20 seconds** wasted. One reused connection: **40 ms** once.
- **Lesson:** this is why we use connection pooling and keep connections open.

**Problem 8. TCP or UDP?** Pick one for each, and say why: (a) loading bank transactions into a database, (b) a live cricket match stream, (c) a DNS lookup, (d) a Kafka producer sending events, (e) a multiplayer online game, (f) uploading a file to S3.

- **Answer:**
    - 
        1. TCP: every byte must arrive, in order.
    - 
        1. UDP: a late frame is useless, a small glitch is fine.
    - 
        1. UDP: tiny question and answer, speed matters (falls back to TCP for large answers).
    - 
        1. TCP: events must not be lost or reordered.
    - 
        1. UDP: low delay matters more than a lost update.
    - 
        1. TCP: HTTPS runs on TCP, and a corrupted file is not acceptable.

**Problem 9. What happens when a packet is lost?** 10 packets are sent and packet 4 is lost on the way. Describe what happens with TCP and with UDP.

- **Answer (TCP):** the receiver notices that number 4 is missing and keeps asking for it (or the sender times out). The sender **sends packet 4 again**. The receiver puts it in order, and the app gets all 10 packets, in order. Delivery takes a bit longer.
- **Answer (UDP):** nobody notices. The receiver gets 9 packets and the app continues. Fast, but one piece is permanently missing.

**Problem 10. What changes along the way?** A laptop sends data to a server through 3 routers (laptop, R1, R2, R3, server). How many links does the data cross, and which addresses change?

- **Answer:** There are **4 links** (laptop to R1, R1 to R2, R2 to R3, R3 to server), so the data travels in 4 different frames. On each link the **MAC addresses change** (a new sender and receiver pair). The **source IP and destination IP stay the same** the whole way, and so do the ports.

## 8. Basic interview questions with answers

**Q1. What is the OSI model and why do we use it?** It splits network communication into 7 layers: Physical, Data Link, Network, Transport, Session, Presentation, Application. Each layer has one job, so engineers can understand the system in parts and find which layer a problem is in.

**Q2. Name the 7 layers in order.** From the top: Application, Presentation, Session, Transport, Network, Data Link, Physical (All People Seem To Need Data Processing).

**Q3. What is the difference between the OSI and TCP/IP models?** OSI is a 7-layer theory model used for teaching. TCP/IP is the 4-layer model the internet really uses: Application, Transport, Internet, Network Access. OSI layers 5 to 7 become TCP/IP Application, and OSI layers 1 and 2 become Network Access.

**Q4. At which layer do HTTP, TCP and IP work?** HTTP at the Application layer (OSI 7), TCP at the Transport layer (OSI 4), IP at the Internet layer (OSI 3).

**Q5. What is encapsulation?** As data moves down the layers, each layer adds its own header around it (data, then segment, then packet, then frame). The receiver removes the headers in reverse order. It is like putting a letter in an envelope, then in a box.

**Q6. What is a header?** Extra control information a layer adds at the front of the data, such as ports and sequence numbers in TCP, or source and destination addresses in IP.

**Q7. What is the difference between a MAC address and an IP address?** A MAC address identifies a network card on the local network, works at Layer 2 and changes at every hop. An IP address identifies a machine across networks, works at Layer 3, and stays the same from sender to receiver.

**Q8. What is a port and what is a socket?** A port (0 to 65535) identifies a program on a machine, for example 5432 for PostgreSQL. A socket is an IP address plus a port.

**Q9. What is the difference between TCP and UDP?** TCP is connection-based and reliable: it uses a handshake, acknowledgements, resends lost data, keeps order, and controls flow and congestion. UDP is connectionless and fast: no handshake, no guarantee, no ordering, and an 8-byte header. Use TCP when correctness matters and UDP when low delay matters more than a few lost packets.

**Q10. Why do databases use TCP, but video streaming uses UDP?** Databases need every byte exactly once and in order, which TCP guarantees. In live video, a late packet is useless, and waiting to resend it would freeze the video, so UDP drops it and moves on.

**Q11. Explain the TCP 3-way handshake. Why three steps?** The client sends SYN, the server replies SYN-ACK, the client sends ACK. Three steps confirm that both sides can send and receive, and let them agree on starting sequence numbers. After this, the connection is open.

**Q12. How does TCP make delivery reliable?** Sequence numbers, acknowledgements, resending data that was not acknowledged, and a checksum to detect corruption. It also uses flow control (to protect the receiver) and congestion control (to protect the network).

**Q13. What is the difference between flow control and congestion control?** Flow control stops a fast sender from overwhelming a slow **receiver** (using the window size). Congestion control stops senders from overloading the **network**.

**Q14. Do Kafka and Spark Streaming use UDP because they are streaming?** No. They use TCP. Streaming in data engineering means processing data continuously, not the UDP protocol. Only live video, voice and similar use UDP.

**Q15. What happens when you call an API from Python?** DNS finds the IP address, a TCP connection is opened (3-way handshake), a TLS handshake sets up encryption for HTTPS, the HTTP request is sent, the server replies, and the connection is closed or reused.

**Q16. Why is connection pooling useful?** Opening a new TCP connection costs at least one round trip each time (more with TLS and login). A pool reuses open connections, so many queries share one setup cost.

---

## 9. One-page cheat sheet

- OSI: Physical, Data Link, Network, Transport, Session, Presentation, Application (1 to 7).
- PDU: bits, frame, packet, segment/datagram, data.
- TCP/IP: Application, Transport, Internet, Network Access.
- HTTP = L7, TCP/UDP = L4, IP = L3, Ethernet/MAC = L2, cable = L1.
- MAC changes every hop, IP stays the same (without NAT).
- TCP: handshake, reliable, ordered, flow + congestion control, 20-byte header. UDP: no connection, no guarantee, 8-byte header.
- Handshake = SYN, SYN-ACK, ACK. Close = FIN, ACK, FIN, ACK.
- MSS = MTU - 40 = 1460. MTU = 1500.
- Transfer time = (bytes x 8) / bits per second.
- BDP = bandwidth x RTT. Window-limited throughput = window / RTT.
- Hosts in subnet = 2^(32 - prefix) - 2 (AWS: minus 5).
- Refused = no listener. Timed out = blocked/dropped. Name not known = DNS.
- Streaming data (Kafka/Spark) runs on TCP, video streaming on UDP/QUIC.

---

##