# Automated-SDLC-Compliance-Engine

A policy-driven compliance engine that scans source code for SDLC violations using regex, logs results to a PostgreSQL audit trail, and outputs structured JSON reports.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Engine | Java 17+ |
| Database | PostgreSQL |
| Testing | JUnit 5 |
| Data Format | JSON |

---

## Project Structure

```
automated-sdlc-compliance-engine/
├── src/
│   ├── main/java/com/yourorg/asce/
│   │   ├── Main.java
│   │   ├── engine/
│   │   │   ├── ScanEngine.java          # Core regex scanning
│   │   │   ├── PolicyLoader.java        # Loads JSON policies
│   │   │   └── FileScanner.java         # File traversal
│   │   ├── model/
│   │   │   ├── CompliancePolicy.java
│   │   │   ├── ScanResult.java
│   │   │   └── Violation.java
│   │   ├── db/
│   │   │   ├── DatabaseConnection.java  # HikariCP pool
│   │   │   ├── ViolationRepository.java
│   │   │   └── AuditLogRepository.java
│   │   └── report/
│   │       └── ReportGenerator.java     # JSON output
│   └── test/java/com/yourorg/asce/
│       ├── engine/ScanEngineTest.java
│       ├── db/ViolationRepositoryTest.java
│       └── report/ReportGeneratorTest.java
├── policies/                            # JSON policy files
├── reports/                             # Generated reports (gitignored)
├── sql/schema.sql
└── pom.xml
```

---

## Prerequisites

- Java 17+
- Maven 3.8+
- PostgreSQL 14+

---

## Setup

```bash
git clone https://github.com/your-org/automated-sdlc-compliance-engine.git
cd automated-sdlc-compliance-engine
cp .env.example .env        # Add DB credentials
mvn clean install
```

**`.env`**
```env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=asce_db
DB_USER=asce_user
DB_PASSWORD=yourpassword
POLICY_DIR=./policies
REPORT_OUTPUT_DIR=./reports
```

---

## Database Setup

```sql
CREATE DATABASE asce_db;
CREATE USER asce_user WITH ENCRYPTED PASSWORD 'yourpassword';
GRANT ALL PRIVILEGES ON DATABASE asce_db TO asce_user;
```

```bash
psql -U asce_user -d asce_db -f sql/schema.sql
```

**`sql/schema.sql`**
```sql
CREATE TABLE scan_audit_log (
    id              BIGSERIAL PRIMARY KEY,
    scan_id         UUID NOT NULL DEFAULT gen_random_uuid(),
    triggered_at    TIMESTAMP NOT NULL DEFAULT NOW(),
    target_path     TEXT NOT NULL,
    status          VARCHAR(20) NOT NULL, -- PASSED | FAILED | ERROR
    total_files     INT NOT NULL DEFAULT 0,
    violation_count INT NOT NULL DEFAULT 0
);

CREATE TABLE violations (
    id              BIGSERIAL PRIMARY KEY,
    scan_id         UUID NOT NULL REFERENCES scan_audit_log(scan_id),
    file_path       TEXT NOT NULL,
    line_number     INT,
    policy_id       VARCHAR(100) NOT NULL,
    severity        VARCHAR(20) NOT NULL, -- CRITICAL | HIGH | MEDIUM | LOW
    matched_pattern TEXT NOT NULL,
    matched_value   TEXT,
    detected_at     TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Immutable audit trail — no updates or deletes allowed
CREATE RULE no_update_audit      AS ON UPDATE TO scan_audit_log DO INSTEAD NOTHING;
CREATE RULE no_delete_audit      AS ON DELETE TO scan_audit_log DO INSTEAD NOTHING;
CREATE RULE no_update_violations AS ON UPDATE TO violations     DO INSTEAD NOTHING;
CREATE RULE no_delete_violations AS ON DELETE TO violations     DO INSTEAD NOTHING;
```

---

## Key Dependencies (`pom.xml`)

```xml
<dependencies>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.16.1</version>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <version>42.7.1</version>
    </dependency>
    <dependency>
        <groupId>com.zaxxer</groupId>
        <artifactId>HikariCP</artifactId>
        <version>5.1.0</version>
    </dependency>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.1</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <version>5.8.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## Policies

Policies are JSON files in `policies/` loaded at runtime. Each defines regex patterns to match against source files.

```json
{
  "id": "SEC-001",
  "name": "Hardcoded Secrets Detection",
  "severity": "CRITICAL",
  "blockOnViolation": true,
  "fileTypes": [".java", ".properties", ".yml"],
  "patterns": [
    "(?i)(password|passwd)\\s*=\\s*['\"][^'\"]{4,}['\"]",
    "(?i)(api[_-]?key)\\s*=\\s*['\"][^'\"]{8,}['\"]",
    "AKIA[0-9A-Z]{16}"
  ]
}
```

---

## Running

```bash
# Via Maven
mvn exec:java -Dexec.mainClass="com.yourorg.asce.Main" \
  -Dexec.args="--target ./src --policies ./policies"

# Via JAR
mvn package
java -jar target/asce-1.0.0.jar --target ./src --policies ./policies
```

| Argument | Description | Default |
|---|---|---|
| `--target` | Directory to scan | `./src` |
| `--policies` | Policy JSON directory | `./policies` |
| `--output` | Report output directory | `./reports` |
| `--fail-on` | Minimum severity for exit code 1 | `CRITICAL` |
| `--dry-run` | Skip DB writes | `false` |

**Exit codes:** `0` = passed · `1` = violations found · `2` = engine error

---

## JSON Report Output

Reports are written to `reports/scan-{uuid}-{timestamp}.json`.

```json
{
  "scanId": "f3a1c920-8b2d-4e77-a3f1-0012ab45cd67",
  "triggeredAt": "2025-03-25T14:32:00Z",
  "status": "FAILED",
  "summary": {
    "totalFilesScanned": 84,
    "totalViolations": 2,
    "critical": 1,
    "medium": 1,
    "low": 0
  },
  "violations": [
    {
      "policyId": "SEC-001",
      "severity": "CRITICAL",
      "filePath": "src/main/java/com/example/Config.java",
      "lineNumber": 42,
      "matchedValue": "password=\"supersecret123\"",
      "detectedAt": "2025-03-25T14:32:01Z"
    }
  ]
}
```

---

## Testing

```bash
mvn test                          # Run all tests
mvn test -Dtest=ScanEngineTest    # Run a specific class
```

Tests cover: regex match accuracy, false-positive prevention, DB insert/immutability, and JSON report correctness.

---

## Roadmap

- [ ] Maven plugin for pipeline integration
- [ ] Incremental scanning (changed files only)
- [ ] SARIF output format support
- [ ] Auto-remediation for low-severity violations
- [ ] Web dashboard for scan history

---

