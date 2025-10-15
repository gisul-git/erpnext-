version: "3"

services:
  backend:
    image: frappe/erpnext:v15.82.1
    networks:
      - frappe_network
    restart: on-failure
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    environment:
      DB_HOST: db
      DB_PORT: "3306"
      MYSQL_ROOT_PASSWORD: admin
      MARIADB_ROOT_PASSWORD: admin

  db:
    image: mariadb:10.6
    networks:
      - frappe_network
    restart: on-failure
    environment:
      MYSQL_ROOT_PASSWORD: admin
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
    volumes:
      - db-data:/var/lib/mysql

  redis-cache:
    image: redis:6.2-alpine
    networks:
      - frappe_network
    restart: on-failure

  redis-queue:
    image: redis:6.2-alpine
    networks:
      - frappe_network
    restart: on-failure
    volumes:
      - redis-queue-data:/data

  redis-socketio:
    image: redis:6.2-alpine
    networks:
      - frappe_network
    restart: on-failure

  frontend:
    image: frappe/erpnext:v15.82.1
    networks:
      - frappe_network
    depends_on:
      - backend
      - websocket
    restart: on-failure
    environment:
      BACKEND: backend:8000
      SOCKETIO: websocket:9000
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    ports:
      - "8080:8080"

  websocket:
    image: frappe/erpnext:v15.82.1
    networks:
      - frappe_network
    restart: on-failure
    command: bash -c "node /home/frappe/frappe-bench/apps/frappe/socketio.js"
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs

volumes:
  db-data:
  redis-queue-data:
  sites:
  logs:

networks:
  frappe_network:
    driver: bridge
