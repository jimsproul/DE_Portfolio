For your Hyper-V lab, I recommend installing **dbt Core and Dagster together in one Python 3.13 project environment on Ubuntu**. That lets Dagster invoke dbt directly through `dagster-dbt`, while both connect to the PostgreSQL container running under Docker Desktop on the Windows 11 host.

One current compatibility point matters: Dagster 1.13 supports Python 3.10–3.14, but the current `dagster-dbt` 0.29.22 package requires **Python >=3.10 and <3.14**. Therefore, even if Ubuntu 26.04 provides Python 3.14 as its system Python, use **Python 3.13** for this project. ([PyPI][1])

Your target architecture will be:

```text
                         WINDOWS 11 PRO
                               │
                ┌──────────────┴──────────────┐
                │                             │
            Hyper-V                      Docker Desktop
                │                             │
        Ubuntu 26.04 VM                       │
        192.168.250.10                        │
                │                             │
                │                       PostgreSQL
                │                         Container
                │                             │
                │                       192.168.250.1
                │                           :5432
                │                             │
                ├──── dbt ────────────────────┤
                │                             │
                └──── Dagster ────────────────┘
                         │
                         │ invokes
                         ▼
                        dbt
```

## 1. Verify the PostgreSQL container first

Before installing dbt or Dagster, make sure Ubuntu can still reach the PostgreSQL container you configured earlier.

Your connection is:

```text
Windows host:   192.168.250.1
PostgreSQL:     5432
Database:       datalab
User:           labuser
```

From Ubuntu:

```bash
ip -br addr
```

You should have your DevLab interface:

```text
192.168.250.10/24
```

Test Windows:

```bash
ping -c 4 192.168.250.1
```

Then test PostgreSQL:

```bash
nc -vz 192.168.250.1 5432
```

You want:

```text
Connection to 192.168.250.1 5432 port [tcp/postgresql] succeeded!
```

Install the PostgreSQL client if necessary:

```bash
sudo apt update
sudo apt install -y postgresql-client
```

Test an actual database login:

```bash
psql \
    -h 192.168.250.1 \
    -p 5432 \
    -U labuser \
    -d datalab
```

Run:

```sql
SELECT
    current_database(),
    current_user,
    version();
```

Then:

```text
\q
```

Do not proceed until this works. If `psql` cannot connect, dbt and Dagster won't be able to either.

---

# 2. Install Ubuntu development prerequisites

Update Ubuntu:

```bash
sudo apt update
sudo apt full-upgrade -y
```

Install basic development packages:

```bash
sudo apt install -y \
    build-essential \
    git \
    curl \
    wget \
    unzip \
    jq \
    tree \
    netcat-openbsd \
    postgresql-client \
    libpq-dev \
    pkg-config
```

Verify:

```bash
git --version
psql --version
curl --version
```

---

# 3. Use Python 3.13 rather than the Ubuntu system Python

I recommend using **uv** to manage Python for this lab rather than altering Ubuntu's system Python.

This is particularly useful here because the current `dagster-dbt` release requires Python below 3.14. ([PyPI][2])

Install `uv`:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Reload the shell:

```bash
source ~/.bashrc
```

Verify:

```bash
uv --version
```

Install Python 3.13:

```bash
uv python install 3.13
```

Verify available Python versions:

```bash
uv python list
```

You should see a Python 3.13 installation.

---

# 4. Create the data-engineering project

I recommend:

```text
~/data-platform-lab/
```

Create it:

```bash
mkdir -p ~/data-platform-lab
cd ~/data-platform-lab
```

We'll eventually have:

```text
data-platform-lab/
│
├── .venv/
│
├── dbt/
│   └── datalab/
│       ├── dbt_project.yml
│       ├── models/
│       ├── seeds/
│       ├── snapshots/
│       └── tests/
│
├── dagster/
│   └── datalab_orchestrator/
│
├── .env
├── .gitignore
├── pyproject.toml
└── README.md
```

---

