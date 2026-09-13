# Phase 1 — Python Data Engineering Foundation

## Objective

Build a simple local data engineering environment around four core technologies:

```text
Linux
  ↓
Git
  ↓
Python
  ↓
PostgreSQL
```

The environment should support a basic end-to-end data pipeline where:

* **Linux** provides the development platform.
* **Git** provides source control.
* **Python** performs ingestion, transformation, testing, and database interaction.
* **PostgreSQL** provides both the initial operational source database and analytical warehouse.
* **Docker** runs PostgreSQL so that the database does not need to be permanently installed on the Ubuntu host.

The resulting architecture is:

```text
                   Git Repository
                        │
                        ▼
                Python Application
                 /              \
                /                \
               ▼                  ▼
      PostgreSQL Source     PostgreSQL Warehouse
               \                  /
                \                /
                 └──── Docker ───┘
```

For the first implementation, a single PostgreSQL container can contain separate databases or schemas representing the source and warehouse.

---

# 1. Prepare the Linux Development Environment

The following assumes Ubuntu Linux.

In the DE_Portfolio folder create a **"Playground"** folder then change to that folder

```bash
mkdir Playground
cd Playground
```

Update the operating system:

```bash
sudo apt update
sudo apt upgrade -y
```

Install common development utilities:

```bash
sudo apt install -y \
    curl \
    wget \
    git \
    unzip \
    build-essential \
    ca-certificates \
    gnupg \
    lsb-release \
    tree \
    jq
```

Verify the operating system:

```bash
lsb_release -a
```

Check available CPU, memory, and disk space:

```bash
lscpu
free -h
df -h
```

At this point Linux becomes the base platform for everything else.

---

# 2. Configure Git

Verify Git:

```bash
git --version
```

Configure your Git identity:

```bash
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
```

Set the default branch name:

```bash
git config --global init.defaultBranch main
```

Verify the configuration:

```bash
git config --global --list
```

Optional but recommended:

```bash
git config --global core.editor "nano"
```

If VS Code is installed:

```bash
git config --global core.editor "code --wait"
```

---

# 3. Create the Project Directory

Create a dedicated project:

```bash
mkdir python-data-engineer
cd python-data-engineer
```

Initialize Git:

```bash
git init
```

Create the initial structure:

```bash
mkdir -p \
    src/ingestion \
    src/transform \
    src/database \
    tests \
    sql/source \
    sql/warehouse \
    data/raw \
    data/output \
    docker
```

Create initial files:

```bash
touch README.md
touch .gitignore
touch requirements.txt
touch docker-compose.yml
touch .env.example

touch src/__init__.py
touch src/ingestion/__init__.py
touch src/transform/__init__.py
touch src/database/__init__.py
```

The project should now resemble:

```text
/Playground/python-data-engineer/
├── README.md
├── docker-compose.yml
├── requirements.txt
├── .env.example
├── .gitignore
│
├── src/
│   ├── __init__.py
│   ├── ingestion/
│   │   └── __init__.py
│   ├── transform/
│   │   └── __init__.py
│   └── database/
│       └── __init__.py
│
├── sql/
│   ├── source/
│   └── warehouse/
│
├── data/
│   ├── raw/
│   └── output/
│
├── docker/
│
└── tests/
```

Check it with:

```bash
tree
```

---

# 4. Install Python

Install Python and the virtual environment tools:

```bash
sudo apt install -y \
    python3 \
    python3-pip \
    python3-venv \
    python3-dev
```

Verify:

```bash
python3 --version
pip3 --version
```

Do not install project Python packages globally.

Instead, create a virtual environment, verify you are in the right place

```bash
PWD 
```
If .../Playground/python-data-engineer is the current working directory,
then create the virtual environment:

```bash
python3 -m venv .venv
```

Activate it:

```bash
source .venv/bin/activate
```

The shell prompt should now normally show:

```text
(.venv)
```

Upgrade Python packaging tools:

```bash
python -m pip install --upgrade pip setuptools wheel
```

Verify that Python is coming from the virtual environment:

```bash
which python
```

The result should resemble:

```text
/home/<user>/data-platform-lab/python-data-engineer/.venv/bin/python
```

---

# 5. Install the Initial Python Data Engineering Packages

Install a small set of useful packages:

```bash
pip install \
    pandas \
    sqlalchemy \
    psycopg2-binary \
    requests \
    python-dotenv \
    pytest
```

