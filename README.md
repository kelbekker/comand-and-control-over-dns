# DNS-based Command and Control (C&C) - Educational Lab

> **Disclaimer:** This project is created **strictly for educational purposes** and to demonstrate network security concepts (DNS tunneling and covert communication) within an isolated laboratory environment. Do not deploy or use this tool on public networks or systems you do not own. Misuse of this software may be illegal and could be flagged as malicious activity.
> 

This project demonstrates a **DNS-based command-and-control (C&C)** communication channel between a "victim" host and an "attacker" host using Python and the **Scapy** library.

## 🏛 Architecture

```
[ Victim (Private Network) ]
       │
       │ DNS Queries (Data exfiltration, Beacons)
       ▼
[ Attacker (DNS Server / Responder) ]
```

- **Victim:** Sends Base32-encoded DNS queries to the attacker's server to signal readiness, transfer data, and request commands.
- **Attacker:** Listens for DNS queries and responds with commands or acknowledgments encoded in DNS **TXT records**.

## ✨ Features (Lab Only)

- **Remote Shell Execution:** Run commands (`ls`, `dir`, etc.) on the victim host.
- **File Transfer:** Exfiltrate files from the victim host to the attacker server over DNS.
- **Process Control:** Execute files or terminate the victim client remotely (`Exit`).
- **Beaconing Mechanism:** Periodic signal to maintain connection and receive new commands.

## ⚙️ How It Works

### Victim

1. Encodes payloads (commands, status, file chunks) in **Base32**.
2. Sends messages via **DNS queries** directed at the attacker’s controlled domain.
3. Decodes and executes received commands:
    - `Run <command>` → Executes locally and sends output back over DNS.
    - `Send <filename>` → Reads file contents and transfers them in chunks.
    - `Exit` → Safely terminates the victim process.
4. Handles payload chunking to respect DNS subdomain limits (**max 63 bytes** per label).

### Attacker

1. Sniffs incoming DNS requests from the victim using Scapy.
2. Decodes **Base32** payloads extracted from DNS subdomains.
3. Responds via **DNS TXT records** containing encoded commands or acknowledgments.
4. Handles different packet types:
    - `BEACON` → Initial victim registration signal.
    - `CHUNK` → Data fragment (command output or file content).
    - `EOO` → End of Output marker.
    - `EOF` → End of File transfer marker.

## 🛠 File Structure

| File | Description |
| --- | --- |
| `attacker_dns.py` | DNS responder; listens for queries, issues commands, and processes exfiltrated data. |
| `victim_dns.py` | Victim client; sends beacons, executes local commands, and exfiltrates data over DNS. |
| `README.md` | Project documentation. |

## ⚙️ Configuration

### 1. Domain & DNS Setup

To route DNS traffic through public servers to your lab, set up NS and A records (e.g., in GoDaddy, Route 53, etc.):

- **A Record:** Points your hostname to the attacker server's IP address.
- **NS Record:** Delegates subdomains to your attacker's server IP.

**Example Setup:**

```
TYPE: NS  | NAME: work | DATA: work.yourdomain.com
TYPE: A   | NAME: work | DATA: <Attacker_Public_IP>
```

**Verification:**

```
nslookup test.work.yourdomain.com
```

### 2. Script Settings

Update configuration variables inside both `attacker_dns.py` and `victim_dns.py`:

- `DNS_SERVER`: IP address of the attacker's DNS server.
- `INTERFACE`: Network interface to sniff/send raw packets (e.g., `eth0`, `enp8s0f1`).
- `PORT`: DNS service port (default: `53`).
- `URL`: Controlled domain used for tunneling (e.g., `work.yourdomain.com`).

### 3. Protocol Chunking Limits

- **Subdomain Query Limit:** Max `63 bytes` per label.
- **TXT Record Limit:** Max `255 bytes` per record.

## 🔄 Data Flow

1. **Beaconing:** Victim sends a `BEACON` query to notify availability.
2. **Command Issue:** Attacker captures query and replies with a TXT record containing an encoded command.
3. **Execution:** Victim decodes and runs the command locally.
4. **Exfiltration:** Output is broken into Base32 chunks and sent back as DNS queries.
5. **Reassembly:** Attacker stitches chunks together until an `EOO` or `EOF` flag is received.

## 🚀 Running the Lab

> **Note:** Root privileges (`sudo`) are required on both hosts to construct raw DNS packets and sniff traffic.
> 

### 1. Start the Attacker DNS Server:

```
sudo python3 attacker_dns.py
```

### 2. Start the Victim Client:

```
sudo python3 victim_dns.py
```

## 💡 Practical Lab Notes

- **Keep Listener Active:** Always keep `attacker_dns.py` running before starting the victim, as public/intermediate recursive DNS servers will return errors if your listener doesn't respond.
- **Static IP:** When running in cloud providers like AWS, assign an **Elastic IP** to maintain a persistent IP address.
- **Traffic Analysis:** Inspect packet encapsulation using `tcpdump` or Wireshark during tests:
sudo tcpdump -i <INTERFACE> port 53
