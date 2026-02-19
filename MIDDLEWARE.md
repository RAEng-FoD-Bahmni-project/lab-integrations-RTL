# HL7 Lab Middleware — Technical Documentation

## Overview

**captureHl7Messages** is a Java middleware service that bridges laboratory analyzer machines with the Bahmni/OpenELIS hospital information system. It listens for HL7 (Health Level 7) messages transmitted over TCP sockets from lab analyzers, parses the results, and writes them directly into the OpenELIS PostgreSQL database.

### What It Does

```
┌──────────────┐     HL7/TCP      ┌─────────────────────┐     SQL/JDBC     ┌──────────────┐
│ Lab Analyzer  │ ──────────────▶ │  captureHl7Messages  │ ──────────────▶ │  OpenELIS DB  │
│ (e.g. XP-300) │   (socket)      │     (middleware)      │  (PostgreSQL)   │   (Bahmni)    │
└──────────────┘                  └─────────────────────┘                  └──────────────┘
```

**Flow:**
1. Lab analyzer completes a test and transmits results as an HL7 message over TCP
2. Middleware receives the message, extracts the sample ID and test results using configurable regex patterns
3. Middleware checks the OpenELIS database for a matching pending sample
4. If found, it inserts the results, creates result signatures, updates the analysis status, and marks the sample as processed
5. Raw HL7 messages are saved to disk as backup files

---

## Architecture

### Class Structure

| Class | Role |
|-------|------|
| **Main** | Entry point. Reads analyzer configurations from JSON, initializes the PostgreSQL connection pool, and spawns one `WorkingThread` per analyzer. |
| **WorkingThread** | Core processing loop. Opens a `ServerSocket` on the analyzer's configured port, accepts connections, parses HL7 messages using regex, and executes database operations. |
| **Analyzer** | Configuration POJO for a single lab analyzer. Holds port, IP, regex patterns, sample ID prefix, output path, ACK settings, and methods for sample-end detection. |
| **DataBaseUtility** | Singleton PostgreSQL connection pool manager using `PGPoolingDataSource`. Reads DB config from `openelisDB.json`. |
| **PostgresDBConnectionInfo** | POJO for database connection parameters (server IP, port, DB name, credentials, max connections). |
| **Result** | Simple data class holding a parsed test result: test name, value, and abnormality flag. |

### Dependencies

| Library | Version | Purpose |
|---------|---------|---------|
| Jackson Databind | 2.16.1 | JSON parsing for config files |
| Jackson Annotations | 2.16.1 | JSON annotations |
| Jackson Core | 2.16.1 | Core JSON processing |
| PostgreSQL JDBC | — | Database connectivity to OpenELIS |
| Log4j API | 2.23.0 | Logging framework |
| Log4j Core | 2.23.0 | Logging implementation |

---

## Configuration

### Analyzer Configuration (JSON)

The middleware reads analyzer definitions from a JSON array. Each analyzer entry specifies:

```json
[
  {
    "name": "SYSMIX",
    "port": 9100,
    "ip": "192.168.1.50",
    "prefix": "300",
    "outputPath": "/var/log/hl7/SYSMIX/",
    "sampleIDLineRegex": "O\\|\\d+\\|\\|\\^\\^\\s+(\\d+)",
    "valueLineRegex": "R\\|\\d+\\|\\^\\^\\^\\^(\\w+)\\^\\d+\\|\\s*([\\d.]+)",
    "testNameGroupNum": 1,
    "testValueGroupNum": 2,
    "flagGroupNum": -1,
    "expectedNumOfLines": 0,
    "lastLineRegex": "L\\|1\\|N",
    "highExp": "H",
    "lowExp": "L",
    "nonPrintableStop": false,
    "sendACK": false,
    "controlIDRegex": "",
    "ackMsgTemplate": ""
  }
]
```