# 5. Create the Python 3.13 environment

From:

```bash
cd ~/data-platform-lab
```

create:

```bash
uv venv --python 3.13
```

Activate it:

```bash
source .venv/bin/activate
```

Check:

```bash
python --version
```

You want:

```text
Python 3.13.x
```

You can also check:

```bash
which python
```

It should point to something like:

```text
/home/<user>/data-platform-lab/.venv/bin/python
```

---

# 6. Install dbt PostgreSQL adapter

For PostgreSQL, install:

```bash
uv pip install dbt-postgres
```

You do not need to separately install `dbt-core`; `dbt-postgres` will bring in a compatible Core version.

The current PyPI `dbt-postgres` release is 1.11.0 and supports Python 3.10 through 3.14. It installs `psycopg2-binary` by default, which dbt describes as appropriate for development and testing. ([PyPI][3])

Verify:

```bash
dbt --version
```

You should see something resembling:

```text
Core:
  installed: ...

Plugins:
  - postgres: ...
```

---

# 7. Install Dagster

Install:

```bash
uv pip install \
    dagster \
    dagster-webserver \
    dagster-dbt \
    psycopg[binary] \
    sqlalchemy \
    python-dotenv
```

The current Dagster release is 1.13.22, while the matching integration line is `dagster-dbt` 0.29.22. ([PyPI][1])

Verify:

```bash
dagster --version
```

Also:

```bash
python -c "import dagster; print(dagster.__version__)"
```

Check dbt integration:

```bash
python -c "import dagster_dbt; print('dagster-dbt OK')"
```

---

# 8. Save the environment definition

You should record your Python dependencies.

Create:

```bash
uv pip freeze > requirements.txt
```

Later you can reproduce it using:

```bash
uv pip install -r requirements.txt
```

A more modern alternative is to build a proper `pyproject.toml`, but freezing the environment is useful while you are learning the stack.

---

# 9. Create a secure PostgreSQL environment file

Don't embed the PostgreSQL password in Python or dbt SQL.

Create:

```bash
nano ~/data-platform-lab/.env
```

Add:

```bash
POSTGRES_HOST=192.168.250.1
POSTGRES_PORT=5432
POSTGRES_DB=datalab
POSTGRES_USER=labuser
POSTGRES_PASSWORD=labpassword
```

Use your actual password if different.

Protect it:

```bash
chmod 600 .env
```

Add it to Git's ignore file:

```bash
cat > .gitignore <<'EOF'
.venv/
.env
__pycache__/
*.pyc
target/
logs/
dbt_packages/
.DS_Store
EOF
```

This is important:

```text
.env
```

must **never** be committed to GitHub.

---

# 10. Load the environment variables

For the current shell:

```bash
set -a
source .env
set +a
```

Test:

```bash
echo "$POSTGRES_HOST"
```

Expected:

```text
192.168.250.1
```

Do **not** run:

```bash
echo "$POSTGRES_PASSWORD"
```

in normal troubleshooting logs.

Test the connection using environment variables:

```bash
PGPASSWORD="$POSTGRES_PASSWORD" \
psql \
    -h "$POSTGRES_HOST" \
    -p "$POSTGRES_PORT" \
    -U "$POSTGRES_USER" \
    -d "$POSTGRES_DB" \
    -c "SELECT current_timestamp;"
```

---

# Part I — dbt

## 11. Create the dbt project directory

Run:

```bash
mkdir -p ~/data-platform-lab/dbt
cd ~/data-platform-lab/dbt
```

Initialize:

```bash
dbt init datalab
```

When dbt asks you to select an adapter, choose:

```text
postgres
```

It may prompt for database connection information.

We'll replace the generated profile ourselves afterward.

---

# 12. Understand the dbt configuration

dbt has two major configuration files:

```text
dbt_project.yml
```

which describes the project, and:

```text
~/.dbt/profiles.yml
```

which describes database connectivity.

This separation is intentional:

```text
Git repository
│
└── dbt_project.yml
       project definition


User/server configuration
│
└── ~/.dbt/profiles.yml
       credentials
```

---

# 13. Configure dbt `profiles.yml`

Create the directory if necessary:

```bash
mkdir -p ~/.dbt
```

Edit:

```bash
nano ~/.dbt/profiles.yml
```

Use:

```yaml
datalab:

  target: dev

  outputs:

    dev:

      type: postgres

      host: "{{ env_var('POSTGRES_HOST') }}"

      port: "{{ env_var('POSTGRES_PORT') | int }}"

      user: "{{ env_var('POSTGRES_USER') }}"

      password: "{{ env_var('POSTGRES_PASSWORD') }}"

      dbname: "{{ env_var('POSTGRES_DB') }}"

      schema: dbt_dev

      threads: 4
```

This avoids storing the password directly in `profiles.yml`.

---

# 14. Check `dbt_project.yml`

Move into your dbt project:

```bash
cd ~/data-platform-lab/dbt/datalab
```

Open:

```bash
nano dbt_project.yml
```

A simple version is:

```yaml
name: 'datalab'

version: '1.0.0'

config-version: 2

profile: 'datalab'

model-paths:
  - "models"

analysis-paths:
  - "analyses"

test-paths:
  - "tests"

seed-paths:
  - "seeds"

macro-paths:
  - "macros"

snapshot-paths:
  - "snapshots"

clean-targets:
  - "target"
  - "dbt_packages"

models:

  datalab:

    staging:

      +materialized: view

    marts:

      +materialized: table
```

---

# 15. Test the dbt connection

Remember to load `.env`:

```bash
cd ~/data-platform-lab

set -a
source .env
set +a
```

Then:

```bash
cd dbt/datalab
```

Run:

```bash
dbt debug
```

The important result is:

```text
Connection test: OK connection ok
```

You should ultimately get:

```text
All checks passed!
```

At this point:

```text
dbt
 │
 │ TCP 5432
 ▼
192.168.250.1
 │
 ▼
Windows
 │
 ▼
Docker Desktop
 │
 ▼
PostgreSQL
```

is working.

---

# 16. Create database schemas for the lab

Connect:

```bash
PGPASSWORD="$POSTGRES_PASSWORD" \
psql \
    -h "$POSTGRES_HOST" \
    -U "$POSTGRES_USER" \
    -d "$POSTGRES_DB"
```

Create schemas:

```sql
CREATE SCHEMA IF NOT EXISTS raw;

CREATE SCHEMA IF NOT EXISTS staging;

CREATE SCHEMA IF NOT EXISTS marts;

CREATE SCHEMA IF NOT EXISTS dbt_dev;
```

Check:

```sql
\dn
```

Exit:

```text
\q
```

---

# 17. Create a simple source table

For learning, create:

```sql
CREATE TABLE IF NOT EXISTS raw.customers
(
    customer_id BIGINT PRIMARY KEY,
    first_name  VARCHAR(100),
    last_name   VARCHAR(100),
    email       VARCHAR(255),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Insert some data:

```sql
INSERT INTO raw.customers
(
    customer_id,
    first_name,
    last_name,
    email
)
VALUES
    (1, 'Alice', 'Anderson', 'alice@example.com'),
    (2, 'Bob', 'Baker', 'bob@example.com'),
    (3, 'Carol', 'Carter', 'carol@example.com')
ON CONFLICT (customer_id) DO NOTHING;
```

---

# 18. Create dbt source definition

Inside:

```bash
cd ~/data-platform-lab/dbt/datalab
```

Create directories:

```bash
mkdir -p models/staging
mkdir -p models/marts
```

Create:

```bash
nano models/staging/sources.yml
```

Use:

```yaml
version: 2

sources:

  - name: raw
    schema: raw

    tables:

      - name: customers

        columns:

          - name: customer_id

            tests:
              - not_null
              - unique
