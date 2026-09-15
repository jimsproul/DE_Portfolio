For your lab, I would use **GitHub Actions to deploy automatically whenever `main` changes**, but I would separate **build/test** from **deployment**:

```text
Developer Push / Merge
        │
        ▼
GitHub Repository
        │
        ▼
GitHub Actions
        │
        ├── Checkout
        ├── Validate
        ├── Run Tests
        └── Deploy
                │
                ▼
        Self-hosted Runner
        Ubuntu 26.04 VM
                │
                ▼
        Docker / PostgreSQL / Airflow
```

Because your deployment target is inside your Windows 11/Hyper-V lab, the cleanest design is to install a **GitHub self-hosted runner on the Ubuntu VM**. GitHub supports Ubuntu 20.04 and later for self-hosted runners, so Ubuntu 26.04 is within the supported OS family. ([GitHub Docs][1])

One security qualification matters: GitHub explicitly recommends self-hosted runners primarily for **private repositories**, because workflow code can execute directly on the runner and potentially expose secrets or other resources. ([GitHub Docs][2])

---

# 1. Target architecture

I recommend:

```text
Windows 11 Pro
│
├── Docker Desktop
│    │
│    ├── PostgreSQL
│    └── Airflow
│
└── Hyper-V
     │
     └── Ubuntu 26.04
          │
          ├── Git
          ├── Python
          ├── dbt
          ├── PostgreSQL client
          └── GitHub Actions self-hosted runner
                    │
                    │ deploy
                    ▼
             Local lab services
```

The GitHub runner is effectively your **deployment agent**.

---

# 2. Repository layout

A useful repository structure would be:

```text
data-platform-lab/
│
├── .github/
│   └── workflows/
│       └── deploy.yml
│
├── dags/
│   └── example_pipeline.py
│
├── src/
│   ├── ingestion/
│   └── transformations/
│
├── sql/
│
├── dbt/
│
├── tests/
│
├── docker/
│
├── scripts/
│   └── deploy.sh
│
├── requirements.txt
├── README.md
└── .gitignore
```

The important directory is:

```text
.github/workflows/
```

Any YAML files placed there define GitHub Actions workflows.

---

# 3. Protect the `main` branch

Before automating deployments, treat `main` as your deployable branch.

On GitHub:

```text
Repository
→ Settings
→ Branches
→ Branch protection rules
```

Create a rule for:

```text
main
```

I recommend:

```text
Require pull request before merging

Require status checks to pass

Require branches to be up to date before merging

Do not allow force pushes

Do not allow branch deletion
```

Then your process becomes:

```text
feature branch
      │
      ▼
Pull Request
      │
      ▼
Tests
      │
      ▼
Merge to main
      │
      ▼
DEPLOYMENT
```

That is substantially safer than deploying every developer commit directly.

---

# 4. Create a GitHub Environment

GitHub Environments are useful for deployment-specific variables, secrets, URLs and approval controls. Jobs must reference the environment before they can access its environment-level secrets. ([GitHub Docs][3])

Go to:

```text
Repository
→ Settings
→ Environments
→ New environment
```

Create:

```text
development
```

Eventually you could have:

```text
development
test
production
```

For your current lab:

```text
development
```

is enough.

---

# 5. Install the GitHub Actions runner on Ubuntu

Inside your Ubuntu 26.04 Hyper-V VM, first create a dedicated service account.

```bash
sudo useradd \
    --create-home \
    --shell /bin/bash \
    github-runner
```

You could use your normal account, but a dedicated runner identity provides better separation.

Create a working directory:

```bash
sudo mkdir -p /opt/actions-runner
```

Change ownership:

```bash
sudo chown github-runner:github-runner /opt/actions-runner
```

---

# 6. Get the current runner installation commands

On GitHub:

```text
Repository
→ Settings
→ Actions
→ Runners
→ New self-hosted runner
```

Choose:

```text
Linux
x64
```

GitHub will display commands appropriate for the **current runner release**, including the registration token. GitHub recommends using these generated commands when adding the runner. ([GitHub Docs][4])

You will see instructions conceptually like:

```bash
mkdir actions-runner
cd actions-runner
```

followed by downloading a runner archive:

```bash
curl -o actions-runner-linux-x64-<VERSION>.tar.gz \
    -L https://github.com/actions/runner/releases/download/<VERSION>/actions-runner-linux-x64-<VERSION>.tar.gz
```

