# Linux Mint Portable Server Setup

Complete installation and configuration block for **Apache**, **PostgreSQL**, **Python CGI (`psycopg2`)**, **`ttyd`**, **Docker**, **`kubectl`**, **`k3d`**, **Prometheus**, **Node Exporter**, and **Grafana**.

```bash
# 1. Install system packages (Apache, PostgreSQL, Python, ttyd, Docker)
sudo apt update
sudo apt install -y apache2 postgresql postgresql-contrib python3 python3-psycopg2 ttyd docker.io curl

# 2. Enable native systemd services
sudo systemctl enable --now apache2 postgresql docker
sudo usermod -aG docker "$USER"

# 3. Configure PostgreSQL user and database
sudo -u postgres psql -c "DO \$\$ BEGIN IF NOT EXISTS (SELECT FROM pg_catalog.pg_roles WHERE rolname = 'chanithu') THEN CREATE USER chanithu WITH PASSWORD 'StrongPass123' SUPERUSER; END IF; END \$\$;"
sudo -u postgres psql -tc "SELECT 1 FROM pg_database WHERE datname = 'chanidb'" | grep -q 1 || sudo -u postgres psql -c "CREATE DATABASE chanidb OWNER chanithu;"

# 4. Enable Apache modules and configure reverse proxies + Python CGI
sudo a2enmod cgid headers proxy proxy_http proxy_wstunnel

sudo tee /etc/apache2/sites-available/000-default.conf > /dev/null <<'EOF'
<VirtualHost *:80>
    DocumentRoot /var/www/html
    ProxyPreserveHost On

    # Python CGI + PostgreSQL
    ScriptAlias /cgi-bin/ /usr/lib/cgi-bin/
    <Directory "/usr/lib/cgi-bin">
        AllowOverride None
        Options +ExecCGI -MultiViews +SymLinksIfOwnerMatch
        Require all granted
    </Directory>

    # ttyd Web Terminal (port 8080)
    RedirectMatch 301 ^/terminal$ /terminal/
    ProxyPass /terminal/ http://127.0.0.1:8080/terminal/ upgrade=websocket
    ProxyPassReverse /terminal/ http://127.0.0.1:8080/terminal/

    # Grafana Dashboard (port 3000)
    RedirectMatch 301 ^/grafana$ /grafana/
    ProxyPass /grafana/ http://127.0.0.1:3000/ upgrade=websocket
    ProxyPassReverse /grafana/ http://127.0.0.1:3000/

    # Prometheus Metrics (port 9090)
    RedirectMatch 301 ^/prometheus$ /prometheus/
    ProxyPass /prometheus/ http://127.0.0.1:9090/
    ProxyPassReverse /prometheus/ http://127.0.0.1:9090/
</VirtualHost>
EOF

sudo systemctl reload apache2

# 5. Create Python CGI + PostgreSQL verification script
sudo tee /usr/lib/cgi-bin/dbtest.py > /dev/null <<'EOF'
#!/usr/bin/env python3
import psycopg2

print("Content-Type: text/html\n")
try:
    conn = psycopg2.connect(dbname="chanidb", user="chanithu", password="StrongPass123", host="127.0.0.1")
    cur = conn.cursor()
    cur.execute("SELECT version();")
    version = cur.fetchone()
    print(f"<h2>Connected to PostgreSQL!</h2><p>{version}</p>")
    cur.close()
    conn.close()
except Exception as e:
    print(f"<h2>Database Error:</h2><pre>{e}</pre>")
EOF

sudo chmod +x /usr/lib/cgi-bin/dbtest.py

# 6. Install kubectl and k3d
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm -f kubectl
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# 7. Create k3d Kubernetes cluster
sudo k3d cluster list | grep -q "chanithu" || sudo k3d cluster create chanithu
sudo mkdir -p /root/.kube && sudo k3d kubeconfig get chanithu | sudo tee /root/.kube/config > /dev/null

# 8. Deploy Prometheus, Node Exporter, and Grafana into Kubernetes
sudo kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['localhost:9090']
      - job_name: 'node-exporter'
        static_configs:
          - targets: ['node-exporter-service:9100']
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus:latest
          args:
            - --config.file=/etc/prometheus/prometheus.yml
            - --web.external-url=http://localhost/prometheus/
            - --web.route-prefix=/
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: config
              mountPath: /etc/prometheus/
      volumes:
        - name: config
          configMap:
            name: prometheus-config
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
spec:
  selector:
    app: prometheus
  ports:
    - port: 9090
      targetPort: 9090
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
      containers:
        - name: node-exporter
          image: prom/node-exporter:latest
          args:
            - --path.procfs=/host/proc
            - --path.sysfs=/host/sys
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
            - name: sys
              mountPath: /host/sys
              readOnly: true
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
---
apiVersion: v1
kind: Service
metadata:
  name: node-exporter-service
spec:
  selector:
    app: node-exporter
  ports:
    - port: 9100
      targetPort: 9100
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:latest
          ports:
            - containerPort: 3000
          env:
            - name: GF_SECURITY_ADMIN_USER
              value: "admin"
            - name: GF_SECURITY_ADMIN_PASSWORD
              value: "StrongPass123"
            - name: GF_SERVER_ROOT_URL
              value: "http://localhost/grafana/"
            - name: GF_SERVER_SERVE_FROM_SUB_PATH
              value: "true"
---
apiVersion: v1
kind: Service
metadata:
  name: grafana-service
spec:
  selector:
    app: grafana
  ports:
    - port: 3000
      targetPort: 3000
EOF

# 9. Wait for Kubernetes pods and start port-forwards
sudo kubectl rollout status deployment/prometheus deployment/grafana
sudo kubectl port-forward service/prometheus-service 9090:9090 > /var/log/prometheus-pf.log 2>&1 &
sudo kubectl port-forward service/grafana-service 3000:3000 > /var/log/grafana-pf.log 2>&1 &

# 10. Start ttyd web terminal in the foreground with live logs
ttyd -c root:7530ADEF -W -i 127.0.0.1 -p 8080 -b /terminal bash
```