Save the dependencies:

```bash
pip freeze > requirements.txt
```

The major packages have the following responsibilities:

| Package           | Purpose                                        |
| ----------------- | ---------------------------------------------- |
| `pandas`          | Data manipulation and transformation           |
| `sqlalchemy`      | Database abstraction and connection management |
| `psycopg2-binary` | PostgreSQL database driver                     |
| `requests`        | API ingestion                                  |
| `python-dotenv`   | Environment variable management                |
| `pytest`          | Automated Python testing                       |

Later projects can add packages as needed.

---

# 6. Install Docker

PostgreSQL will run in Docker rather than directly on Ubuntu.

This gives several advantages:

* easy database creation and removal;
* reproducible environments;
* isolation from the host operating system;
* simple PostgreSQL version changes;
* easy migration later to Kubernetes or cloud infrastructure.

Install Docker:

```bash
sudo apt install -y docker.io docker-compose-v2
```

Enable Docker:

```bash
sudo systemctl enable --now docker
```

Verify:

```bash
sudo docker --version
```

Test Docker:

```bash
sudo docker run --rm hello-world
```

---

# 7. Allow Docker to Run Without `sudo`

Add your account to the Docker group:

```bash
sudo usermod -aG docker "$USER"
```

Apply the group membership:

```bash
newgrp docker
```

Verify:

```bash
docker ps
```

You should no longer need:

```bash
sudo docker ...
```

for normal Docker operations.

---

# 8. Create the PostgreSQL Docker Configuration

Create the following `docker-compose.yml`:

```yaml
services:

  postgres:
    image: postgres:16

    container_name: de-postgres

    restart: unless-stopped

    environment:
      POSTGRES_USER: dataengineer
      POSTGRES_PASSWORD: dataengineer
      POSTGRES_DB: data_platform

    ports:
      - "5432:5432"

    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./sql:/sql

volumes:
  postgres_data:
```

This creates:

```text
Ubuntu Host
    │
    └── Docker
          │
          └── PostgreSQL
                │
                └── Port 5432
```

The PostgreSQL data itself lives in a Docker volume:

```text
postgres_data
```

This means stopping or recreating the container will not automatically destroy the database.

---

# 9. Start PostgreSQL

From the project root:

```bash
docker compose up -d
```

Check the container:

```bash
docker ps
```

You should see something similar to:

```text
CONTAINER ID   IMAGE         STATUS        PORTS
xxxxxxxxxxxx   postgres:16   Up ...        0.0.0.0:5432->5432/tcp
```

Check the PostgreSQL startup logs:

```bash
docker logs de-postgres
```

---

# 10. Connect to PostgreSQL

You do not need PostgreSQL server installed on Ubuntu.

You can initially use PostgreSQL's own `psql` client from inside the container:

```bash
docker exec -it de-postgres \
    psql -U dataengineer -d data_platform
```

You should receive a prompt similar to:

```text
data_platform=#
```

Test the database:

```sql
SELECT version();
```

List databases:

```sql
\l
```

Exit:

```sql
\q
```

---

# 11. Create Source and Warehouse Schemas

For the initial project, use one PostgreSQL instance but logically separate operational and analytical data.

Reconnect:

```bash
docker exec -it de-postgres \
    psql -U dataengineer -d data_platform
```

Create schemas:

```sql
CREATE SCHEMA source;
CREATE SCHEMA warehouse;
```

Verify:

```sql
\dn
```

The architecture now becomes:

```text
PostgreSQL
│
├── source
│   └── Operational Data
│
└── warehouse
    └── Analytical Data
```

This provides a useful simulation of two different layers without requiring two database servers.

---

# 12. Create an Operational Source Table

Create:

```text
sql/source/create_customers.sql
```

with:

```sql
CREATE TABLE IF NOT EXISTS source.customers
(
    customer_id INTEGER PRIMARY KEY,
    first_name  VARCHAR(100),
    last_name   VARCHAR(100),
    email       VARCHAR(255),
    state       VARCHAR(2),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Run it:

```bash
docker exec -i de-postgres \
    psql -U dataengineer -d data_platform \
    < sql/source/create_customers.sql