and extracting:

```bash
tar xzf ./actions-runner-linux-x64-<VERSION>.tar.gz
```

Do **not** blindly copy a version number from an old tutorial. Use the command GitHub generates for your repository.

---

# 7. Register the runner

Switch to the service account:

```bash
sudo -iu github-runner
```

Then:

```bash
cd /opt/actions-runner
```

Run GitHub's generated configuration command.

It will resemble:

```bash
./config.sh \
    --url https://github.com/YOUR_USERNAME/YOUR_REPOSITORY \
    --token YOUR_TEMPORARY_REGISTRATION_TOKEN
```

The registration token is temporary.

You will be prompted for:

```text
Runner group
Runner name
Additional labels
Work folder
```

I recommend:

```text
Runner name:
ubuntu26-devlab

Labels:
ubuntu26,devlab,deployment

Work folder:
_work
```

The runner will automatically also receive standard labels such as:

```text
self-hosted
Linux
X64
```

GitHub allows workflows to target self-hosted runners using these labels. ([GitHub Docs][5])

---

# 8. Test the runner manually

Still as `github-runner`:

```bash
cd /opt/actions-runner
```

Run:

```bash
./run.sh
```

You should eventually see something like:

```text
Connected to GitHub

Listening for Jobs
```

GitHub documents this as the indication that the runner has successfully registered and is available for jobs. ([GitHub Docs][4])

In GitHub, check:

```text
Settings
→ Actions
→ Runners
```

You should see:

```text
ubuntu26-devlab

Idle
```

Stop the interactive runner:

```text
Ctrl+C
```

---

# 9. Install the runner as a Linux service

From:

```bash
cd /opt/actions-runner
```

the runner package contains:

```bash
svc.sh
```

Install the service:

```bash
sudo ./svc.sh install github-runner
```

Start it:

```bash
sudo ./svc.sh start
```

Check:

```bash
sudo ./svc.sh status
```

Now the runner starts automatically with Ubuntu.

GitHub requires the self-hosted runner application to be running and able to make outbound HTTPS connections to GitHub in order to receive jobs. ([GitHub Docs][1])

---

# 10. Test connectivity to GitHub

From Ubuntu:

```bash
curl -I https://github.com
```

You need outbound HTTPS:

```text
TCP 443
```

The runner initiates outbound communication, so you normally do **not** need to expose an inbound GitHub Actions port through your router or Windows firewall. GitHub documents outbound HTTPS as a runner communication requirement. ([GitHub Docs][1])

---

# 11. Verify Git is installed

```bash
git --version
```

If necessary:

```bash
sudo apt update
sudo apt install -y git
```

---

# 12. Install the deployment dependencies

Since your deployment is going to touch PostgreSQL and Airflow, install:

```bash
sudo apt install -y \
    git \
    curl \
    jq \
    postgresql-client \
    python3 \
    python3-pip \
    python3-venv
```

---

# 13. Create the deployment location

Let's put deployed application content under:

```text
/opt/datalab
```

Create it:

```bash
sudo mkdir -p /opt/datalab
```

Give ownership to the deployment account:

```bash
sudo chown github-runner:github-runner /opt/datalab
```

Your deployment destination becomes:

```text
/opt/datalab/
```

---

# 14. Create a first deployment script

In your repository create:

```text
scripts/deploy.sh
```

For example:

```bash
#!/usr/bin/env bash

set -euo pipefail

echo "========================================"
echo "Starting Data Lab deployment"
echo "========================================"

DEPLOY_DIR="/opt/datalab"

echo "Deployment directory:"
echo "${DEPLOY_DIR}"

mkdir -p "${DEPLOY_DIR}"

echo "Copying application files..."

rsync -av \
    --delete \
    --exclude ".git" \
    --exclude ".github" \
    ./ "${DEPLOY_DIR}/"

echo "Deployment complete."
```

Make it executable:

```bash
chmod +x scripts/deploy.sh
```

Commit that permission change:

```bash
git add scripts/deploy.sh
git commit -m "Add deployment script"
```

---

# 15. Install rsync

Because the deployment script uses it:

```bash
sudo apt install -y rsync
```

---

# 16. Create the GitHub Actions workflow

Create:

```text
.github/workflows/deploy.yml
```

Start with:

```yaml
name: Deploy Data Platform Lab

on:
  push:
    branches:
      - main

  workflow_dispatch:

permissions:
  contents: read

jobs:

  test:
    name: Validate and Test

    runs-on: ubuntu-latest

    steps:

      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Show repository
        run: |
          pwd
          ls -la

      - name: Set up Python
        uses: actions/setup-python@v6
        with:
          python-version: "3.13"

      - name: Install Python dependencies
        if: ${{ hashFiles('requirements.txt') != '' }}
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run tests
        run: |
          if [ -d tests ]; then
              python -m pytest tests
          else
              echo "No tests directory found."
          fi


  deploy:

    name: Deploy to DevLab

    needs:
      - test

    runs-on:
      - self-hosted
      - Linux
      - X64
      - devlab

    environment:
      name: development
      url: http://192.168.250.1:8080

    steps:

      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Show deployment information
        run: |
          echo "Deploying commit:"
          echo "${GITHUB_SHA}"

          echo "Branch:"
          echo "${GITHUB_REF}"

          echo "Runner:"
          hostname

      - name: Deploy
        run: |
          chmod +x scripts/deploy.sh
          ./scripts/deploy.sh
```

The key trigger is:

```yaml
on:
  push:
    branches:
      - main
```

That means:

> Whenever a commit reaches `main`, run this workflow.

GitHub's workflow syntax supports branch filtering on `push` events exactly for this purpose. ([GitHub Docs][5])

---

# 17. Why I separated test and deployment

Notice:

```yaml
test:
```

runs on:

```yaml
runs-on: ubuntu-latest
```

while:

```yaml
deploy:
```

runs on:

```yaml
runs-on:
  - self-hosted
  - devlab
```

That produces:

```text
GitHub-hosted VM
      │
      │ build/test
      │
      ▼
PASS
      │
      ▼
Self-hosted Ubuntu
      │
      │ deploy only
      ▼
Lab
```

This is preferable to running arbitrary build/test code on your persistent deployment server.

GitHub warns that self-hosted runners are not guaranteed to start from a clean environment for each job and can retain state across runs. ([GitHub Docs][2])

---

# 18. Add pytest

If you're using Python, put this in:

```text
requirements.txt
```

or a development requirements file:

```text
pytest
```

A trivial test could be:

```text
tests/test_basic.py
```

```python
def test_basic():
    assert 1 + 1 == 2
```

Then the pipeline can demonstrate:

```text
Push
  ↓
Checkout
  ↓
Install dependencies
  ↓
pytest
  ↓
Deploy
```

---

# 19. Commit the workflow

```bash
git add .
```

```bash
git commit -m "Add GitHub Actions deployment pipeline"
```

Push:

```bash
git push origin main
```

Immediately go to:

```text
GitHub
→ Repository
→ Actions
```

You should see:

```text
Deploy Data Platform Lab
```

running.

---

# 20. Pipeline execution sequence

The workflow now operates as:

```text
git push origin main
          │
          ▼
GitHub receives commit
          │
          ▼
push event matches:
branch = main
          │
          ▼
GitHub Action starts
          │
          ▼
TEST JOB
          │
          ├── checkout
          ├── Python
          ├── dependencies
          └── pytest
          │
       success?
        /    \
      no      yes
      │        │
      STOP     ▼
             DEPLOY
               │
               ▼
        self-hosted runner
               │
               ▼
        /opt/datalab
```

The `needs` statement enforces the dependency:

```yaml
needs:
  - test
```

If tests fail, deployment doesn't run.

---

# 21. Add Airflow DAG deployment

Since your Airflow containers are running on Windows, you now have two reasonable deployment models.

For your existing architecture, I would deploy DAG files to a Windows folder mounted into Airflow.

For example, Airflow Docker Compose might map:

```yaml
volumes:
  - C:/DataLab/airflow/dags:/opt/airflow/dags
```

You can share that directory to Ubuntu or deploy it through a Windows-accessible location.

A cleaner method for your lab is to create a Windows SMB share.

For example:

```text
C:\DataLab\airflow\dags
```

shared as:

```text
\\192.168.250.1\airflow-dags
```

Then mount it in Ubuntu.

---

# 22. Install CIFS support

Ubuntu:

```bash
sudo apt install -y cifs-utils
```

Create:

```bash
sudo mkdir -p /mnt/airflow-dags
```

You can then mount the Windows share:

```bash
sudo mount -t cifs \
    //192.168.250.1/airflow-dags \
    /mnt/airflow-dags \
    -o username=YOUR_WINDOWS_USER
```