```

---

# 19. Create a staging model

Create:

```bash
nano models/staging/stg_customers.sql
```

Use:

```sql
select

    customer_id,

    first_name,

    last_name,

    lower(email) as email,

    created_at

from {{ source('raw', 'customers') }}
```

---

# 20. Create a mart model

Create:

```bash
nano models/marts/dim_customers.sql
```

Use:

```sql
select

    customer_id,

    first_name || ' ' || last_name as customer_name,

    email,

    created_at

from {{ ref('stg_customers') }}
```

This demonstrates dbt lineage:

```text
raw.customers
      │
      ▼
stg_customers
      │
      ▼
dim_customers
```

---

# 21. Run dbt

Run:

```bash
dbt run
```

Then:

```bash
dbt test
```

Or together:

```bash
dbt build
```

`dbt build` is generally the better workflow because it builds and tests the project as a dependency graph.

Check PostgreSQL:

```bash
PGPASSWORD="$POSTGRES_PASSWORD" \
psql \
    -h "$POSTGRES_HOST" \
    -U "$POSTGRES_USER" \
    -d "$POSTGRES_DB"
```

Then:

```sql
SELECT *
FROM dbt_dev.dim_customers;
```

---

# 22. Generate dbt documentation

Run:

```bash
dbt docs generate
```

Then:

```bash
dbt docs serve --host 0.0.0.0 --port 8081
```

Because dbt is running inside Ubuntu, you can access it from Ubuntu at:

```text
http://localhost:8081
```

And, assuming the Ubuntu VM has:

```text
192.168.250.10
```

from Windows:

```text
http://192.168.250.10:8081
```

You may need an Ubuntu firewall rule if UFW is enabled:

```bash
sudo ufw allow from 192.168.250.0/24 to any port 8081 proto tcp
```

---

# Part II — Dagster

## 23. Create the Dagster project directory

Run:

```bash
mkdir -p ~/data-platform-lab/dagster/datalab_orchestrator
cd ~/data-platform-lab/dagster/datalab_orchestrator
```

We'll start with a simple explicit project rather than hiding the architecture behind scaffolding.

Create:

```text
datalab_orchestrator/
│
├── definitions.py
├── assets.py
└── resources.py
```

---

# 24. Create the PostgreSQL Dagster resource

Create:

```bash
nano resources.py
```

Use:

```python
import os

from dagster import ConfigurableResource
import psycopg


class PostgresResource(ConfigurableResource):

    host: str
    port: int
    database: str
    user: str
    password: str

    def get_connection(self):

        return psycopg.connect(
            host=self.host,
            port=self.port,
            dbname=self.database,
            user=self.user,
            password=self.password,
        )


def postgres_resource():

    return PostgresResource(
        host=os.environ["POSTGRES_HOST"],
        port=int(os.environ["POSTGRES_PORT"]),
        database=os.environ["POSTGRES_DB"],
        user=os.environ["POSTGRES_USER"],
        password=os.environ["POSTGRES_PASSWORD"],
    )
```

---

# 25. Create a Dagster PostgreSQL test asset

Create:

```bash
nano assets.py
```

Use:

```python
import dagster as dg

from resources import PostgresResource


@dg.asset
def postgres_connection_test(
    context: dg.AssetExecutionContext,
    postgres: PostgresResource,
):

    with postgres.get_connection() as connection:

        with connection.cursor() as cursor:

            cursor.execute(
                """
                SELECT
                    current_database(),
                    current_user,
                    CURRENT_TIMESTAMP
                """
            )

            row = cursor.fetchone()

    context.log.info(
        f"Connected to database={row[0]}, "
        f"user={row[1]}, "
        f"time={row[2]}"
    )

    return {
        "database": row[0],
        "user": row[1],
        "timestamp": str(row[2]),
    }
```

---

# 26. Create Dagster Definitions

Create:

```bash
nano definitions.py
```

Use:

```python
import dagster as dg

from assets import postgres_connection_test
from resources import postgres_resource


