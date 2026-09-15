Yes. For your Windows 11 Pro + Hyper-V + Ubuntu 26.04 VM, I would use a **dual-network design**:

```text
                         WINDOWS 11 PRO
                    64 GB physical memory
                             │
                ┌────────────┴────────────┐
                │                         │
           Hyper-V VM               Docker Desktop
                │                         │
        Ubuntu 26.04                  Linux Containers
        8 CPU / 16 GB                │
                │                    ├── PostgreSQL
                │                    │      :5432
                │                    │
                │                    └── Airflow
                │                           :8080
                │
       ┌────────┴─────────┐
       │                  │
Default Switch       DevLab Switch
Internet             192.168.250.0/24
DHCP/NAT                  │
                         │
Ubuntu               Windows Host
192.168.250.10  ↔    192.168.250.1
```

The **Default Switch** remains Ubuntu's Internet route. The new **DevLab Internal switch** provides a permanent, predictable connection between Ubuntu and your Windows-hosted Docker containers.

Docker Desktop publishes container ports onto Windows; Docker documents that published ports are exposed through the Docker Desktop backend, and you can bind them to a specific host IP instead of every interface. ([Docker Documentation][1])

---

# 1. Overall target

We'll build:

| Service                 | Runs on                   | Address from Ubuntu         |
| ----------------------- | ------------------------- | --------------------------- |
| PostgreSQL              | Docker Desktop on Windows | `192.168.250.1:5432`        |
| Airflow UI/API          | Docker Desktop on Windows | `http://192.168.250.1:8080` |
| Ubuntu Internet         | Hyper-V Default Switch    | DHCP/NAT                    |
| Host/VM private network | Hyper-V DevLab switch     | `192.168.250.0/24`          |

I recommend keeping your **development PostgreSQL database separate from Airflow's internal metadata PostgreSQL database**. Airflow's official Compose environment already includes its own PostgreSQL service. ([Apache Airflow][2])

That gives you:

```text
Docker Desktop
│
├── PostgreSQL-Lab
│      database for your ETL/ELT work
│
└── Airflow
       ├── API/UI
       ├── Scheduler
       ├── DAG processor
       ├── Worker
       ├── Triggerer
       └── Airflow metadata PostgreSQL
```

This is much cleaner than using Airflow's metadata database for your own tables.

---

# Part I — Create a permanent Windows ↔ Ubuntu network

## 2. Shut down Ubuntu

Inside Ubuntu:

```bash
sudo poweroff
```

Do not merely suspend the VM.

---

# 3. Create a Hyper-V Internal switch

On Windows open **PowerShell as Administrator**.

Run:

```powershell
New-VMSwitch -Name "DevLab" -SwitchType Internal
```

Verify:

```powershell
Get-VMSwitch
```

You should see something like:

```text
Name            SwitchType
----            ----------
Default Switch  Internal
DevLab          Internal
```

An Internal Hyper-V switch allows communication between the Windows host and attached VMs. Hyper-V's virtual-switch architecture is specifically intended to provide this VM/host/network connectivity. ([Microsoft Learn][3])

---

# 4. Find the new Windows adapter

Run:

```powershell
Get-NetAdapter
```

You should find:

```text
vEthernet (DevLab)
```

For example:

```text
Name                  Status
----                  ------
Ethernet              Up
vEthernet (Default Switch) Up
vEthernet (DevLab)    Up
```

---

# 5. Assign Windows 192.168.250.1

Run:

```powershell
New-NetIPAddress `
    -InterfaceAlias "vEthernet (DevLab)" `
    -IPAddress 192.168.250.1 `
    -PrefixLength 24