For automation, do not put the Windows password directly in `deploy.sh`.

Use a protected credential file or a GitHub Environment secret.

---

# 23. Deploy Airflow DAGs

Extend:

```text
scripts/deploy.sh
```

with something like:

```bash
AIRFLOW_DAG_DIR="/mnt/airflow-dags"

if [ -d dags ]; then

    echo "Deploying Airflow DAGs..."

    rsync -av \
        --delete \
        dags/ \
        "${AIRFLOW_DAG_DIR}/"

fi
```

Now:

```text
GitHub
  ↓
Ubuntu runner
  ↓
/mnt/airflow-dags
  ↓
Windows filesystem
  ↓
Airflow Docker bind mount
  ↓
/opt/airflow/dags
```

---

# 24. Deploy PostgreSQL SQL changes

Suppose your repository contains:

```text
sql/
├── 001_create_schema.sql
├── 002_create_customer.sql
└── 003_create_orders.sql
```

You could execute SQL during deployment using:

```bash
psql \
    -h 192.168.250.1 \
    -U "$POSTGRES_USER" \
    -d "$POSTGRES_DB" \
    -f sql/001_create_schema.sql
```

But don't hardcode:

```text
username
password
```

into Git.

---

# 25. Create GitHub deployment secrets

Go to:

```text
Repository
→ Settings
→ Environments
→ development
```

Create secrets such as:

```text
POSTGRES_USER

POSTGRES_PASSWORD
```

And variables such as:

```text
POSTGRES_HOST
POSTGRES_PORT
POSTGRES_DB
```

For example:

```text
POSTGRES_HOST = 192.168.250.1
POSTGRES_PORT = 5432
POSTGRES_DB   = datalab
```

Use secrets for credentials and variables for non-sensitive configuration.

---

# 26. Use secrets in the workflow

Your deployment job can contain:

```yaml
env:

  POSTGRES_HOST: ${{ vars.POSTGRES_HOST }}
  POSTGRES_PORT: ${{ vars.POSTGRES_PORT }}
  POSTGRES_DB: ${{ vars.POSTGRES_DB }}

  POSTGRES_USER: ${{ secrets.POSTGRES_USER }}
  POSTGRES_PASSWORD: ${{ secrets.POSTGRES_PASSWORD }}
```

Then the deployment script can reference:

```bash
$POSTGRES_HOST
$POSTGRES_USER
$POSTGRES_PASSWORD
```

instead of hardcoded values.

---

# 27. PostgreSQL authentication without putting password on command line

PostgreSQL clients recognize:

```text
PGPASSWORD
```

so you could temporarily export:

```bash
export PGPASSWORD="${POSTGRES_PASSWORD}"
```

and then:

```bash
psql \
    -h "${POSTGRES_HOST}" \
    -p "${POSTGRES_PORT}" \
    -U "${POSTGRES_USER}" \
    -d "${POSTGRES_DB}" \
    -f sql/deploy.sql
```

Then:

```bash
unset PGPASSWORD
```

For a lab that is acceptable. For a production system I'd move toward a proper credential or workload-identity mechanism.

---

# 28. Improved workflow with database deployment

A fuller deployment job could look like this:

```yaml
deploy:

  name: Deploy DevLab

  needs:
    - test

  runs-on:
    - self-hosted
    - Linux
    - X64
    - devlab

  environment:
    name: development
    url: http://192.168.250.1:8080

  env:

    POSTGRES_HOST: ${{ vars.POSTGRES_HOST }}
    POSTGRES_PORT: ${{ vars.POSTGRES_PORT }}
    POSTGRES_DB: ${{ vars.POSTGRES_DB }}

    POSTGRES_USER: ${{ secrets.POSTGRES_USER }}
    POSTGRES_PASSWORD: ${{ secrets.POSTGRES_PASSWORD }}

  steps:

    - name: Checkout
      uses: actions/checkout@v4

    - name: Deploy application
      run: |
        chmod +x scripts/deploy.sh
        ./scripts/deploy.sh

    - name: Test PostgreSQL connection
      env:
        PGPASSWORD: ${{ secrets.POSTGRES_PASSWORD }}
      run: |
        psql \
          -h "${POSTGRES_HOST}" \
          -p "${POSTGRES_PORT}" \
          -U "${POSTGRES_USER}" \
          -d "${POSTGRES_DB}" \
          -c "SELECT current_timestamp;"

    - name: Deploy SQL
      env:
        PGPASSWORD: ${{ secrets.POSTGRES_PASSWORD }}
      run: |
        if [ -f sql/deploy.sql ]; then

          psql \
            -h "${POSTGRES_HOST}" \
            -p "${POSTGRES_PORT}" \
            -U "${POSTGRES_USER}" \
            -d "${POSTGRES_DB}" \
            -v ON_ERROR_STOP=1 \
            -f sql/deploy.sql

        fi
```