defs = dg.Definitions(

    assets=[
        postgres_connection_test,
    ],

    resources={
        "postgres": postgres_resource(),
    },
)
```

---

# 27. Start Dagster

Return to:

```bash
cd ~/data-platform-lab
```

Activate the environment if necessary:

```bash
source .venv/bin/activate
```

Load variables:

```bash
set -a
source .env
set +a
```

Then:

```bash
cd dagster/datalab_orchestrator
```

Start Dagster:

```bash
dagster dev -f definitions.py
```

Dagster's current documentation describes it as an asset-oriented data orchestrator with integrated lineage and observability. ([Dagster Docs][4])

You should see a message indicating that the Dagster webserver is running.

Typically:

```text
http://127.0.0.1:3000
```

---

# 28. Open Dagster

Inside Ubuntu:

```text
http://localhost:3000
```

You should see the Dagster UI.

Find:

```text
postgres_connection_test
```

Select:

```text
Materialize
```

The event logs should show something similar to:

```text
Connected to database=datalab,
user=labuser,
time=...
```

You have now demonstrated:

```text
Dagster
   │
   │ psycopg
   ▼
192.168.250.1:5432
   │
   ▼
PostgreSQL container
```

---

# 29. Make Dagster accessible from Windows

Stop Dagster:

```text
Ctrl+C
```

Run it listening on all Ubuntu interfaces:

```bash
dagster dev \
    -f definitions.py \
    -h 0.0.0.0 \
    -p 3000
```

If UFW is enabled:

```bash
sudo ufw status
```

Allow only your DevLab subnet:

```bash
sudo ufw allow from 192.168.250.0/24 to any port 3000 proto tcp
```

Now from the Windows host browser:

```text
http://192.168.250.10:3000
```

Do **not** expose port 3000 through your Internet router.

---

# Part III — Integrate Dagster and dbt

This is where the architecture becomes much more useful.

Instead of:

```text
Dagster → separate Python transformations

and

dbt → separately launched transformations
```

you want:

```text
             Dagster
                │
                ▼
              dbt
                │
        ┌───────┴───────┐
        ▼               ▼
 stg_customers     dim_customers
        │               │
        └──── PostgreSQL┘
```

The `dagster-dbt` package exists specifically for integrating dbt projects into Dagster's asset graph. The current integration package is 0.29.22. ([PyPI][2])

---

# 30. First compile the dbt project

Before Dagster loads the dbt manifest:

```bash
cd ~/data-platform-lab/dbt/datalab
```

Run:

```bash
dbt parse
```

or:

```bash
dbt build
```

This creates:

```text
target/manifest.json
```

Dagster uses dbt's artifacts to understand the dbt graph.

---

# 31. Replace `assets.py` with dbt integration

Edit:

```bash
cd ~/data-platform-lab/dagster/datalab_orchestrator

nano assets.py
```

Use:

```python
from pathlib import Path

import dagster as dg

from dagster_dbt import (
    DbtCliResource,
    DbtProject,
    dbt_assets,
)


DBT_PROJECT_DIR = Path(
    "/home/YOUR_USERNAME/data-platform-lab/dbt/datalab"
)


dbt_project = DbtProject(
    project_dir=DBT_PROJECT_DIR,
)


@dbt_assets(
    manifest=dbt_project.manifest_path,
)
def datalab_dbt_assets(
    context: dg.AssetExecutionContext,
    dbt: DbtCliResource,
):

    yield from dbt.cli(
        ["build"],
        context=context,
    ).stream()
```

Replace:

```text
YOUR_USERNAME
```

with your actual Ubuntu user.

You can determine it with:

```bash
whoami
```

---

# 32. Update Dagster definitions

Edit:

```bash
nano definitions.py
```

Use:

```python
import dagster as dg

from dagster_dbt import DbtCliResource

from assets import (
    DBT_PROJECT_DIR,
    datalab_dbt_assets,
)