```

Verify:

```powershell
Get-NetIPAddress -InterfaceAlias "vEthernet (DevLab)" -AddressFamily IPv4
```

You want:

```text
IPAddress       : 192.168.250.1
PrefixLength    : 24
```

The network is therefore:

```text
Network:       192.168.250.0
Windows:       192.168.250.1
Ubuntu:        192.168.250.10
Broadcast:     192.168.250.255
Mask:          255.255.255.0
```

---

# 6. Add a second network adapter to Ubuntu

Open:

```text
Hyper-V Manager
```

Select:

```text
Ubuntu-26.04-DE
→ Settings
```

Your existing network adapter should remain connected to:

```text
Default Switch
```

Do not change it.

Now select:

```text
Add Hardware
→ Network Adapter
→ Add
```

Set the second adapter to:

```text
Virtual switch: DevLab
```

You now have:

```text
Ubuntu VM
│
├── NIC #1 → Default Switch
│           Internet
│
└── NIC #2 → DevLab
            Windows/Docker communication
```

Start Ubuntu.

---

# Part II — Configure Ubuntu's private address

## 7. Identify the two interfaces

Inside Ubuntu run:

```bash
ip addr
```

You'll probably see names similar to:

```text
lo
eth0
eth1
```

or:

```text
ens160
ens192
```

Use:

```bash
ip -br addr
```

This is easier to read.

For example:

```text
lo        UNKNOWN  127.0.0.1/8
eth0      UP       172.23.144.25/20
eth1      UP
```

The interface with an automatically assigned address is probably your **Default Switch** interface.

The new unconfigured interface is your **DevLab** interface.

Assume for the following example that it is:

```text
eth1
```

Substitute your actual interface name.

---

# 8. Configure DevLab with NetworkManager

Ubuntu Desktop uses NetworkManager.

First check:

```bash
nmcli device status
```

Example:

```text
DEVICE  TYPE      STATE
eth0    ethernet  connected
eth1    ethernet  connected
```

List connections:

```bash
nmcli connection show
```

Find the connection associated with your DevLab NIC.

You can create a clearly named connection:

```bash
sudo nmcli connection add \
    type ethernet \
    ifname eth1 \
    con-name DevLab \
    ipv4.method manual \
    ipv4.addresses 192.168.250.10/24
```

Very important: **do not set a gateway on this interface.**

Your Default Switch remains your Internet gateway.

Disable IPv6 on this particular connection if you want to simplify the lab:

```bash
sudo nmcli connection modify DevLab ipv6.method disabled
```

Bring it up:

```bash
sudo nmcli connection up DevLab
```

---

# 9. Verify the Ubuntu network

Run:

```bash
ip -br addr
```

You should now see something similar to:

```text
eth0  UP  172.x.x.x/...
eth1  UP  192.168.250.10/24
```

Check routing:

```bash
ip route
```

You should have something conceptually like:

```text
default via 172.x.x.1 dev eth0

172.x.x.x/... dev eth0

192.168.250.0/24 dev eth1
    src 192.168.250.10
```

There should **not** be:

```text
default via 192.168.250.1
```

The Default Switch remains responsible for Internet access.

---

# 10. Test Windows ↔ Ubuntu

From Ubuntu:

```bash
ping -c 4 192.168.250.1
```

You should get replies.

From Windows PowerShell:

```powershell
ping 192.168.250.10
```

If Windows doesn't answer ICMP, that doesn't necessarily mean TCP connectivity is broken because Windows Firewall may block ping.

We'll test TCP later.

---

# Part III — Install Docker Desktop on Windows

## 11. Install Docker Desktop

Install **Docker Desktop for Windows** and run **Linux containers**.

Docker Desktop currently supports several Linux-container backends on Windows, including WSL 2 and Hyper-V; WSL 2 remains the normal default and works fine even though your separate Ubuntu development machine runs under Hyper-V. ([Docker Documentation][4])

You don't need Docker installed inside Ubuntu for this architecture.

Conceptually:

```text
Windows
│
├── Hyper-V
│    └── Ubuntu VM
│
└── Docker Desktop
     └── Linux container VM/backend
