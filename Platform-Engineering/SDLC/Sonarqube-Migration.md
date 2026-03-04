---
title: SonarQube Migration (EC2 -> EKS + Self-hosted DB -> AWS RDS Postgres)
owner: Platform Engineering
last_updated: 2026-03-04
---

# SonarQube step-by-step migration

This document describes a practical, command-driven migration of **SonarQube**:

1. **Compute migration:** from a SonarQube deployment on **EC2** to **AWS EKS**.
2. **Database migration:** from a **self-hosted SQL database** to **PostgreSQL on AWS RDS**.
3. **Schema migration:** highlights schema validation/migration using the **SonarDB copy tool** approach.
4. **Commands:** includes step-by-step commands and verification steps.

> Assumptions
>
> - You already have access to AWS, EKS, and the existing SonarQube EC2 instance.
> - You can reach the **source DB** from your machine (or from a bastion) and you can reach the **RDS endpoint**.
> - You will use **Helm** to install SonarQube on EKS.
> - Replace placeholders like `<...>` with your real values.

## Checklist (requirements coverage)

- [x] Migration steps for **EC2 > EKS**
- [x] Migration steps for **EC2 -> EKS**
- [x] Migration steps for **self-hosted SQL -> AWS RDS Postgres**
- [x] Highlight **schema migration** using a **SonarDB copy tool** workflow
- [x] Provide **step-by-step commands**

## Pre-migration: collect current state (source)

### 1) Record versions and configuration

On the EC2 host (or wherever SonarQube runs), capture:

- SonarQube version
- Current DB engine/version
- Current `sonar.properties`
- Installed plugins list

Example commands:

```bash
# On the SonarQube EC2 instance
sudo -i

sonar_version_file=/opt/sonarqube/lib/sonar-application-*.jar
ls -la $sonar_version_file

# Typical locations (adapt to your install)
cat /opt/sonarqube/conf/sonar.properties | sed -n '1,200p'

# Plugins
ls -1 /opt/sonarqube/extensions/plugins
```

### 2) Confirm the source database connection details

Identify the DB connection details currently used:

- `sonar.jdbc.url`
- `sonar.jdbc.username`

If credentials are stored in a secret store, ensure you can retrieve them for migration.

## Target architecture overview

### Target components

- **EKS namespace**: `sonarqube` (recommended)
- **SonarQube** running via Helm chart
- **AWS RDS for PostgreSQL** for `sonarqube` database
- **Kubernetes secrets** for DB credentials
- **Persistent storage** for SonarQube (only for ephemeral needs; DB is the source of truth)

### Compatibility notes

- SonarQube supports PostgreSQL. Ensure your target SonarQube version supports the Postgres version you plan on RDS.
- If migrating from a *non-Postgres* engine, prefer using an official/compatible migration approach (dump/transform) and validate schema/data carefully.

## Phase A: Provision AWS RDS Postgres (target DB)

You can create RDS using Console, Terraform, or AWS CLI. This section focuses on the *must-have* configuration.

### 1) Create the database and user

Recommended:

- DB name: `sonarqube`
- User: `sonar`
- Storage: sized to your current DB + growth
- Backups enabled
- Security group allows inbound **5432** from:
	- EKS node security group **or** EKS security group (depending on your setup)
	- admin/bastion IP for maintenance

### 2) Create database objects (minimal)

Connect to RDS (from a bastion/your machine) and create the DB/user if not already.

```bash
# Example: connect with psql (on a machine that can reach RDS)
export PGHOST="<rds-endpoint>"
export PGPORT="5432"
export PGUSER="<master-user>"
export PGPASSWORD="<master-password>"

psql -d postgres
```

Inside `psql`:

```sql
-- Create DB and user for SonarQube
CREATE DATABASE sonarqube;

CREATE USER sonar WITH ENCRYPTED PASSWORD '<sonar-password>';

GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonar;

-- Recommended for SonarQube: allow required extensions when needed
-- (SonarQube generally manages its own schema; keep extensions minimal.)
```

> Note: Some orgs also set parameter group values related to connection limits, work_mem, or max_connections based on load. Keep defaults unless you have performance data.

## Phase B: Export / copy data from source DB and migrate schema

There are two common patterns:

1. **Same engine migration** (Postgres > Postgres): use `pg_dump`/`pg_restore` or logical replication.
2. **Cross-engine migration** (e.g., MySQL/SQL Server > Postgres): use an ETL/conversion tool and validate schema carefully.

This doc highlights a **SonarDB copy tool workflow** to emphasize **schema migration/validation**, plus notes for native Postgres dumps.

### Option 1 (recommended when source is Postgres): pg_dump/pg_restore