defs = dg.Definitions(

    assets=[
        datalab_dbt_assets,
    ],

    resources={

        "dbt": DbtCliResource(
            project_dir=DBT_PROJECT_DIR,
        ),

    },

)
```

---

# 33. Start integrated Dagster/dbt

From:

```bash
cd ~/data-platform-lab
```

activate:

```bash
source .venv/bin/activate
```

Load environment:

```bash
set -a
source .env
set +a
```

Run:

```bash
cd dagster/datalab_orchestrator
```

Then:

```bash
dagster dev \
    -f definitions.py \
    -h 0.0.0.0 \
    -p 3000
```

Open:

```text
http://localhost:3000
```

or from Windows:

```text
http://192.168.250.10:3000
```

Dagster should now display the dbt model graph.

Conceptually:

```text
raw.customers
      │
      ▼
stg_customers
      │
      ▼
dim_customers
```

You can materialize the dbt assets from Dagster.

---

# 34. Verify the complete stack

From Ubuntu:

### PostgreSQL

```bash
nc -vz 192.168.250.1 5432
```

### dbt

```bash
cd ~/data-platform-lab/dbt/datalab

dbt debug
```

Then:

```bash
dbt build
```

### Dagster

```bash
cd ~/data-platform-lab/dagster/datalab_orchestrator

dagster dev \
    -f definitions.py \
    -h 0.0.0.0 \
    -p 3000
```

### Database

```bash
PGPASSWORD="$POSTGRES_PASSWORD" \
psql \
    -h "$POSTGRES_HOST" \
    -U "$POSTGRES_USER" \
    -d "$POSTGRES_DB" \
    -c "SELECT * FROM dbt_dev.dim_customers;"
```

---

# 35. Recommended directory structure

Your lab should now look approximately like:

```text
~/data-platform-lab/
│
├── .venv/
│
├── .env
├── .gitignore
├── requirements.txt
│
├── dbt/
│   │
│   └── datalab/
│       │
│       ├── dbt_project.yml
│       │
│       ├── models/
│       │   │
│       │   ├── staging/
│       │   │   ├── sources.yml
│       │   │   └── stg_customers.sql
│       │   │
│       │   └── marts/
│       │       └── dim_customers.sql
│       │
│       ├── seeds/
│       ├── snapshots/
│       ├── macros/
│       └── target/
│
├── dagster/
│   │
│   └── datalab_orchestrator/
│       │
│       ├── definitions.py
│       ├── assets.py
│       └── resources.py
│
├── ingestion/
├── tests/
└── README.md
```

---

# 36. Make environment loading easier

Instead of running this every session:

```bash
set -a
source .env
set +a
```

create:

```bash
nano ~/data-platform-lab/activate.sh
```

with:

```bash
#!/usr/bin/env bash

set -e

PROJECT_DIR="$HOME/data-platform-lab"

cd "$PROJECT_DIR"

source .venv/bin/activate

set -a
source .env
set +a

echo
echo "Data Platform Lab activated"
echo
echo "Python:"
python --version

echo
echo "dbt:"
dbt --version

echo
echo "Dagster:"
dagster --version

echo
echo "PostgreSQL host:"
echo "${POSTGRES_HOST}:${POSTGRES_PORT}"
```

Make executable:

```bash
chmod +x activate.sh
```

Run:

```bash
source ~/data-platform-lab/activate.sh
```

---

# 37. Test without exposing the password

Create:

```bash
nano ~/data-platform-lab/test-environment.sh
```

Use:

```bash
#!/usr/bin/env bash

set -euo pipefail


echo "================================"
echo "Data Platform Connectivity Test"
echo "================================"


echo
echo "1. Windows host"

ping -c 1 "${POSTGRES_HOST}"


echo
echo "2. PostgreSQL TCP port"

nc -zv \
    "${POSTGRES_HOST}" \
    "${POSTGRES_PORT}"


echo
echo "3. PostgreSQL database"

