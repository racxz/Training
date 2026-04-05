# ELK Guide


- deploy a local ELK stack with Docker,
- enable Fleet Server,
- enroll one endpoint with Elastic Agent,
- understand how Elasticsearch, Logstash, Kibana, Fleet Server, and Elastic Agent work together.



## 1. Learning Goal

By the end of this guide you will be able to understand

- what each Elastic component does,
- which ports are used,
- how containers communicate internally,
- how endpoint agents communicate with Fleet Server and Elasticsearch,
- how to bring the stack up again safely in the future.

## 2. What We Are Building

We are building a small single-node Elastic lab with these components:

- **Elasticsearch**: stores and indexes data
- **Logstash**: part of the ELK stack, available for future ingest use
- **Kibana**: web UI for search, dashboards, and Fleet management
- **Fleet Server**: central enrollment and policy server for Elastic Agents
- **Elastic Agent**: installed on Windows or Linux endpoints and managed through Fleet

## 3. High-Level Architecture

There are two important paths to understand.

### A. Management path

This is the control path used to enroll and manage endpoint agents:

`Elastic Agent -> Fleet Server:8220 -> Kibana Fleet policies -> Elasticsearch`

In practice:

- the Elastic Agent enrolls to Fleet Server on port `8220`
- Fleet Server handles check-ins and policy delivery
- Kibana is where administrators create and manage policies
- Elasticsearch stores the Fleet-related state and data

### B. Data path

This is the telemetry path:

`Elastic Agent -> Elasticsearch:9200 -> Kibana`

In practice:

- Elastic Agent checks in to Fleet Server on `8220` for management
- Elastic Agent sends collected logs, metrics, and endpoint telemetry to Elasticsearch on `9200`
- Kibana reads the indexed data from Elasticsearch

Important distinction:

- `8220` is the **control plane**
- `9200` is the **data plane**

## 4. Ports You Must Understand

- `9200`: Elasticsearch HTTP API
- `9300`: Elasticsearch transport port for internal cluster/node communication
- `5601`: Kibana web UI
- `8220`: Fleet Server enrollment and management port
- `5044`: Logstash Beats input, present in the base stack but not required for this lab
- `9600`: Logstash monitoring API

## 5. Which Container Uses Which Port

- **Elasticsearch container**
  - `9200`
  - `9300`

- **Kibana container**
  - `5601`

- **Logstash container**
  - `5044`
  - `9600`
  - optional extra inputs from the stock compose file

- **Fleet Server container**
  - `8220`

## 6. Internal Container Communication

Inside Docker Compose, containers talk to each other by service name.

Examples:

- Kibana talks to Elasticsearch at `http://elasticsearch:9200`
- Kibana knows Fleet Server as `http://fleet-server:8220`
- Fleet Server talks to Elasticsearch using internal Docker networking

This works because all services are attached to the same Docker network.

Important:

- `fleet-server` works **inside Docker only**
- Windows or Linux endpoints outside Docker usually cannot resolve `fleet-server`
- external endpoints must use the Ubuntu server IP or a real hostname for Endpoint Agent communication

## 7. Service Authentication Map

Even though the containers are on the same Docker network, they do not trust each other automatically. They still authenticate.

### Kibana -> Elasticsearch

- Username: `kibana_system`
- Password source: `.env`
- Consumed by: `kibana/config/kibana.yml`

### Logstash -> Elasticsearch

- Username: `logstash_internal`
- Password source: `.env`
- Used by the default Logstash-to-Elasticsearch output path

### Fleet Server -> Elasticsearch

- Username: `elastic`
- Password source: `.env`
- Configured in: `extensions/fleet/fleet-compose.yml`

This is acceptable for a lab, but in a more mature environment, a dedicated service token is preferred for Fleet Server.

## 8. Prerequisites

On the Ubuntu Docker host, you need:

- Docker Engine
- Docker Compose
- enough RAM for Elasticsearch
- shell access with sudo

Before bringing up the stack, set:

```bash
sudo sysctl -w vm.max_map_count=262144
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-elasticsearch.conf
sudo sysctl --system
sysctl vm.max_map_count
```

Expected result:

```text
vm.max_map_count = 262144
```

If your system already shows a higher value such as `1048576`, that is also fine.

## 9. Clone the Repository

```bash
git clone https://github.com/deviantony/docker-elk.git
cd docker-elk
find . -name "*.sh" -exec chmod +x {} +
```

## 10. Files You Need to Care About

For this teaching deployment, focus on:

- `.env`
- `kibana/config/kibana.yml`
- `extensions/fleet/fleet-compose.yml`

We are not modifying the custom Palo Alto Logstash pipeline in this guide.

## 11. Set Passwords in `.env`

Edit `.env` and set at least:

```env
ELASTIC_PASSWORD='Elastic@1234'
LOGSTASH_INTERNAL_PASSWORD='Elastic@1234'
KIBANA_SYSTEM_PASSWORD='Elastic@1234'
```

For a real environment, use unique strong passwords. For lab training, consistent values make it easier to follow the flow.

## 12. Generate Kibana Encryption Keys

Run:

```bash
docker compose up kibana-genkeys
```

Copy the generated keys into `kibana/config/kibana.yml`:

```yaml
xpack.security.encryptionKey: "<32+ char key>"
xpack.encryptedSavedObjects.encryptionKey: "<32+ char key>"
xpack.reporting.encryptionKey: "<32+ char key>"
```

These keys are important for stable saved objects and session handling in Kibana.

## 13. Fleet Is Already Preconfigured in the Repo

The repository already contains Fleet-related configuration.

### In Kibana

`kibana/config/kibana.yml` already includes:

```yaml
xpack.fleet.agents.fleet_server.hosts: [ http://fleet-server:8220 ]
```

It also predefines:

- Fleet packages
- Fleet output to Elasticsearch
- a Fleet Server agent policy

### In the Fleet extension

`extensions/fleet/fleet-compose.yml` already includes:

```yaml
FLEET_SERVER_ENABLE: '1'
FLEET_SERVER_INSECURE_HTTP: '1'
FLEET_SERVER_HOST: 0.0.0.0
FLEET_SERVER_POLICY_ID: fleet-server-policy
KIBANA_FLEET_SETUP: '1'
ELASTICSEARCH_USERNAME: elastic
ELASTICSEARCH_PASSWORD: ${ELASTIC_PASSWORD:-}
```

This means you do **not** manually install Elastic Agent inside the Fleet Server container. The Fleet Server container is already built from the Elastic Agent image.

## 14. One Small Fleet Change We Recommend

To make Fleet Server persist across reboots like the rest of the stack, use:

```yaml
restart: unless-stopped
```

This should be present in:

- `extensions/fleet/fleet-compose.yml`

## 15. First-Time Initialization

Run the setup service once:

```bash
docker compose up setup
```

Wait for it to complete successfully.

This step initializes Elasticsearch users and roles based on `.env`.

## 16. Start the Base ELK Stack

```bash
docker compose up -d --build
docker compose ps
```

Validate:

```bash
curl -u elastic:Elastic@1234 http://127.0.0.1:9200
docker compose logs elasticsearch --tail=100
docker compose logs kibana --tail=100
```

## 17. Start Fleet Server

Run:

```bash
docker compose -f docker-compose.yml -f extensions/fleet/fleet-compose.yml up -d
```

This adds Fleet Server to the running ELK stack. It does not replace the other containers.

After this, your steady-state lab should have four persistent services:

- Elasticsearch
- Logstash
- Kibana
- Fleet Server

Validate Fleet:

```bash
docker compose -f docker-compose.yml -f extensions/fleet/fleet-compose.yml ps
docker compose -f docker-compose.yml -f extensions/fleet/fleet-compose.yml logs fleet-server --tail=100
```

## 18. Open the Fleet Port

If the Ubuntu host firewall is enabled:

```bash
sudo ufw allow 8220/tcp
sudo ufw reload
```

## 19. Verify Fleet in Kibana

Open Kibana:

```text
http://<ubuntu_server_ip>:5601
```

Log in as `elastic`.

Then:

1. Open **Fleet**
2. Confirm **Fleet Server** shows as healthy
3. Create or review an Agent Policy
4. Generate an enrollment token or enrollment command

## 20. Endpoint Enrollment Rule You Must Remember

This value:

```text
http://fleet-server:8220
```

works only inside Docker.

When enrolling a real endpoint, replace `fleet-server` with:

- the Ubuntu server IP, or
- a hostname that the endpoint can resolve

Example:

```text
http://192.168.0.189:8220
```

## 21. Why `--insecure` Is Required in This Lab

The Fleet extension in this lab uses insecure HTTP rather than HTTPS.

So Elastic Agent will reject enrollment unless you add:

```text
--insecure
```

This is acceptable for lab training only.

## 22. Validated Windows Endpoint Enrollment

Run PowerShell as Administrator.

Download and extract the agent:

```powershell
$ProgressPreference = 'SilentlyContinue'
Invoke-WebRequest -Uri https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-9.3.2-windows-x86_64.zip -OutFile elastic-agent-9.3.2-windows-x86_64.zip
Expand-Archive .\elastic-agent-9.3.2-windows-x86_64.zip -DestinationPath .
cd .\elastic-agent-9.3.2-windows-x86_64
```

Install and enroll:

```powershell
.\elastic-agent.exe install --url=http://192.168.0.189:8220 --enrollment-token=<token> --insecure
```

Expected result:

- Elastic Agent installs under `C:\Program Files\Elastic\Agent`
- it registers as a Windows service
- the endpoint appears in Kibana Fleet as healthy

## 23. Validated Linux Enrollment Pattern

On Linux, the same rule applies:

- use the Ubuntu server IP or hostname
- include `--insecure`

Example:

```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-9.3.2-linux-x86_64.tar.gz
tar xzvf elastic-agent-9.3.2-linux-x86_64.tar.gz
cd elastic-agent-9.3.2-linux-x86_64
sudo ./elastic-agent install --url=http://192.168.0.189:8220 --enrollment-token=<token> --insecure
```

## 24. What Happens When Enrollment Works

When the endpoint enrolls successfully:

1. Elastic Agent checks in to Fleet Server on `8220`
2. Fleet Server provides the assigned policy
3. Elastic Agent starts the integrations defined by that policy
4. Elastic Agent sends collected data to Elasticsearch on `9200`
5. Kibana shows the enrolled agent in Fleet and the resulting telemetry in Discover and dashboards

## 25. Common Mistakes

### Mistake 1: Using `fleet-server` from Windows or Linux

Problem:

- external endpoints cannot resolve the Docker service name

Fix:

- use the Ubuntu server IP or real hostname instead

### Mistake 2: Forgetting `--insecure`

Problem:

- enrollment fails because Fleet Server is using HTTP in the lab

Fix:

- add `--insecure` to the install command

### Mistake 3: Running `docker compose logs` outside the repo directory

Problem:

- Docker Compose cannot find `docker-compose.yml`

Fix:

```bash
cd /path/to/docker-elk
docker compose logs kibana --tail=50
```

### Mistake 4: Elasticsearch not starting cleanly

Problem:

- `vm.max_map_count` was not set

Fix:

- set it before starting the stack

## 26. How to Explain the Whole Setup to Someone Else

A clean teaching explanation is:

“We deployed Elasticsearch, Logstash, Kibana, and Fleet Server in Docker. Elasticsearch stores the data, Kibana is the management and search UI, Logstash is available as the ingest layer, and Fleet Server is the control plane for endpoint agents. Endpoint Elastic Agents enroll to Fleet Server on port 8220 for policy management, but they send their actual logs and telemetry to Elasticsearch on port 9200. Kibana reads that indexed data from Elasticsearch and also lets us manage the agents through Fleet.”

If you want the English-only version:

“We deployed Elasticsearch, Logstash, Kibana, and Fleet Server in Docker. Elasticsearch stores the data, Kibana is the search and management UI, Logstash is available as an ingest layer, and Fleet Server is the control plane for endpoint agents. Elastic Agents enroll to Fleet Server on port 8220 for management and policy updates, while their telemetry is sent to Elasticsearch on port 9200. Kibana reads that data from Elasticsearch and also manages the agents through Fleet.”

## 27. Maintenance Basics

Keep Elasticsearch, Kibana, Logstash, Fleet Server, and Elastic Agent on compatible versions.

For stack upgrades:

1. update `ELASTIC_VERSION` in `.env`
2. rebuild images
3. restart containers
4. validate Elasticsearch, Kibana, Fleet Server, and agent enrollment

Typical commands:

```bash
docker compose build --no-cache
docker compose up -d
docker compose -f docker-compose.yml -f extensions/fleet/fleet-compose.yml up -d
docker compose ps
```

For endpoint agent upgrades:

- test on one endpoint first
- confirm telemetry still arrives
- then roll out more widely

## 28. Final Teaching Summary

If I were teaching a junior engineer, I would make them remember these five facts:

1. Elasticsearch stores the data on `9200`
2. Kibana is the web UI on `5601`
3. Fleet Server manages endpoint agents on `8220`
4. `fleet-server` is only an internal Docker name, not an endpoint URL
5. In this lab, endpoint agents need `--insecure` because Fleet uses HTTP instead of HTTPS

That is enough to understand the system before moving on to more advanced topics such as custom Logstash pipelines, firewall log ingestion, TLS hardening, and production-grade agent management.