```bash
# Source side
export PGHOST="<source-db-host>"
export PGPORT="5432"
export PGUSER="<source-db-user>"
export PGPASSWORD="<source-db-password>"

pg_dump --format=custom --no-owner --no-privileges --jobs=4 --file sonarqube.dump sonarqube

# Target side
export PGHOST="<rds-endpoint>"
export PGPORT="5432"
export PGUSER="<rds-master-or-restore-user>"
export PGPASSWORD="<rds-password>"

# Clean restore (optional) - only if DB is empty OR you accept destructive reset
psql -d postgres -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='sonarqube' AND pid <> pg_backend_pid();"
psql -d postgres -c "DROP DATABASE IF EXISTS sonarqube;"
psql -d postgres -c "CREATE DATABASE sonarqube;"

pg_restore --no-owner --no-privileges --jobs=4 --dbname sonarqube sonarqube.dump
```

### Option 2 (schema migration highlight): SonarDB copy tool workflow

Use this approach when you need **explicit schema mapping/verification** during a cross-engine migration.

#### 1) Extract schema and data from source

What you do here depends on your source DB engine.

- If source is MySQL: export schema + data (or use a migration tool that generates Postgres DDL).
- If source is SQL Server: export via BCP/SSIS or a migration tool.

#### 2) Use a copy/migration tool to apply schema and load data into Postgres

**SonarDB copy tool**: the goal is to:

1. Create/transform schema in Postgres to match what SonarQube expects.
2. Copy data into Postgres.
3. Validate row counts and critical tables.

> Repo note
>
> This workspace contains `data/copy.yml` / `data/copy.yaml`, but those are Ansible playbooks for copying kube/vault-related files.
> If your SonarDB copy tool is stored elsewhere (scripts, container image, internal repo), update this doc with the correct path.

Below is a **generic, repeatable** pattern that works well in practice.

##### A) Prepare connection strings

```bash
# Source (self-hosted)
export SRC_DB_URL="<src-connection-string>"
export SRC_DB_USER="<src-user>"
export SRC_DB_PASSWORD="<src-password>"

# Target (AWS RDS Postgres)
export DST_PG_HOST="<rds-endpoint>"
export DST_PG_PORT="5432"
export DST_PG_DB="sonarqube"
export DST_PG_USER="sonar"
export DST_PG_PASSWORD="<sonar-password>"
```

##### B) Run schema migration (generate + apply)

If your SonarDB copy tool supports **schema-only** first, do it:

```bash
# Example pattern (replace with your actual tool)
# 1) generate Postgres schema
sonardb-copy \
	--source-url "$SRC_DB_URL" \
	--source-user "$SRC_DB_USER" \
	--source-password "$SRC_DB_PASSWORD" \
	--target-pg-host "$DST_PG_HOST" \
	--target-pg-port "$DST_PG_PORT" \
	--target-pg-db "$DST_PG_DB" \
	--target-pg-user "$DST_PG_USER" \
	--target-pg-password "$DST_PG_PASSWORD" \
	--mode schema \
	--output schema.sql

# 2) apply schema.sql to RDS
export PGHOST="$DST_PG_HOST"
export PGPORT="$DST_PG_PORT"
export PGDATABASE="$DST_PG_DB"
export PGUSER="$DST_PG_USER"
export PGPASSWORD="$DST_PG_PASSWORD"

psql -v ON_ERROR_STOP=1 -f schema.sql
```

##### C) Copy data

```bash
sonardb-copy \
	--source-url "$SRC_DB_URL" \
	--source-user "$SRC_DB_USER" \
	--source-password "$SRC_DB_PASSWORD" \
	--target-pg-host "$DST_PG_HOST" \
	--target-pg-port "$DST_PG_PORT" \
	--target-pg-db "$DST_PG_DB" \
	--target-pg-user "$DST_PG_USER" \
	--target-pg-password "$DST_PG_PASSWORD" \
	--mode data
```

##### D) Validate schema + data

At minimum, validate:

- tables exist
- row counts for large tables are within expected ranges
- indexes exist

```bash
export PGHOST="$DST_PG_HOST"
export PGPORT="$DST_PG_PORT"
export PGDATABASE="$DST_PG_DB"
export PGUSER="$DST_PG_USER"
export PGPASSWORD="$DST_PG_PASSWORD"

psql -c "\dt" 

# Example validation queries (adapt table names to your version)
psql -c "SELECT COUNT(*) AS projects FROM projects;" || true
psql -c "SELECT COUNT(*) AS issues FROM issues;" || true
psql -c "SELECT COUNT(*) AS users FROM users;" || true
```

