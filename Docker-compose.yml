x-logging: &default-logging
  driver: "json-file"
  options:
    max-size: "50m"
    max-file: "3"


services:

  # ============================================================
  # NGINX - HTTPS REVERSE PROXY
  # ============================================================
  nginx:
    image: nginx:1.28-alpine
    container_name: nginx
    restart: unless-stopped

    ports:
      - "80:80"
      - "443:443"

    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro

    depends_on:
      grafana:
        condition: service_healthy

    networks:
      - monitoring

    logging: *default-logging


  # ============================================================
  # PROMETHEUS - METRICS
  # ============================================================
  prometheus:
    image: prom/prometheus:v3.13.1
    container_name: prometheus
    restart: unless-stopped

    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=15d
      - --storage.tsdb.retention.size=10GB
      - --web.enable-lifecycle

    volumes:
      - prometheus_data:/prometheus
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/rules:/etc/prometheus/rules:ro
      - ./prometheus/targets:/etc/prometheus/targets:ro

    expose:
      - "9090"

    healthcheck:
      test:
        [
          "CMD-SHELL",
          "wget --no-verbose --tries=1 --spider http://127.0.0.1:9090/-/healthy || exit 1"
        ]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

    mem_limit: 4g
    cpus: "2.0"

    networks:
      - monitoring

    logging: *default-logging


  # ============================================================
  # GRAFANA - DASHBOARDS
  # ============================================================
  grafana:
    image: grafana/grafana:13.1.1
    container_name: grafana
    restart: unless-stopped

    environment:
      GF_SECURITY_ADMIN_USER: ${GRAFANA_ADMIN_USER}
      GF_SECURITY_ADMIN_PASSWORD__FILE: /run/secrets/grafana_admin_password

      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_USERS_ALLOW_ORG_CREATE: "false"

      GF_SERVER_DOMAIN: ${GRAFANA_DOMAIN}
      GF_SERVER_ROOT_URL: https://${GRAFANA_DOMAIN}/

      GF_SECURITY_COOKIE_SECURE: "true"
      GF_SECURITY_COOKIE_SAMESITE: strict

      GF_AUTH_ANONYMOUS_ENABLED: "false"

    secrets:
      - grafana_admin_password

    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro

    expose:
      - "3000"

    healthcheck:
      test:
        [
          "CMD-SHELL",
          "wget --no-verbose --tries=1 --spider http://127.0.0.1:3000/api/health || exit 1"
        ]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

    depends_on:
      prometheus:
        condition: service_healthy
      loki:
        condition: service_started

    mem_limit: 1g
    cpus: "1.0"

    networks:
      - monitoring

    logging: *default-logging


  # ============================================================
  # ALERTMANAGER - ALERT ROUTING
  # ============================================================
  alertmanager:
    image: prom/alertmanager:v0.28.0
    container_name: alertmanager
    restart: unless-stopped

    command:
      - --config.file=/etc/alertmanager/alertmanager.yml
      - --storage.path=/alertmanager

    secrets:
      - slack_webhook_url

    volumes:
      - alertmanager_data:/alertmanager
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro

    expose:
      - "9093"

    healthcheck:
      test:
        [
          "CMD-SHELL",
          "wget --no-verbose --tries=1 --spider http://127.0.0.1:9093/-/healthy || exit 1"
        ]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 20s

    networks:
      - monitoring

    logging: *default-logging


  # ============================================================
  # LOKI - LOG STORAGE
  # ============================================================
  loki:
    image: grafana/loki:3.5.5
    container_name: loki
    restart: unless-stopped

    command:
      - -config.file=/etc/loki/loki-config.yml

    volumes:
      - loki_data:/loki
      - ./loki/loki-config.yml:/etc/loki/loki-config.yml:ro

    expose:
      - "3100"

    healthcheck:
      test:
        [
          "CMD-SHELL",
          "wget --no-verbose --tries=1 --spider http://127.0.0.1:3100/ready || exit 1"
        ]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

    mem_limit: 2g
    cpus: "1.5"

    networks:
      - monitoring

    logging: *default-logging


  # ============================================================
  # GRAFANA ALLOY - LOG COLLECTION
  # ============================================================
  alloy:
    image: grafana/alloy:v1.18.0
    container_name: alloy
    restart: unless-stopped

    command:
      - run
      - --disable-reporting
      - /etc/alloy/config.alloy

    volumes:
      - ./alloy/config.alloy:/etc/alloy/config.alloy:ro

      # Host logs
      - /var/log:/var/log:ro

      # Docker socket - container discovery
      - /var/run/docker.sock:/var/run/docker.sock:ro

      # Docker container logs
      - /var/lib/docker/containers:/var/lib/docker/containers:ro

    depends_on:
      loki:
        condition: service_healthy

    networks:
      - monitoring

    logging: *default-logging


  # ============================================================
  # NODE EXPORTER - EC2 HOST METRICS
  # ============================================================
  node-exporter:
    image: prom/node-exporter:v1.10.2
    container_name: node-exporter
    restart: unless-stopped

    pid: host

    command:
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys
      - --path.rootfs=/host
      - --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)

    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/host:ro,rslave

    expose:
      - "9100"

    networks:
      - monitoring

    logging: *default-logging


  # ============================================================
  # BACKUP
  #
  # AWS authentication:
  # EC2 IAM ROLE
  #
  # DO NOT put AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY here.
  # ============================================================
  backup:
    image: offen/docker-volume-backup:v2.48.2
    container_name: monitoring-backup
    restart: unless-stopped

    environment:
      BACKUP_CRON_EXPRESSION: "0 3 * * *"
      BACKUP_FILENAME: "monitoring-%Y-%m-%d.tar.gz"
      BACKUP_RETENTION_DAYS: "14"

      AWS_S3_BUCKET_NAME: ${BACKUP_S3_BUCKET}
      AWS_REGION: ${AWS_REGION}

      EXEC_STOP_CONTAINERS: "grafana prometheus alertmanager loki"

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

      - prometheus_data:/backup/prometheus:ro
      - grafana_data:/backup/grafana:ro
      - alertmanager_data:/backup/alertmanager:ro
      - loki_data:/backup/loki:ro

      - ./backups:/archive

    networks:
      - monitoring

    logging: *default-logging


# ================================================================
# NETWORK
# ================================================================
networks:
  monitoring:
    driver: bridge


# ================================================================
# PERSISTENT VOLUMES
# ================================================================
volumes:
  prometheus_data:
  grafana_data:
  alertmanager_data:
  loki_data:


# ================================================================
# DOCKER SECRETS
# ================================================================
secrets:

  grafana_admin_password:
    file: ./secrets/grafana_admin_password.txt

  slack_webhook_url:
    file: ./secrets/slack_webhook_url.txt
