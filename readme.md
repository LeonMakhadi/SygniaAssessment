# Task 1 — Local MySQL Setup

Local MySQL environment for **Task 1**, containing two tables used for
reconciliation/testing:

- `source_calculations` — expected amounts per client (source of truth)
- `distribution_output` — actual amounts distributed per client

The dataset includes intentional edge cases (duplicates, missing rows, extra
rows, rounding differences, negative amounts) so reconciliation logic can be
exercised against real-world data quality issues.

---

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed and running
- A database client — [DBeaver](https://dbeaver.io/) recommended, or `mysql` CLI

---

## 1. Start MySQL in Docker

```bash
docker run --name little-mysql \
  -e MYSQL_ROOT_PASSWORD=your_password \
  -p 3306:3306 \
  -d mysql:latest
```

Verify the container is running:

```bash
docker ps
```

> **Note:** The container does not create an application database for you.
> You'll create it in step 3.

To stop / restart later:

```bash
docker stop little-mysql
docker start little-mysql
```

---

## 2. Connect with DBeaver

Create a new **MySQL** connection using:

| Setting  | Value                          |
| -------- | ------------------------------ |
| Host     | `localhost`                    |
| Port     | `3306`                         |
| Username | `root`                         |
| Password | the password used in step 1    |

Click **Test Connection**, then **Finish**.

---

## 3. Create the database and tables

```sql
CREATE DATABASE task1;
USE task1;
```

### `source_calculations`

```sql
CREATE TABLE source_calculations (
    client_id           VARCHAR(10) PRIMARY KEY,
    client_name         VARCHAR(100),
    source_amount       DECIMAL(12,2),
    fee_percentage      DECIMAL(5,2),
    expected_net_amount DECIMAL(12,2)
);
```

Seed data:

```sql
INSERT INTO source_calculations
(client_id, client_name, source_amount, fee_percentage, expected_net_amount)
VALUES
('C001', 'Client A', 10000.00, 2.00,  9800.00),
('C002', 'Client B',  5250.75, 2.00,  5145.74),
('C003', 'Client C',   875.00, 2.00,   857.50),
('C004', 'Client D', 12340.10, 2.00, 12093.30),
('C005', 'Client E',  3000.00, 2.00,  2940.00),
('C006', 'Client F',   999.99, 2.00,   979.99),
('C007', 'Client G', 45000.00, 2.00, 44100.00),
('C008', 'Client H',   150.00, 2.00,   147.00),
('C009', 'Client I',  2500.50, 2.00,  2450.49),
('C010', 'Client J',  7777.77, 2.00,  7622.21),
('C011', 'Client K',  -200.00, 2.00,  -196.00),
('C012', 'Client L',  6000.00, 2.00,  5880.00);
```

### `distribution_output`

```sql
CREATE TABLE distribution_output (
    client_id          VARCHAR(10),
    client_name        VARCHAR(100),
    distributed_amount DECIMAL(12,2),
    status             VARCHAR(20)
);
```

Seed data:

```sql
INSERT INTO distribution_output
(client_id, client_name, distributed_amount, status)
VALUES
('C001', 'Client A',  9800.00, 'COMPLETED'),
('C002', 'Client B',  5145.73, 'COMPLETED'),
('C003', 'Client C',   857.50, 'COMPLETED'),
('C004', 'Client D', 12093.30, 'COMPLETED'),
('C006', 'Client F',   979.99, 'COMPLETED'),
('C007', 'Client G', 44100.00, 'COMPLETED'),
('C007', 'Client G', 44100.00, 'COMPLETED'),
('C008', 'Client H',   147.00, 'COMPLETED'),
('C009', 'Client I',  2450.49, 'COMPLETED'),
('C010', 'Client J',  7622.21, 'COMPLETED'),
('C011', 'Client K',     0.00, 'COMPLETED'),
('C012', 'Client L',  5880.00, 'COMPLETED'),
('C013', 'Client M',   320.00, 'COMPLETED');
```

---

## 4. Verify the setup

```sql
SELECT COUNT(*) FROM source_calculations;    -- expect 12
SELECT COUNT(*) FROM distribution_output;    -- expect 13
```

Quick side-by-side comparison:

```sql
SELECT
    s.client_id,
    s.client_name,
    s.expected_net_amount,
    d.distributed_amount,
    d.distributed_amount - s.expected_net_amount AS delta,
    d.status
FROM source_calculations s
LEFT JOIN distribution_output d ON d.client_id = s.client_id
ORDER BY s.client_id;
```

---

## 
---

## Troubleshooting

| Problem | Fix |
| ------- | --- |
| `Port 3306 already in use` | Stop the local MySQL service, or map a different host port (e.g. `-p 3307:3306`) |
| `Access denied for user 'root'` | Re-check `MYSQL_ROOT_PASSWORD` from step 1 |
| `Unknown database 'task1'` | Run `CREATE DATABASE task1;` first |
| Container name conflict | `docker rm -f little-mysql` and re-run the `docker run` command |
| Need to reset everything | `docker rm -f little-mysql`, then repeat from step 1 |

---

## Reset / teardown

```bash
docker stop little-mysql
docker rm little-mysql
```

Re-run the `docker run` command in step 1 to start fresh.
