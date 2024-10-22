To add an **Nginx web server** using Docker and monitor it using **Prometheus** and **Grafana**, you need to run the Nginx container and use **Prometheus Node Exporter** to collect system-level metrics. You'll also configure **Nginx Exporter** to scrape Nginx-specific metrics, and finally, visualize everything in Grafana.

Here’s a step-by-step guide:

---

### Steps:

### 1. **Run Nginx Web Server in a Docker Container**

Run an Nginx container using Docker:

```bash
docker run -d \
  --name nginx \
  -p 80:80 \
  nginx
```

This command pulls the **nginx** image and exposes it on port **80** of your Docker host. Verify by accessing the Nginx web server in your browser:

```
http://<your-server-ip>
```

---

### 2. **Run Prometheus Container**

Now, configure and run **Prometheus** to monitor Nginx metrics.

#### a. Create Prometheus Configuration File (`prometheus.yml`):

Create a directory to store the configuration:

```bash
mkdir -p ~/prometheus
cd ~/prometheus
nano prometheus.yml
```

In this file, configure Prometheus to scrape both **Node Exporter** and **Nginx Exporter** metrics:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'nginx_exporter'
    static_configs:
      - targets: ['localhost:9113']
```

#### b. Run Prometheus Container:

Now run the Prometheus container with the custom configuration file:

```bash
docker run -d \
  -p 9090:9090 \
  -v ~/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
  --name prometheus \
  prom/prometheus
```

This exposes Prometheus on port **9090**.

#### c. Verify Prometheus:

Check if Prometheus is running by accessing:

```
http://<your-server-ip>:9090
```

---

### 3. **Run Node Exporter to Collect System Metrics**

Prometheus **Node Exporter** collects system-level metrics (CPU, memory, etc.). To monitor the host where Nginx is running, launch **Node Exporter** in a container:

```bash
docker run -d \
  -p 9100:9100 \
  --name node-exporter \
  prom/node-exporter
```

- This exposes Node Exporter on port **9100**.

---

### 4. **Run Nginx Exporter to Scrape Nginx Metrics**

Nginx doesn’t expose metrics natively, so we’ll use **Nginx Exporter** to gather metrics from the Nginx container.

#### a. Install Nginx Exporter:

Run the **Nginx Exporter** container, which scrapes metrics from Nginx’s status page:

```bash
docker run -d \
  --name nginx-exporter \
  -p 9113:9113 \
  nginx/nginx-prometheus-exporter:latest \
  -nginx.scrape-uri=http://<your-server-ip>:80/nginx_status
```

- Ensure that Nginx is configured to expose the status page at `http://localhost/nginx_status`.

#### b. Configure Nginx to Expose the Status Page:

If your Nginx container doesn't have the status page configured, modify its default configuration.

1. **Edit Nginx Configuration**: 
   You need to access the Nginx container and modify its configuration to enable the **stub_status** module.

2. **Steps**:
   
   - Run a bash shell inside the Nginx container:

     ```bash
     docker exec -it nginx bash
     ```

   - Edit the Nginx default config file (`/etc/nginx/nginx.conf` or `/etc/nginx/conf.d/default.conf`) to include the following:

     ```nginx
     server {
       listen 80;

       location /nginx_status {
         stub_status on;
         access_log off;
         allow 127.0.0.1;
         allow <your-server-ip>;
         deny all;
       }
     }
     ```

   - Exit the container and reload Nginx:

     ```bash
     docker exec nginx nginx -s reload
     ```

Now the **Nginx status page** should be available at:

```
http://<your-server-ip>:80/nginx_status
```

---

### 5. **Run Grafana Container**

Now, run the Grafana container to visualize the metrics from Prometheus:

```bash
docker run -d \
  -p 3000:3000 \
  --name grafana \
  grafana/grafana
```

- Grafana will be available at `http://<your-server-ip>:3000` with default credentials **admin/admin**.

---

### 6. **Add Prometheus as a Data Source in Grafana**

Once Grafana is up and running:

1. Log in to **Grafana** (http://<your-server-ip>:3000) with the default credentials **admin/admin**.
2. Go to **Configuration** → **Data Sources** → **Add Data Source**.
3. Select **Prometheus** from the list.
4. Set the URL to **http://<your-server-ip>:9090**.
5. Click **Save & Test** to verify the connection.

---

### 7. **Create Grafana Dashboards for Nginx and System Monitoring**

Once Prometheus is added as a data source, you can create dashboards to visualize Nginx and host metrics:

#### a. Nginx Metrics:
1. Go to **Create** → **Dashboard** → **Add New Panel**.
2. Use **Nginx Exporter** metrics like:

   - Requests: `nginx_http_requests_total`
   - Active Connections: `nginx_connections_active`
   - Accepted Connections: `nginx_connections_accepted`
   - Handled Connections: `nginx_connections_handled`

#### b. System Metrics (Node Exporter):
1. Similarly, create panels for system metrics:
   
   - CPU Usage: `rate(node_cpu_seconds_total[5m])`
   - Memory Usage: `node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes`
   - Disk Usage: `node_filesystem_avail_bytes / node_filesystem_size_bytes`

---

### 8. **Optional: Make Data Persistent (Volume Mounting)**

To make sure your configuration and data persist even after container restarts, you can mount Docker volumes for **Prometheus** and **Grafana**.

#### a. Prometheus Data Persistence:

```bash
docker run -d \
  -p 9090:9090 \
  -v ~/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v ~/prometheus_data:/prometheus \
  --name prometheus \
  prom/prometheus
```

#### b. Grafana Data Persistence:

```bash
docker run -d \
  -p 3000:3000 \
  -v ~/grafana_data:/var/lib/grafana \
  --name grafana \
  grafana/grafana
```

---

### 9. **Verify the Setup**

- **Nginx** should be running on port **80**.
- **Prometheus** should be running on port **9090** and scraping Nginx metrics through the Nginx exporter on **9113**.
- **Grafana** should be running on port **3000**, connected to Prometheus, and displaying metrics.

### 10. **Security Considerations**

- Use **firewalls/security groups** to ensure that ports **80 (Nginx)**, **9090 (Prometheus)**, **9113 (Nginx Exporter)**, and **3000 (Grafana)** are open only to trusted IP addresses.
- Use **SSL/TLS** to secure access to Grafana and Nginx.

---