| Field | Type | Description |
|-------|------|-------------|
| `name` | String | Unique analyzer identifier. Must match `machineid` in the `machine_mapping` table. |
| `port` | int | TCP port to listen on for this analyzer's HL7 messages. |
| `ip` | String | IP address of the analyzer (informational). |
| `prefix` | String | Accession number prefix prepended to sample IDs when querying the database. |
| `outputPath` | String | Directory path where raw HL7 message files are saved as backup. |
| `sampleIDLineRegex` | String | Regex to extract the sample/accession ID from HL7 messages. Group 1 captures the ID. |
| `valueLineRegex` | String | Regex to extract test names and values from result lines. |
| `testNameGroupNum` | int | Regex group number for the test name in `valueLineRegex`. |
| `testValueGroupNum` | int | Regex group number for the test value in `valueLineRegex`. |
| `flagGroupNum` | int | Regex group number for the abnormality flag. Set to `-1` if not applicable. |
| `expectedNumOfLines` | int | If > 0, the sample is considered complete after this many lines. |
| `lastLineRegex` | String | Regex matching the last line of a sample's HL7 message block. |
| `highExp` / `lowExp` | String | Flag values indicating high/low abnormal results (e.g., "H", "L"). |
| `nonPrintableStop` | boolean | If true, a non-printable character signals end-of-sample. |
| `sendACK` | boolean | If true, sends an HL7 ACK message back to the analyzer after processing. |
| `controlIDRegex` | String | Regex to extract the message control ID (for ACK messages). |
| `ackMsgTemplate` | String | HL7 ACK message template. `#@msgControlID#@` is replaced with the actual control ID. |

### Database Configuration (`openelisDB.json`)

```json
[
  {
    "serverIP": "localhost",
    "port": 5432,
    "dbName": "clinlims",
    "userName": "clinlims",
    "password": "clinlims",
    "maxDBConnections": 10
  }
]
```

### Logging Configuration (`log4j2.xml`)

- Console output + rolling file appender
- Log files rotate at 10 MB, keeping 5 archives
- Files older than 30 days are auto-deleted
- **Note:** Update the `LOG_DIR` property to match your deployment path

---

## Database Interactions

The middleware interacts with the following OpenELIS tables:

### Tables Involved

| Table | Operation | Purpose |
|-------|-----------|---------|
| `sample` | SELECT, UPDATE | Look up pending samples by accession number; mark as processed (status_id = 2) |
| `sample_item` | SELECT (via subquery) | Link samples to their items |
| `machine_mapping` | SELECT | Map analyzer test codes to OpenELIS test IDs |
| `result` | INSERT | Store test result values |
| `result_signature` | INSERT | Auto-sign results as "Open ELIS" |
| `analysis` | UPDATE | Mark analysis as completed (status_id = 16) |
| `result_seq` | NEXTVAL | Generate result IDs |
| `result_signature_seq` | NEXTVAL | Generate result signature IDs |

### Processing Flow (per sample)

```
1. Receive HL7 message → Extract sample ID
2. SELECT from sample WHERE accession_number = prefix + sampleID
   └─ If no pending sample found → skip, log warning
3. For each test result in the message:
   a. Check if test exists in machine_mapping for this analyzer
   b. Check if the test slot is empty (no existing result)
   c. NEXTVAL('result_seq') → get new result ID
   d. INSERT INTO result (value, abnormal flag, etc.)
   e. NEXTVAL('result_signature_seq') → get signature ID
   f. INSERT INTO result_signature (auto-signed by "Open ELIS")
   g. UPDATE analysis SET status_id = 16 (completed)
4. UPDATE sample SET status_id = 2 (processed)
5. Save raw HL7 to file for audit trail
```

---

## HL7 Message Format

The middleware parses HL7-style messages (ASTM/LIS2-A2 protocol variant). Example from a Sysmex XP-300 hematology analyzer:

```
H|\^&|||XP-300^00-14^^^^B6508^AP807129||||||||E1394-97    ← Header
P|1                                                        ← Patient
O|1||^^           1984^M|^^^^WBC\^^^^RBC\...|||||||N|...   ← Order (sample ID: 1984)
R|1|^^^^WBC^1| 10.2|10*3/uL||N||||||20240221164355        ← Result (WBC = 10.2)
R|2|^^^^RBC^1| 4.73|10*6/uL||N||||||20240221164355        ← Result (RBC = 4.73)
R|3|^^^^HGB^1| 13.2|g/dL||N||||||20240221164355           ← Result (HGB = 13.2)
...
R|20|^^^^PCT^1| 0.25|%||W||||||20240221164355             ← Result (PCT = 0.25)
L|1|N                                                      ← Terminator
```