Notice:

```text
-v ON_ERROR_STOP=1
```

This is important.

If PostgreSQL encounters an SQL error, `psql` returns failure and GitHub Actions marks the deployment step unsuccessful.

---

# 29. Add deployment health checks

You should never consider:

```text
files copied successfully
```

equivalent to:

```text
deployment succeeded
```

Add health checks.

For PostgreSQL:

```bash
pg_isready \
    -h 192.168.250.1 \
    -p 5432
```

For Airflow:

```bash
curl --fail http://192.168.250.1:8080/
```

A health-check workflow step:

```yaml
- name: Verify deployment
  run: |

    echo "Testing PostgreSQL..."

    pg_isready \
      -h "${POSTGRES_HOST}" \
      -p "${POSTGRES_PORT}"

    echo "Testing Airflow..."

    curl \
      --fail \
      --silent \
      --show-error \
      http://192.168.250.1:8080/
```

If either service fails, the workflow fails.

---

# 30. Add concurrency protection

You don't want two deployments happening simultaneously.

Add this near the top of the workflow:

```yaml
concurrency:

  group: devlab-deployment

  cancel-in-progress: false
```

That creates a deployment queue.

For production systems this matters greatly.

---

# 31. Add a timeout

Don't allow a stuck deployment to run indefinitely.

For example:

```yaml
deploy:

  timeout-minutes: 15
```

GitHub supports per-job and per-step timeouts. ([GitHub Docs][5])

---

# 32. Add manual deployment capability

Keep:

```yaml
workflow_dispatch:
```

along with:

```yaml
push:
  branches:
    - main
```

That provides two triggers:

```text
automatic:
merge/push → main

manual:
Actions → Run workflow
```

This becomes extremely useful for testing deployment logic.

---

# 33. Recommended complete workflow

For your lab, I'd ultimately use something close to:

```yaml
name: Data Platform CI/CD

on:

  push:
    branches:
      - main

  workflow_dispatch:


permissions:
  contents: read


concurrency:

  group: devlab-deployment
  cancel-in-progress: false


jobs:

  test:

    name: Build and Test

    runs-on: ubuntu-latest

    timeout-minutes: 10

    steps:

      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v6
        with:
          python-version: "3.13"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip

          if [ -f requirements.txt ]; then
            pip install -r requirements.txt
          fi

      - name: Run tests
        run: |
          if [ -d tests ]; then
            python -m pytest tests
          fi


  deploy:

    name: Deploy DevLab

    needs:
      - test

    runs-on:
      - self-hosted
      - Linux
      - X64
      - devlab

    timeout-minutes: 15

    environment:

      name: development

      url: http://192.168.250.1:8080

    env:

      POSTGRES_HOST: ${{ vars.POSTGRES_HOST }}

      POSTGRES_PORT: ${{ vars.POSTGRES_PORT }}

      POSTGRES_DB: ${{ vars.POSTGRES_DB }}

      POSTGRES_USER: ${{ secrets.POSTGRES_USER }}


    steps:

      - name: Checkout
        uses: actions/checkout@v4


      - name: Show deployment version
        run: |

          echo "Repository: ${GITHUB_REPOSITORY}"

          echo "Commit: ${GITHUB_SHA}"

          echo "Actor: ${GITHUB_ACTOR}"

          echo "Runner: $(hostname)"


      - name: Deploy files
        run: |

          chmod +x scripts/deploy.sh

          ./scripts/deploy.sh


      - name: Deploy database
        env:

          PGPASSWORD: ${{ secrets.POSTGRES_PASSWORD }}

        run: |

          if [ -f sql/deploy.sql ]; then

            psql \
              -h "${POSTGRES_HOST}" \
              -p "${POSTGRES_PORT}" \
              -U "${POSTGRES_USER}" \
              -d "${POSTGRES_DB}" \
              -v ON_ERROR_STOP=1 \
              -f sql/deploy.sql

          else

            echo "No database deployment file."

          fi


      - name: Verify PostgreSQL
        env:

          PGPASSWORD: ${{ secrets.POSTGRES_PASSWORD }}

        run: |

          psql \
            -h "${POSTGRES_HOST}" \
            -p "${POSTGRES_PORT}" \
            -U "${POSTGRES_USER}" \
            -d "${POSTGRES_DB}" \
            -c "SELECT current_timestamp;"


      - name: Verify Airflow
        run: |

          curl \
            --fail \
            --silent \
            --show-error \
            http://192.168.250.1:8080/
```

