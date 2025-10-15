version: "3"

services:
  # Database
  db:
    image: mariadb:10.6
    networks:
      - frappe_network
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --skip-name-resolve
      - --skip-host-cache
      - --max-allowed-packet=256M
      - --innodb-buffer-pool-size=1G
    volumes:
      - db-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-padmin"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Cache
  redis-cache:
    image: redis:6.2-alpine
    networks:
      - frappe_network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

  # Redis Queue
  redis-queue:
    image: redis:6.2-alpine
    networks:
      - frappe_network
    restart: unless-stopped
    volumes:
      - redis-queue-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

  # Redis SocketIO
  redis-socketio:
    image: redis:6.2-alpine
    networks:
      - frappe_network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

  # Configurator - Creates site on first run
  configurator:
    image: frappe/erpnext:v15.82.1
    networks:
      - frappe_network
    restart: "no"
    depends_on:
      db:
        condition: service_healthy
      redis-cache:
        condition: service_healthy
      redis-queue:
        condition: service_healthy
    entrypoint:
      - bash
      - -c
    command:
      - >
        ls -1 apps > sites/apps.txt;
        bench set-config -g db_host $$DB_HOST;
        bench set-config -gp db_port $$DB_PORT;
        bench set-config -g redis_cache "redis://$$REDIS_CACHE";
        bench set-config -g redis_queue "redis://$$REDIS_QUEUE";
        bench set-config -g redis_socketio "redis://$$REDIS_SOCKETIO";
        bench set-config -gp socketio_port $$SOCKETIO_PORT;
        if [ ! -d "sites/$$SITE_NAME" ]; then
          echo "Creating site $$SITE_NAME...";
          bench new-site $$SITE_NAME 
            --mariadb-root-password $$MYSQL_ROOT_PASSWORD 
            --admin-password $$ADMIN_PASSWORD 
            --no-mariadb-socket 
            --install-app erpnext;
        else
          echo "Site $$SITE_NAME already exists, skipping creation.";
        fi
    environment:
      DB_HOST: db
      DB_PORT: "3306"
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      ADMIN_PASSWORD: ${ADMIN_PASSWORD}
      SITE_NAME: ${SITE_NAME}
      REDIS_CACHE: redis-cache:6379
      REDIS_QUEUE: redis-queue:6379
      REDIS_SOCKETIO: redis-socketio:6379
      SOCKETIO_PORT: "9000"
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs

  # Backend - Gunicorn
  backend:
    image: frappe/erpnext:v15.82.1
    networks:
      - frappe_network
    restart: unless-stopped
    depends_on:
      configurator:
        condition: service_completed_successfully
    command: ["bench", "serve", "--bind", "0.0.0.0:8000"]
    environment:
      DB_HOST: db
      DB_PORT: "3306"
      REDIS_CACHE: redis-cache:6379
      REDIS_QUEUE: redis-queue:6379
      REDIS_SOCKETIO: redis-socketio:6379
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs

  # Queue Worker - Short
  queue-short:
    image: frappe/erpnext:v15.82.1
    networks:
      - frappe_network
    restart: unless-stopped
    depends_on:
      configurator:
        condition: service_completed_successfully
    command: ["bench", "worker", "--queue", "short"]
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs

  # Queue Worker - Default
  queue-default:
    image: frappe/erpnext:v15.82.1
    networks:
      - frappe_network
    restart: unless-stopped
    depends_on:
      configurator:
        condition: service_completed_successfully
    command: ["bench", "worker", "--queue", "default"]
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs

  # Queue Worker - Long
  queue-long:
    image: frappe/erpnext:v15.82.1
    networks:
      - frappe_network
    restart: unless-stopped
    depends_on:
      configurator:
        condition: service_completed_successfully
    command: ["bench", "worker", "--queue", "long"]
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs

  # Scheduler
  scheduler:
    image: frappe/erpnext:v15.82.1
    networks:
      - frappe_network
    restart: unless-stopped
    depends_on:
      configurator:
        condition: service_completed_successfully
    command: ["bench", "schedule"]
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs

  # WebSocket
  websocket:
    image: frappe/erpnext:v15.82.1
    networks:
      - frappe_network
    restart: unless-stopped
    depends_on:
      configurator:
        condition: service_completed_successfully
    environment:
      REDIS_SOCKETIO: redis-socketio:6379
    command: ["node", "/home/frappe/frappe-bench/apps/frappe/socketio.js"]
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs

  # Frontend - Nginx
  frontend:
    image: frappe/erpnext:v15.82.1
    networks:
      - frappe_network
    depends_on:
      - backend
      - websocket
    restart: unless-stopped
    environment:
      BACKEND: backend:8000
      SOCKETIO: websocket:9000
      FRAPPE_SITE_NAME_HEADER: $$host
      UPSTREAM_REAL_IP_ADDRESS: 127.0.0.1
      UPSTREAM_REAL_IP_HEADER: X-Forwarded-For
      UPSTREAM_REAL_IP_RECURSIVE: "off"
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    ports:
      - "80:8080"
      # For HTTPS, uncomment below and configure SSL
      # - "443:8443"

volumes:
  db-data:
  redis-queue-data:
  sites:
  logs:

networks:
  frappe_network:
    driver: bridge


    # Database Configuration
MYSQL_ROOT_PASSWORD=YourStrongPassword123!

# ERPNext Site Configuration
SITE_NAME=erp.yourdomain.com
# OR use your public IP if no domain
# SITE_NAME=123.45.67.89

# ERPNext Admin Password
ADMIN_PASSWORD=YourAdminPassword123!

# Note: Change these passwords before deployment!