```

---

# 13. Load Sample Operational Data

Create:

```text
sql/source/load_customers.sql
```

```sql
INSERT INTO source.customers
(
    customer_id,
    first_name,
    last_name,
    email,
    state
)
VALUES
    (1, 'John', 'Smith', 'john@example.com', 'TX'),
    (2, 'Susan', 'Jones', 'susan@example.com', 'CA'),
    (3, 'Robert', 'Williams', 'robert@example.com', 'TX');
```

Load it:

```bash
docker exec -i de-postgres \
    psql -U dataengineer -d data_platform \
    < sql/source/load_customers.sql
```

Verify:

```bash
docker exec -it de-postgres \
    psql -U dataengineer -d data_platform \
    -c "SELECT * FROM source.customers;"
```

---

# 14. Create the Analytical Warehouse Table

Create:

```text
sql/warehouse/create_dim_customer.sql
```

```sql
CREATE TABLE IF NOT EXISTS warehouse.dim_customer
(
    customer_key SERIAL PRIMARY KEY,
    customer_id  INTEGER NOT NULL,
    full_name    VARCHAR(201),
    email        VARCHAR(255),
    state        VARCHAR(2),
    load_time    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Run it:

```bash
docker exec -i de-postgres \
    psql -U dataengineer -d data_platform \
    < sql/warehouse/create_dim_customer.sql
```

The data flow will eventually be:

```text
source.customers
       │
       │ Extract
       ▼
     Python
       │
       │ Transform
       ▼
warehouse.dim_customer
```

---

# 15. Create Environment Variables

Do not hard-code passwords into Python programs.

Create:

```text
.env
```

```bash
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DATABASE=data_platform
POSTGRES_USER=dataengineer
POSTGRES_PASSWORD=dataengineer
```

Create an example version for Git:

```text
.env.example
```

```bash
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DATABASE=data_platform
POSTGRES_USER=your_username
POSTGRES_PASSWORD=your_password
```

---

# 16. Configure `.gitignore`

Create:

```gitignore
# Python
__pycache__/
*.py[cod]

# Virtual environments
.venv/
venv/

# Environment variables
.env

# Testing
.pytest_cache/

# IDEs
.vscode/
.idea/

# OS
.DS_Store

# Generated data
data/output/*
```

The real `.env` file should never be committed.

---

# 17. Create a Python Database Connection

Create:

```text
src/database/connection.py
```

```python
import os

from dotenv import load_dotenv
from sqlalchemy import create_engine


load_dotenv()


def get_engine():
    host = os.getenv("POSTGRES_HOST")
    port = os.getenv("POSTGRES_PORT")
    database = os.getenv("POSTGRES_DATABASE")
    user = os.getenv("POSTGRES_USER")
    password = os.getenv("POSTGRES_PASSWORD")

    connection_string = (
        f"postgresql+psycopg2://"
        f"{user}:{password}@"
        f"{host}:{port}/{database}"
    )

    return create_engine(connection_string)
```

This centralizes database connectivity.

Other Python programs can now import the connection instead of defining their own credentials.

---

# 18. Test Python-to-PostgreSQL Connectivity

Create:

```text
src/database/test_connection.py
```

```python
from sqlalchemy import text

from connection import get_engine


engine = get_engine()

with engine.connect() as connection:
    result = connection.execute(
        text("SELECT version()")
    )

    print(result.scalar())
```

Run:

```bash
python src/database/test_connection.py
```

A successful response should display the PostgreSQL version.

---

# 19. Create the First ETL Pipeline

Create:

```text
src/pipeline.py
```

```python
import pandas as pd
from sqlalchemy import text

from database.connection import get_engine


def extract(engine):
    query = """
        SELECT
            customer_id,
            first_name,
            last_name,
            email,
            state
        FROM source.customers
    """

    return pd.read_sql(query, engine)


def transform(df):
    df["full_name"] = (
        df["first_name"]
        + " "
        + df["last_name"]
    )

    return df[
        [
            "customer_id",
            "full_name",
            "email",
            "state",
        ]
    ]


def load(df, engine):

    with engine.begin() as connection:
        connection.execute(
            text("TRUNCATE TABLE warehouse.dim_customer")
        )

    df.to_sql(
        name="dim_customer",
        schema="warehouse",
        con=engine,
        if_exists="append",
        index=False,
    )


def main():

    engine = get_engine()

    df = extract(engine)

    transformed_df = transform(df)

    load(transformed_df, engine)

    print(
        f"Pipeline completed successfully. "
        f"{len(transformed_df)} records loaded."
    )


if __name__ == "__main__":
    main()
```

Run it from the project root:

```bash
python -m src.pipeline
```

---

# 20. Verify the Warehouse

Query the warehouse:

```bash
docker exec -it de-postgres \
    psql -U dataengineer -d data_platform \
    -c "SELECT * FROM warehouse.dim_customer;"
```

The result should resemble:

```text
 customer_key | customer_id |    full_name     |       email        | state
--------------+-------------+------------------+--------------------+-------
 1            | 1           | John Smith       | john@example.com   | TX
 2            | 2           | Susan Jones      | susan@example.com  | CA
 3            | 3           | Robert Williams  | robert@example.com | TX
```

You now have a functioning ETL pipeline.

---

# 21. Add a Basic Python Test

Create:

```text
tests/test_transform.py
```

```python
import pandas as pd

from src.pipeline import transform


def test_transform():

    source = pd.DataFrame(
        [
            {
                "customer_id": 1,
                "first_name": "John",
                "last_name": "Smith",
                "email": "john@example.com",
                "state": "TX",
            }
        ]
    )

    result = transform(source)

    assert result.iloc[0]["full_name"] == "John Smith"
    assert result.iloc[0]["state"] == "TX"
```

Run:

```bash
pytest
```

This introduces automated testing into the project before the pipeline becomes more complicated.

---

# 22. Commit the Initial Environment to Git

Review the files:

```bash
git status
```

Stage them:

```bash
git add .
```

Commit:

```bash
git commit -m "Build initial Python PostgreSQL data pipeline"
```

View the history:

```bash
git log --oneline
```

---

# 23. Normal Development Startup Procedure

After the initial installation, a normal development session becomes very simple.

Enter the project:

```bash
cd ~/data-platform-lab/python-data-engineer
```

Activate Python:

```bash
source .venv/bin/activate
```

Start PostgreSQL:

```bash
docker compose up -d
```

Check it:

```bash
docker compose ps
```

Run tests:

```bash
pytest
```

Run the pipeline:

```bash
python -m src.pipeline
```

Check the warehouse:

```bash
docker exec -it de-postgres \
    psql -U dataengineer -d data_platform
```

---

# 24. Shut Down the Environment

Stop PostgreSQL without deleting its data:

```bash
docker compose stop
```

Restart it later:

```bash
docker compose start
```

Or shut down and remove the container:

```bash
docker compose down
```

The database persists because its files are stored in:

```text
postgres_data
```

To intentionally destroy the database and start over:

```bash
docker compose down -v
```

> **Warning:** `-v` deletes the PostgreSQL Docker volume and therefore deletes the database contents.

---

# 25. Final Environment

The completed Phase 1 environment looks like this:

```text
Ubuntu Linux
│
├── Git
│   └── Source Control
│
├── Python Virtual Environment
│   │
│   ├── pandas
│   ├── SQLAlchemy
│   ├── psycopg2
│   ├── requests
│   ├── pytest
│   └── python-dotenv
│
└── Docker
    │
    └── PostgreSQL
        │
        ├── source
        │   └── customers
        │
        └── warehouse
            └── dim_customer
```

The operational workflow is:

```text
Operational Data
      │
      ▼
PostgreSQL
source schema
      │
      │ EXTRACT
      ▼
    Python
      │
      │ TRANSFORM
      ▼
    Python
      │
      │ LOAD
      ▼
PostgreSQL
warehouse schema
      │
      ▼
Analytical Data
```

## Primary Tools

* **Linux** — host operating system and development platform
* **Git** — version control
* **Python** — ingestion, transformation, automation, and testing
* **PostgreSQL** — operational source and analytical warehouse
* **Docker** — isolated PostgreSQL runtime

## Skills Practiced

This first environment provides hands-on practice with:

* Linux command-line operations
* Git repositories and version control
* Python virtual environments
* Python package management
* PostgreSQL SQL
* Docker containers and volumes
* environment-variable management
* database connectivity
* SQLAlchemy
* ETL pipeline construction
* source-to-target transformations
* basic dimensional modeling
* automated testing with `pytest`
* separation of source and analytical data

The important architectural principle is that **PostgreSQL is infrastructure, not part of the laptop's permanent host configuration**. That pattern makes the next phases—dbt, Airflow, Jenkins, Spark, Kubernetes, Terraform, and eventually cloud services—much easier to introduce without redesigning the development environment.