---

# 34. What happens when you update `main`

For example:

```bash
git checkout -b feature/add-customer-pipeline
```

Make changes.

Then:

```bash
git add .
git commit -m "Add customer ingestion pipeline"
git push origin feature/add-customer-pipeline
```

Open a Pull Request.

GitHub runs your required tests.

After the PR is merged:

```text
feature/add-customer-pipeline
            │
            ▼
         main
            │
            ▼
       push event
            │
            ▼
      GitHub Actions
            │
            ├── Test
            │
            ▼
          PASS
            │
            ▼
        Deployment
            │
            ▼
      Ubuntu runner
            │
      ┌─────┴─────┐
      ▼           ▼
 PostgreSQL     Airflow
```

That's a genuine CI/CD pipeline rather than simply a shell script that happens to run after a commit.

---

# 35. A more realistic future pipeline

As your lab matures, I'd evolve it into:

```text
Developer
   │
   ▼
Feature Branch
   │
   ▼
Pull Request
   │
   ├── Python lint
   ├── Unit tests
   ├── SQL validation
   ├── DAG import test
   ├── dbt compile
   └── Terraform validate
          │
          ▼
        Merge
          │
          ▼
         main
          │
          ▼
       BUILD
          │
          ├── Python package
          ├── Docker image
          └── Artifact
          │
          ▼
     DEPLOY DEV
          │
          ├── database migration
          ├── Airflow DAGs
          └── containers
          │
          ▼
       TEST
          │
          ├── health checks
          ├── smoke tests
          └── data tests
          │
          ▼
     APPROVAL GATE
          │
          ▼
       PRODUCTION
```

GitHub Environments fit naturally into the approval-gate portion because deployment jobs can be tied to a named environment and environment protection rules. ([GitHub Docs][3])

---

# 36. One important recommendation for your specific lab

I would **not** make your persistent Ubuntu development environment execute PR validation jobs.

Instead:

```text
GitHub-hosted runners
        │
        ├── lint
        ├── unit test
        ├── security checks
        └── compile/validate

Self-hosted Ubuntu runner
        │
        └── deployment only
```

That isolates the high-trust deployment host from much of the code-execution risk. GitHub specifically warns that self-hosted runners aren't ephemeral and that a compromised job can leave persistent changes or access local resources. ([GitHub Docs][2])

For your Windows/Hyper-V/Airflow/PostgreSQL learning platform, this gives you a very realistic enterprise pattern:

```text
                         GITHUB
                           │
               ┌───────────┴───────────┐
               │                       │
             CI                        CD
               │                       │
      GitHub-hosted Linux      Self-hosted Ubuntu
               │                       │
       test / validate                 │
               │                       │
               └────── PASS ───────────┘
                                       │
                                       ▼
                              Windows DevLab Network
                                       │
                           ┌───────────┴───────────┐
                           ▼                       ▼
                       PostgreSQL               Airflow
                           │                       │
                           └──────────┬────────────┘
                                      ▼
                              Data Engineering Lab
```

That is the architecture I would build before adding **dbt, Docker image builds, Terraform, Kubernetes, AWS, Azure, or GCP deployment targets**.

[1]: https://docs.github.com/en/actions/reference/runners/self-hosted-runners?utm_source=chatgpt.com "Self-hosted runners reference - GitHub Docs"
[2]: https://docs.github.com/en/actions/reference/security/secure-use?utm_source=chatgpt.com "Secure use reference - GitHub Docs"
[3]: https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments?utm_source=chatgpt.com "Deployments and environments - GitHub Docs"
[4]: https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/add-runners?utm_source=chatgpt.com "Adding self-hosted runners - GitHub Docs"
[5]: https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax?utm_source=chatgpt.com "Workflow syntax for GitHub Actions - GitHub Docs"