```

They are separate environments.

---

# 12. Verify Docker Desktop

Open PowerShell:

```powershell
docker version
```

Then:

```powershell
docker compose version
```

You should get versions for both.

Test Docker:

```powershell
docker run --rm hello-world
```

---

# Part IV — Build the PostgreSQL lab container

## 13. Create a project directory

From PowerShell:

```powershell
mkdir C:\DataLab
mkdir C:\DataLab\postgres
cd C:\DataLab\postgres
```

---

# 14. Create PostgreSQL Compose file

Create:

```text
C:\DataLab\postgres\compose.yaml
```

with:

```yaml
services:

  postgres-lab:
    image: postgres:17
    container_name: postgres-lab

    restart: unless-stopped

    environment:
      POSTGRES_USER: labuser
      POSTGRES_PASSWORD: labpassword
      POSTGRES_DB: datalab

    ports:
      - "192.168.250.1:5432:5432"

    volumes:
      - postgres_lab_data:/var/lib/postgresql/data

    healthcheck:
      test:
        [
          "CMD-SHELL",
          "pg_isready -U labuser -d datalab"
        ]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_lab_data:
```

The important line is:

```yaml
ports:
  - "192.168.250.1:5432:5432"
```

It means:

```text
Windows
192.168.250.1:5432
        │
        ↓
Docker
        │
        ↓
PostgreSQL container
5432
```

Docker supports the full:

```text
HOST_IP:HOST_PORT:CONTAINER_PORT
```

syntax specifically for this purpose. ([Docker Documentation][5])

This is preferable to:

```yaml
- "5432:5432"
```

because that would normally bind PostgreSQL to all available Windows interfaces.

---

# 15. Start PostgreSQL

From:

```powershell
cd C:\DataLab\postgres
```

run:

```powershell
docker compose up -d
```

Check:

```powershell
docker compose ps
```

You should see:

```text
postgres-lab   Up (healthy)
```

---

# 16. Examine PostgreSQL logs

Run:

```powershell
docker logs postgres-lab
```

Eventually you want to see that PostgreSQL is ready to accept connections.

---

# 17. Test PostgreSQL inside the container

Run:

```powershell
docker exec -it postgres-lab psql -U labuser -d datalab
```

Then:

```sql
SELECT version();
```

and:

```sql
SELECT current_database();
```

Exit:

```text
\q
```

---

# Part V — Allow Ubuntu through Windows Firewall

## 18. Create a restricted PostgreSQL firewall rule

Open **PowerShell as Administrator**.

Run:

```powershell
New-NetFirewallRule `
    -DisplayName "DevLab PostgreSQL" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalAddress 192.168.250.1 `
    -LocalPort 5432 `
    -RemoteAddress 192.168.250.0/24 `
    -Action Allow
```

This says:

```text
ALLOW:
192.168.250.0/24
        ↓
192.168.250.1:5432

DO NOT simply expose PostgreSQL everywhere.
```

---

# Part VI — Connect Ubuntu to PostgreSQL

## 19. Install PostgreSQL client

Inside Ubuntu:

```bash
sudo apt update
```

Then:

```bash
sudo apt install -y postgresql-client
```

Also install useful networking tools:

```bash
sudo apt install -y \
    curl \
    netcat-openbsd \
    dnsutils
```

---

# 20. Test the TCP port

From Ubuntu:

```bash
nc -vz 192.168.250.1 5432
```

You want something like:

```text
Connection to 192.168.250.1 5432 port [tcp/postgresql] succeeded!
```

---

# 21. Connect with psql

Run:

```bash
psql \
    -h 192.168.250.1 \
    -p 5432 \
    -U labuser \
    -d datalab