> Table names vary by SonarQube version; if these fail, list tables first and validate key ones you find (e.g., `projects`, `issues`, `rules`, `snapshots`, `ce_activity`, `users`).

## Phase C: Prepare EKS (namespace, secrets, storage)

### 1) Configure kubectl and verify access

```bash
aws eks update-kubeconfig --region <region> --name <cluster-name>
kubectl get nodes
```

### 2) Create namespace

```bash
kubectl create namespace sonarqube
```

### 3) Create Kubernetes secret for DB credentials

```bash
kubectl -n sonarqube create secret generic sonarqube-db \
	--from-literal=jdbcUsername="sonar" \
	--from-literal=jdbcPassword="<sonar-password>"
```

## Phase D: Deploy SonarQube on EKS (Helm)

### 1) Add SonarSource Helm repo

```bash
helm repo add sonarqube https://SonarSource.github.io/helm-chart-sonarqube
helm repo update
```

### 2) Create a values file for RDS Postgres

Create `sonarqube-values.yaml` (example):

```yaml
sonarqube:
	jdbcOverwrite:
		enable: true
		jdbcUrl: "jdbc:postgresql://<rds-endpoint>:5432/sonarqube"
		jdbcUsername: "${jdbcUsername}"
		jdbcPassword: "${jdbcPassword}"

	# If you want to reference the secret keys directly (chart-specific), you may use
	# extraEnv or secret-based configuration depending on the chart version.

postgresql:
	enabled: false
```

> Note: Helm chart configuration keys differ by chart version. Validate with:
> `helm show values sonarqube/sonarqube | less`

### 3) Install/upgrade SonarQube

```bash
helm upgrade --install sonarqube sonarqube/sonarqube \
	--namespace sonarqube \
	-f sonarqube-values.yaml

kubectl -n sonarqube get pods
kubectl -n sonarqube logs deploy/sonarqube -f
```

### 4) Expose SonarQube (Ingress/ALB)

Depending on your org standard, expose via:

- `Service` type `LoadBalancer` (quick, less control)
- Ingress controller (ALB/Nginx) (recommended)

Example Ingress skeleton:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: sonarqube
	namespace: sonarqube
spec:
	rules:
		- host: sonarqube.<your-domain>
			http:
				paths:
					- path: /
						pathType: Prefix
						backend:
							service:
								name: sonarqube
								port:
									number: 9000
```

## Phase E: Cutover plan (minimize downtime)

### 1) Freeze changes on source

To reduce drift:

- stop background jobs and analysis submission (pause CI)
- optionally stop SonarQube service on EC2 during final sync

```bash
# On EC2 (example service name; adjust)
sudo systemctl stop sonarqube
sudo systemctl status sonarqube
```

### 2) Run final DB sync / final copy

Re-run the final migration step (dump+restore or SonarDB copy) to capture the final state.

### 3) Start SonarQube on EKS

Once the target DB is final and reachable, start/rollout SonarQube pods:

```bash
kubectl -n sonarqube rollout status deploy/sonarqube
kubectl -n sonarqube get pods -owide
```

### 4) Validate application

Validate:

- UI loads
- background tasks running
- projects visible
- new analysis works

```bash
kubectl -n sonarqube logs deploy/sonarqube --tail=200
```

If SonarQube performs internal DB upgrade on startup, let it complete and monitor logs carefully.

## Rollback strategy

If cutover fails:

1. Keep EC2 instance intact until fully validated.
2. Restore the old DB endpoint configuration (DNS or config) if needed.
3. Re-enable pipelines against EC2 SonarQube.

Practical rollback steps:

- Point DNS back to old SonarQube endpoint
- Start SonarQube on EC2: `sudo systemctl start sonarqube`
- Scale down EKS deployment: `kubectl -n sonarqube scale deploy/sonarqube --replicas=0`

## Post-migration hardening

- Enable automated RDS backups and test restore.
- Ensure RDS parameter group aligns with expected load.
- Set up monitoring (CloudWatch metrics, logs, alarms).
- Rotate DB credentials and store them in Secrets Manager / External Secrets.
- Ensure EKS pods have resource requests/limits tuned.

## Appendix: Useful troubleshooting

### DB connectivity from EKS

Typical causes:

- security group rules missing
- wrong subnet route/NACL
- RDS not in correct VPC

Quick check: run a temporary pod and try to connect to Postgres.

```bash
kubectl -n sonarqube run psql-client --rm -it --restart=Never \
	--image=postgres:16 \
	--env="PGPASSWORD=<sonar-password>" \
	-- psql -h <rds-endpoint> -U sonar -d sonarqube -c "SELECT 1;"
```

