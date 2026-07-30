[فارسی](README-persian.md) | [English](README.md)

# Custom Metrics: Node Exporter Textfile & JSON Exporter

Sometimes you need to monitor specific, custom data that standard exporters (like Node Exporter or cAdvisor) don't provide out-of-the-box. This could be the output of a custom script, a business metric, or a specific field from an external JSON API.

In this section, we will cover two incredibly flexible ways to export custom data into Prometheus:

1. **Node Exporter `textfile` collector**: Exporting data from custom bash scripts.
2. **JSON Exporter**: Extracting fields from any JSON API.

---

## 1. Node Exporter Textfile Collector

Node Exporter has a built-in feature called the **textfile collector**. It simply reads any file ending with `.prom` inside a specific directory and exposes its contents as Prometheus metrics.

This is extremely useful for grabbing metrics from bash scripts or cron jobs (e.g., checking if a reboot is required, counting logged-in users, or measuring the size of a specific backup folder).

### Step 1: Enable the Textfile Collector

When starting Node Exporter, you must pass the `--collector.textfile.directory` flag.

If you are using **Binary / systemd**:
Edit your `/etc/systemd/system/node_exporter.service` and modify the `ExecStart` line:

```ini
ExecStart=/usr/local/bin/node_exporter --collector.textfile.directory=/var/lib/node_exporter/textfile_collector
```

Then create the directory and restart the service:

```bash
sudo mkdir -p /var/lib/node_exporter/textfile_collector
sudo systemctl daemon-reload
sudo systemctl restart node_exporter
```

If you are using **Docker Compose**:
Update your Node Exporter service command and mount the directory:

```diff
    command:
+      - '--collector.textfile.directory=/textfile_metrics'
    volumes:
+      - '/path/to/host/metrics:/textfile_metrics:ro'
```

### Step 2: Write a Script (Example: Logged-in Users)

Let's write a simple script that counts how many users are currently logged into the server and writes it to a `.prom` file.

Create a script `/usr/local/bin/metrics_users.sh`:

```bash
#!/usr/bin/env bash

DIR="/var/lib/node_exporter/textfile_collector"
ACTIVE_USERS=$(who | wc -l)
TMP_FILE="$DIR/active_users.prom.$$"

echo "# HELP custom_active_users Number of currently logged in users." > $TMP_FILE
echo "# TYPE custom_active_users gauge" >> $TMP_FILE
echo "custom_active_users $ACTIVE_USERS" >> $TMP_FILE
mv $TMP_FILE $DIR/active_users.prom
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/metrics_users.sh
```

### Step 3: Automate with Cron

You can set up a Cron job to run this script every minute:

```bash
crontab -e
```

Add:

```bash
* * * * * /usr/local/bin/metrics_users.sh
```

Now, go to **Prometheus** (or Grafana Explore) and search for `custom_active_users`. You will see the live data!

---

## 2. JSON Exporter

What if the data you want is on the web inside a JSON API? (e.g., Bitcoin price, weather info, API status). You can use the **JSON Exporter** to fetch the URL, parse the JSON, and expose it to Prometheus.

### Step 1: Create the Configuration

Let's track the **Total Pulls of a Docker Image** (e.g., Ubuntu) using the official Docker Hub API. This is a very common DevOps scenario to monitor image usage.
Create a file named `json_exporter_config.yml`:

```yaml
modules:
  default:
    metrics:
      - name: dockerhub_pull_count
        # The JSONPath to extract the value from the API response
        path: "{$.pull_count}"
        help: "Total pull count of the Docker Hub repository"
```

### Step 2: Run JSON Exporter

You can run JSON Exporter using Docker (Recommended) or as a standalone binary.

#### Method A: System Service (Binary)

You can install the binary directly on the server. First, download the binary:

```bash
VERSION=$(curl -s https://api.github.com/repos/prometheus-community/json_exporter/releases/latest | grep '"tag_name"' | cut -d'"' -f4)
wget -O json_exporter.tar.gz https://github.com/prometheus-community/json_exporter/releases/download/${VERSION}/json_exporter-${VERSION#v}.linux-amd64.tar.gz
tar -xvf json_exporter.tar.gz
sudo mv json_exporter-*/json_exporter /usr/local/bin/
rm -rf json_exporter*
```

Create a dedicated system user for security:

```bash
sudo useradd -M -r -s /bin/false json_exporter
```

Move your config file to a system directory and set permissions:

```bash
sudo mkdir -p /etc/json_exporter
sudo cp json_exporter_config.yml /etc/json_exporter/config.yml
sudo chown -R json_exporter:json_exporter /etc/json_exporter
```

Then create a systemd service `/etc/systemd/system/json_exporter.service`:

```ini
[Unit]
Description=JSON Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=json_exporter
Group=json_exporter
ExecStart=/usr/local/bin/json_exporter --config.file=/etc/json_exporter/config.yml
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable json_exporter
sudo systemctl start json_exporter
```

#### Method B: Docker Compose (Recommended)

If you prefer using Docker, create a `docker-compose.yml`:

```yaml
services:
  json_exporter:
    image: prometheuscommunity/json-exporter:latest
    container_name: json_exporter
    restart: unless-stopped
    ports:
      - "7979:7979"
    volumes:
      - ./json_exporter_config.yml:/config.yml:ro
```

Run it: `docker compose up -d`

### Step 3: Configure Prometheus

In your `prometheus.yml`, you need to tell Prometheus to use the JSON Exporter to scrape the external API. Add this job:

```yaml
scrape_configs:
  - job_name: 'json_dockerhub'
    metrics_path: /probe
    static_configs:
      - targets:
        - https://hub.docker.com/v2/repositories/library/ubuntu/ # The API to monitor
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: {JSONـExporter_IP}:7979 # Put the address of the JSON Exporter instead of {JSONـExporter_IP}
```

Restart Prometheus.

### Step 4: View the Data!

Go to **Grafana -> Explore** or the Prometheus UI.
Search for `dockerhub_pull_count`. You will now see the total image pulls graphed natively in your monitoring stack!

> [!TIP]
> **Explore and Learn:** The flexibility of these exporters is limitless. Try looking around in Grafana Explore and experiment with building custom dashboards using these new metrics! :)