**Key segments:**
- **H** — Header: identifies the analyzer model and serial number
- **P** — Patient: patient sequence
- **O** — Order: contains the accession/sample number and requested tests
- **R** — Result: one line per test with name, value, units, and abnormality flag (N=Normal, H=High, L=Low, W=Warning)
- **L** — Terminator: signals end of message

---

## Sample End Detection

The middleware uses three strategies to detect when a complete sample message has been received (evaluated in order):

1. **Line count** — If `expectedNumOfLines > 0`, the message is complete when that many lines have been received.
2. **Last line regex** — If the current line matches `lastLineRegex` (e.g., `L|1|N`), the message is complete.
3. **Non-printable character** — If `nonPrintableStop` is true, a non-printable first character on a line signals completion.

---

## Deployment Guide

### Prerequisites

```bash
# Java 8+ runtime
sudo apt install openjdk-11-jre-headless

# Verify
java -version
```

### 1. Create Directory Structure

```bash
mkdir -p /opt/hl7-middleware
mkdir -p /var/log/hl7/{SYSMIX,logs}   # one subdir per analyzer + logs dir
```

### 2. Place Files

```
/opt/hl7-middleware/
├── captureHl7Messages.jar    # the fat JAR (built from IntelliJ artifact)
├── openelisDB.json           # DB connection config
└── analyzers.json            # analyzer definitions
```

The JAR is built as an IntelliJ IDEA artifact with all dependencies bundled:
```bash
# Output: out/artifacts/captureHl7Messages_jar/captureHl7Messages.jar
```

### 3. Configure Database Connection (`openelisDB.json`)

Point this to the OpenELIS PostgreSQL instance. If Bahmni runs in Docker containers, use the container's IP or Docker network alias:

```json
[
  {
    "serverIP": "172.18.0.2",
    "port": 5432,
    "dbName": "clinlims",
    "userName": "clinlims",
    "password": "<actual_password>",
    "maxDBConnections": 10
  }
]
```

> **Docker note:** The middleware either needs to run on the host network or inside the Docker network to reach the PostgreSQL container. Use `docker inspect <container>` to find the container IP, or add the middleware to the same Docker network.

### 4. Configure Analyzer(s) (`analyzers.json`)

Each analyzer in the JSON array gets its own listening thread. Example for a Sysmex XP-300:

```json
[
  {
    "name": "XP-300",
    "port": 9100,
    "ip": "0.0.0.0",
    "prefix": "300",
    "outputPath": "/var/log/hl7/SYSMIX/",
    "sampleIDLineRegex": "O\\|\\d+\\|\\|\\^\\^\\s+(\\d+)",
    "valueLineRegex": "R\\|\\d+\\|\\^\\^\\^\\^(\\w[\\w-]*)\\^\\d+\\|\\s*([\\d.]+)\\|[^|]*\\|\\|([NHLW])",
    "testNameGroupNum": 1,
    "testValueGroupNum": 2,
    "flagGroupNum": 3,
    "expectedNumOfLines": 0,
    "lastLineRegex": "L\\|1\\|N",
    "highExp": "H",
    "lowExp": "L",
    "nonPrintableStop": false,
    "sendACK": false,
    "controlIDRegex": "",
    "ackMsgTemplate": ""
  }
]
```

> **⚠️ Note:** The exact mechanism for loading the analyzer config file path in `Main.java` may be hardcoded or passed as a command-line argument. Check the JAR or source before deploying.

### 5. Populate `machine_mapping` Table

The `machine_mapping` table is a custom table in the OpenELIS database that maps analyzer test codes to OpenELIS test IDs. One row per test per analyzer:

```sql
-- Connect to the clinlims database
INSERT INTO machine_mapping (machineid, machinetestcode, listestcode)
VALUES 
  ('XP-300', 'WBC', '<openelis_test_id>'),
  ('XP-300', 'RBC', '<openelis_test_id>'),
  ('XP-300', 'HGB', '<openelis_test_id>'),
  ('XP-300', 'HCT', '<openelis_test_id>'),
  ('XP-300', 'MCV', '<openelis_test_id>'),
  ('XP-300', 'MCH', '<openelis_test_id>'),
  ('XP-300', 'MCHC', '<openelis_test_id>'),
  ('XP-300', 'PLT', '<openelis_test_id>'),
  ('XP-300', 'LYM%', '<openelis_test_id>'),
  ('XP-300', 'MXD%', '<openelis_test_id>'),
  ('XP-300', 'NEUT%', '<openelis_test_id>'),
  ('XP-300', 'LYM#', '<openelis_test_id>'),
  ('XP-300', 'MXD#', '<openelis_test_id>'),
  ('XP-300', 'NEUT#', '<openelis_test_id>'),
  ('XP-300', 'RDW-SD', '<openelis_test_id>'),
  ('XP-300', 'RDW-CV', '<openelis_test_id>'),
  ('XP-300', 'PDW', '<openelis_test_id>'),
  ('XP-300', 'MPV', '<openelis_test_id>'),
  ('XP-300', 'P-LCR', '<openelis_test_id>'),
  ('XP-300', 'PCT', '<openelis_test_id>');
```

The `machineid` must match the `name` field in the analyzer JSON config. The `listestcode` values must match the `test_id` in the OpenELIS `test` table.

### 6. Network & Firewall

The lab analyzer must be able to reach the middleware server on the configured TCP port:

```bash
# Open the port (if using ufw)
sudo ufw allow 9100/tcp

# On the analyzer hardware, configure the LIS host IP and port
# to point to the middleware server's IP address
```

### 7. Fix the Log Path

The bundled `log4j2.xml` references a macOS development path. Override via JVM system property:

```bash
java -DLOG_DIR=/var/log/hl7/logs -jar captureHl7Messages.jar
```

### 8. Run as a Systemd Service

```ini
# /etc/systemd/system/hl7-middleware.service
[Unit]
Description=HL7 Lab Middleware
After=network.target docker.service

[Service]
Type=simple
WorkingDirectory=/opt/hl7-middleware
ExecStart=/usr/bin/java -DLOG_DIR=/var/log/hl7/logs -jar captureHl7Messages.jar
Restart=always
RestartSec=10
User=bahmni

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable hl7-middleware
sudo systemctl start hl7-middleware

# Check status
sudo systemctl status hl7-middleware
journalctl -u hl7-middleware -f
```

### Shutdown

The middleware registers a JVM shutdown hook that cleanly closes the database connection pool. `systemctl stop` triggers a graceful shutdown.

---

## Adding a New Analyzer

1. **Capture a sample HL7 message** from the analyzer using a TCP listener:
   ```bash
   nc -l -p 9100 > sample_message.txt
   ```

2. **Write regex patterns** for:
   - Sample ID extraction (`sampleIDLineRegex`)
   - Test result extraction (`valueLineRegex`)
   - End-of-message detection (`lastLineRegex`)

3. **Create `machine_mapping` entries** in the OpenELIS database:
   ```sql
   INSERT INTO machine_mapping (machineid, machinetestcode, listestcode)
   VALUES ('NEW_ANALYZER', 'WBC', '<openelis_test_id>');
   -- repeat for each test
   ```

4. **Add the analyzer** to the JSON configuration array with a unique port.

5. **Create the output directory** for HL7 backup files.

6. **Open the firewall port** for the new analyzer.

7. **Restart the middleware:**
   ```bash
   sudo systemctl restart hl7-middleware
   ```

---

## Known Limitations

- **Single-threaded per analyzer** — Each analyzer gets one thread. If the analyzer sends a new sample before the previous one is fully processed, messages could interleave.
- **No TLS** — TCP connections are unencrypted. Suitable for isolated hospital LANs only.
- **SQL injection risk** — Queries use `String.format()` instead of parameterized queries. The accession numbers and test codes come from the analyzer hardware, but this should be hardened.
- **No reconnection logic** — If the database connection fails mid-processing, the current sample's results may be partially written.
- **No transaction wrapping** — Result inserts, signature creation, and analysis updates are not wrapped in a single transaction. A failure mid-way could leave partial results.
- **Analyzer config loading** — The exact file path for analyzer JSON may be hardcoded in `Main.java`. Verify before deploying.