```

Enter:

```text
labpassword
```

when prompted.

You should get:

```text
datalab=#
```

Test:

```sql
SELECT version();
```

Then:

```sql
CREATE TABLE connection_test
(
    id          BIGSERIAL PRIMARY KEY,
    source      VARCHAR(100),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Insert:

```sql
INSERT INTO connection_test(source)
VALUES ('Ubuntu 26.04 Hyper-V VM');
```

Retrieve it:

```sql
SELECT *
FROM connection_test;
```

You should get something similar to:

```text
 id | source                     | created_at
----+----------------------------+---------------------
 1  | Ubuntu 26.04 Hyper-V VM    | ...
```

Now you've proven the complete path:

```text
Ubuntu
192.168.250.10
        │
        │ TCP 5432
        ▼
Windows
192.168.250.1
        │
        ▼
Docker Desktop
        │
        ▼
postgres-lab
        │
        ▼
datalab database
```

---

# Part VII — Install Airflow with Docker Compose

## 22. Create the Airflow directory

On Windows PowerShell:

```powershell
mkdir C:\DataLab\airflow
cd C:\DataLab\airflow
```

Apache's official documentation recommends using its supplied Docker Compose environment for local learning/development. The current Airflow 3 Compose stack includes the scheduler, DAG processor, API server, worker, triggerer, initialization service and PostgreSQL metadata database. ([Apache Airflow][2])

---

# 23. Download the official Airflow Compose file

Use:

```powershell
curl.exe -LfO https://airflow.apache.org/docs/apache-airflow/stable/docker-compose.yaml
```

You should now have:

```text
C:\DataLab\airflow\docker-compose.yaml
```

Apache explicitly publishes this Compose file for the Airflow Docker quick start. ([Apache Airflow][6])

---

# 24. Create Airflow directories

PowerShell:

```powershell
mkdir dags
mkdir logs
mkdir plugins
mkdir config
```

You should now have:

```text
C:\DataLab\airflow\
│
├── docker-compose.yaml
├── dags\
├── logs\
├── plugins\
└── config\
```

---

# 25. Create the `.env` file

Because the Docker host is Windows, use the standard Airflow container UID.

PowerShell:

```powershell
"AIRFLOW_UID=50000" | Out-File -Encoding ascii .env
```

Check:

```powershell
Get-Content .env
```

Expected:

```text
AIRFLOW_UID=50000
```

Apache specifically documents `50000` as the appropriate value when using the Compose quick start on non-Linux hosts. ([Apache Airflow][2])

---

# 26. Change Airflow's published address

Open:

```text
docker-compose.yaml
```

Find the Airflow API-server service.

Depending on the exact current file, it will be named similar to:

```yaml
airflow-apiserver:
```

or:

```yaml
airflow-api-server:
```

Find its port declaration:

```yaml
ports:
  - "8080:8080"
```

Change it to:

```yaml
ports:
  - "192.168.250.1:8080:8080"
```

If the file uses an environment substitution such as:

```yaml
- "${AIRFLOW_APISERVER_PORT:-8080}:8080"
```

replace that published-port entry with:

```yaml
- "192.168.250.1:8080:8080"
```

The result is:

```text
Ubuntu
        │
        ▼
192.168.250.1:8080
        │
        ▼
Docker Desktop
        │
        ▼
Airflow API/UI :8080
```

Airflow 3 uses an API-server-centric architecture, and its standard Compose deployment makes the UI/API available on port `8080`. ([Apache Airflow][2])

---

# 27. Initialize Airflow

From:

```powershell
C:\DataLab\airflow
```

run:

```powershell
docker compose up airflow-init
```

Allow the initialization container to finish.

Apache's quick start specifically requires this initialization step before starting the remaining Airflow services. ([Apache Airflow][2])

---

# 28. Start Airflow

Run:

```powershell
docker compose up -d
```

Then:

```powershell
docker compose ps
```

You should eventually see Airflow components running.

Depending on the current Airflow Compose release, you'll see services such as:

```text
airflow-apiserver
airflow-scheduler
airflow-dag-processor
airflow-worker
airflow-triggerer
postgres
```

---

# 29. Watch startup

Use:

```powershell
docker compose logs -f
```

Press:

```text
Ctrl+C
```

to stop following logs. It does not stop the containers.

---

# 30. Verify Airflow from Windows

Open:

```text
http://192.168.250.1:8080
```

The official quick start normally creates:

```text
Username: airflow
Password: airflow
```

for development use. ([Apache Airflow][6])

Change credentials before using the environment for anything beyond a local learning lab.

---

# 31. Add Windows Firewall rule for Airflow

From elevated PowerShell:

```powershell
New-NetFirewallRule `
    -DisplayName "DevLab Airflow" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalAddress 192.168.250.1 `
    -LocalPort 8080 `
    -RemoteAddress 192.168.250.0/24 `
    -Action Allow
```

---

# 32. Test Airflow from Ubuntu

From Ubuntu:

```bash
nc -vz 192.168.250.1 8080
```

Expected:

```text
Connection to 192.168.250.1 8080 port [tcp/http-alt] succeeded!
```

Then:

```bash
curl -I http://192.168.250.1:8080
```

You should receive an HTTP response.

---

# 33. Open the Airflow UI from Ubuntu

Ubuntu Firefox:

```text
http://192.168.250.1:8080
```

You should reach the Airflow UI.

This proves:

```text
Ubuntu VM
   │
   │ HTTP :8080
   ▼
Windows DevLab interface
   │
   ▼
Docker Desktop
   │
   ▼
Airflow container
```

---

# Part VIII — Let Airflow use your lab PostgreSQL

There are now **two different connection paths**, and the distinction matters.

From Ubuntu:

```text
PostgreSQL host:
192.168.250.1
```

But from inside an Airflow container, you should normally use:

```text
host.docker.internal
```

Docker Desktop provides `host.docker.internal` specifically so containers can reach services published on the Windows host. ([Docker Documentation][7])

So:

```text
Ubuntu VM
    │
    └── PostgreSQL:
        192.168.250.1:5432


Airflow Container
    │
    └── PostgreSQL:
        host.docker.internal:5432
```

---

# 34. Create PostgreSQL connection in Airflow

Open Airflow:

```text
http://192.168.250.1:8080
```

Go to the connection-management screen and create:

```text
Connection ID:
datalab_postgres

Connection Type:
Postgres

Host:
host.docker.internal

Database:
datalab

Login:
labuser

Password:
labpassword

Port:
5432
```

Airflow's PostgreSQL provider uses the standard host, database, login, password and port parameters for PostgreSQL connections. ([Apache Airflow][8])

---

# 35. Test the full architecture

Your completed traffic paths are now:

```text
                       WINDOWS 11 PRO

        ┌──────────────────────────────────────┐
        │                                      │
        │ Docker Desktop                       │
        │                                      │
        │  ┌──────────────────────────────┐    │
        │  │ PostgreSQL Lab               │    │
        │  │                              │    │
        │  │ datalab                      │    │
        │  │ :5432                        │    │
        │  └──────────────▲───────────────┘    │
        │                 │                    │
        │                 │                    │
        │  ┌──────────────┴───────────────┐    │
        │  │ Airflow                      │    │
        │  │                              │    │
        │  │ API/UI        :8080          │    │
        │  │ Scheduler                    │    │
        │  │ Worker                       │    │
        │  │ DAG Processor                │    │
        │  │ Metadata PostgreSQL          │    │
        │  └──────────────────────────────┘    │
        │                                      │
        └───────────────▲──────────────────────┘
                        │
              192.168.250.1
                        │
                  DevLab vSwitch
                        │
              192.168.250.10
                        │
        ┌───────────────┴──────────────────────┐
        │ Ubuntu 26.04                         │
        │                                      │
        │ Python                               │
        │ dbt                                  │
        │ psql                                 │
        │ VS Code                              │
        │ Git                                  │
        │                                      │
        │ PostgreSQL → 192.168.250.1:5432      │
        │ Airflow    → 192.168.250.1:8080      │
        └──────────────────────────────────────┘
```

---

# 36. Useful verification commands

From **Windows**:

```powershell
docker ps
```

Check PostgreSQL:

```powershell
Test-NetConnection 192.168.250.1 -Port 5432
```

Check Airflow:

```powershell
Test-NetConnection 192.168.250.1 -Port 8080
```

From **Ubuntu**:

```bash
ping -c 2 192.168.250.1
```

PostgreSQL:

```bash
nc -vz 192.168.250.1 5432
```

Airflow:

```bash
nc -vz 192.168.250.1 8080
```

Database login:

```bash
psql -h 192.168.250.1 -U labuser -d datalab
```

Airflow HTTP:

```bash
curl http://192.168.250.1:8080
```

Internet:

```bash
curl https://example.com
```

If all five work, your network design is correct.

---

# 37. Starting and stopping the lab

### PostgreSQL

Start:

```powershell
cd C:\DataLab\postgres
docker compose up -d
```

Stop without deleting data:

```powershell
docker compose down
```

PostgreSQL data remains in:

```text
postgres_lab_data
```

because it's a Docker volume.

Do **not** casually use:

```powershell
docker compose down -v
```

because `-v` deletes the volume and therefore your database.

### Airflow

Start:

```powershell
cd C:\DataLab\airflow
docker compose up -d
```

Stop:

```powershell
docker compose down
```

Status:

```powershell
docker compose ps
```

Logs:

```powershell
docker compose logs -f
```

---

# 38. Recommended next architecture

This arrangement fits particularly well with the data-engineering lab you're building:

```text
Ubuntu Development VM
│
├── Git
├── Python 3
├── Rust
├── dbt
├── PySpark
├── Terraform
├── VS Code / Remote Development
│
│
│ SQL
│
├──────────────────────→ PostgreSQL container
│
│ DAG development
│
└──────────────────────→ Airflow
                           │
                           ├── Extract
                           ├── Load
                           ├── Transform
                           ├── dbt
                           └── PostgreSQL
```

Then you can progressively add:

```text
Phase 1
PostgreSQL
   ↓
Python
   ↓
Airflow


Phase 2
PostgreSQL
   ↓
Python
   ↓
Airflow
   ↓
dbt


Phase 3
Sources
   ↓
Airflow
   ↓
Python / PySpark
   ↓
PostgreSQL
   ↓
dbt


Phase 4
Docker
   ↓
Airflow
   ↓
Kubernetes
   ↓
Terraform


Phase 5
Local Lab
   ↓
   ├── AWS
   ├── Azure
   └── GCP
```

The especially valuable feature of this design is that **Ubuntu behaves like a separate Linux development workstation accessing externally hosted services** rather than having everything installed on one machine. That is much closer to how a real enterprise data-engineering environment behaves.

[1]: https://docs.docker.com/desktop/features/networking/?utm_source=chatgpt.com "Networking on Docker Desktop | Docker Docs"
[2]: https://airflow.apache.org/docs/apache-airflow/3.0.4/howto/docker-compose/?utm_source=chatgpt.com "Running Airflow in Docker — Airflow 3.0.4 Documentation"
[3]: https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/virtual-switch?utm_source=chatgpt.com "Hyper-V Virtual Switch | Microsoft Learn"
[4]: https://docs.docker.com/desktop/setup/install/windows-install/?utm_source=chatgpt.com "Install Docker Desktop on Windows | Docker Docs"
[5]: https://docs.docker.com/get-started/docker-concepts/running-containers/publishing-ports/?utm_source=chatgpt.com "Publishing and exposing ports | Docker Docs"
[6]: https://airflow.apache.org/docs/apache-airflow/3.0.3/tutorial/pipeline.html?utm_source=chatgpt.com "Building a Simple Data Pipeline — Airflow 3.0.3 Documentation"
[7]: https://docs.docker.com/desktop/features/networking/networking-how-tos/?utm_source=chatgpt.com "Explore networking how-tos on Docker Desktop | Docker Docs"
[8]: https://airflow.apache.org/docs/apache-airflow-providers-postgres/stable/connections/postgres.html?utm_source=chatgpt.com "PostgreSQL Connection — apache-airflow-providers-postgres Documentation"