PGPASSWORD="${POSTGRES_PASSWORD}" \
psql \
    -h "${POSTGRES_HOST}" \
    -p "${POSTGRES_PORT}" \
    -U "${POSTGRES_USER}" \
    -d "${POSTGRES_DB}" \
    -c "SELECT CURRENT_TIMESTAMP;"


echo
echo "4. dbt"

cd "$HOME/data-platform-lab/dbt/datalab"

dbt debug


echo
echo "Environment OK"
```

Make executable:

```bash
chmod +x test-environment.sh
```

Then:

```bash
source activate.sh
./test-environment.sh
```

---

# 38. Git configuration

Commit these:

```text
dbt/
dagster/
README.md
requirements.txt
.gitignore
```

Never commit:

```text
.env
.venv/
target/
logs/
```

Verify:

```bash
git status
```

Your `.env` should not appear.

---

# 39. How this fits the GitHub Actions pipeline

The pipeline we discussed previously now has a very natural progression:

```text
Developer
    │
    ▼
GitHub
    │
    ▼
Pull Request
    │
    ├── Python tests
    ├── dbt parse
    ├── dbt compile
    ├── dbt tests
    └── Dagster validation
            │
            ▼
          main
            │
            ▼
GitHub Actions
            │
            ▼
Self-hosted Ubuntu runner
            │
      ┌─────┴──────┐
      │            │
      ▼            ▼
   Dagster         dbt
      │            │
      └──────┬─────┘
             ▼
       PostgreSQL
       Docker/Windows
```

A later GitHub Actions validation step can run:

```yaml
- name: dbt parse
  run: |
    dbt parse \
      --project-dir dbt/datalab
```

and your deployment runner can invoke Dagster/dbt against the actual local PostgreSQL database.

---

# 40. Resource allocation on your VM

For your:

```text
8 vCPU
16 GB RAM
Ubuntu 26.04
```

this particular stack is light.

Typical development use might look roughly like:

```text
Ubuntu Desktop           2–4 GB
Dagster                  <1–2 GB
dbt                      workload dependent
Python tooling           <1 GB
PostgreSQL               Windows/Docker side
Airflow                  Windows/Docker side
---------------------------------------------
Ubuntu VM normally       well below 16 GB
```

Therefore you still have room in the Ubuntu VM for:

```text
PySpark
Terraform
GitHub Actions runner
Rust
Kubernetes tooling
AWS CLI
Azure CLI
gcloud CLI
```

without immediately increasing VM memory.

---

## Final architecture

At this point your lab becomes:

```text
                         GITHUB
                            │
                            ▼
                    GitHub Actions
                            │
                            ▼
                  Ubuntu 26.04 Hyper-V
                  192.168.250.10
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
       Python            Dagster             dbt
          │                 │                 │
          │                 └────────┬────────┘
          │                          │
          └──────────────────────────┤
                                     │
                               TCP 5432
                                     │
                                     ▼
                              192.168.250.1
                                     │
                              Windows 11 Pro
                                     │
                              Docker Desktop
                                     │
                                PostgreSQL
                                     │
                                  datalab
                                     │
                 ┌───────────────────┼──────────────────┐
                 ▼                   ▼                  ▼
                raw               dbt_dev             marts
```

The important design choice is that **Dagster orchestrates and dbt transforms**. Don't duplicate dbt's transformation logic in Dagster. Let Dagster own scheduling, dependencies, execution and observability, while dbt owns SQL models, tests and transformation lineage. That gives this lab a much more realistic modern data-platform architecture. ([Dagster Docs][4])

[1]: https://pypi.org/project/dagster/1.13.22/?utm_source=chatgpt.com "dagster · PyPI"
[2]: https://pypi.org/project/dagster-dbt/0.29.22/?utm_source=chatgpt.com "dagster-dbt · PyPI"
[3]: https://pypi.org/project/dbt-postgres/?utm_source=chatgpt.com "dbt-postgres · PyPI"
[4]: https://docs.dagster.io/?utm_source=chatgpt.com "Overview | Dagster Docs"
