# ZeppelinBot-MacOS
Use This Guide to self host Zeppelin STANDALONE on your Mac
1. Go to https://code.visualstudio.com/download and download for Mac
2. Go to https://www.docker.com/products/docker-desktop/ And download for Apple Sillicon.
3. type `git clone https://github.com/ZeppelinBot/Zeppelin` (Make sure you can use github on your VSCode (Visual Studio Code) terminal.
4. go to .env.example, and right click the file, click Rename and type in .env (erase .example)
5. Fill in the Values under Standalone and General, Put in the vaules.
 ==========================
 GENERAL OPTIONS
 ==========================

 32 character encryption key
KEY= #use `openssl rand -hex 16openssl rand -hex 16` in your terminal,and copy paste the output.

Values from the Discord developer portal

CLIENT_ID= Get the Client id
CLIENT_SECRET= Reset and get the Client Secret
BOT_TOKEN= Get the Bot Token

 The defaults here automatically work for the development environment.
 For production, change localhost:3300 to your domain.
DASHBOARD_URL=https://localhost:3300
API_URL=https://localhost:3300/api

 Comma-separated list of user IDs who should have access to the bot's global commands
STAFF= #add user ids for who will have access to the global commands (you will know later)

 A comma-separated list of server IDs that should be allowed by default
DEFAULT_ALLOWED_SERVERS= #Server IDs for which servers Zeppelin will be used in

 Only required if relevant feature is used
#FISHFISH_API_KEY= #fishfish API key (idk this)

#DEFAULT_SUCCESS_EMOJI= #put the success emoji
#DEFAULT_ERROR_EMOJI= #put the error emoji


 ==========================
 PRODUCTION - STANDALONE
NOTE: You only need to fill in these values for running the standalone production environment
==========================

STANDALONE_WEB_PORT=80

 The MySQL database running in the container is exposed to the host on this port,
 allowing access with database tools such as DBeaver
STANDALONE_MYSQL_PORT=3356
 Password for the Zeppelin database user
STANDALONE_MYSQL_PASSWORD= (set to any password you want but make it secure)
 Password for the MySQL root user
STANDALONE_MYSQL_ROOT_PASSWORD= (Same password as above)

Now we need to alter docker-compose.standalone.yml 
To make it work for MacOS.
just copy paste this:
`version: '3'
name: zeppelin-prod

volumes:
  mysql-data:

services:
  mysql:
    image: mysql:8.0
    platform: linux/amd64   # <-- platform tweak here
    environment:
      MYSQL_ROOT_PASSWORD: ${STANDALONE_MYSQL_ROOT_PASSWORD:?Missing STANDALONE_MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: zeppelin
      MYSQL_USER: zeppelin
      MYSQL_PASSWORD: ${STANDALONE_MYSQL_PASSWORD:?Missing STANDALONE_MYSQL_PASSWORD}
    ports:
      - 127.0.0.1:${STANDALONE_MYSQL_PORT:?Missing STANDALONE_MYSQL_PORT}:3306
    volumes:
      - mysql-data:/var/lib/mysql
    command:
      - "--skip-log-bin"
    healthcheck:
      test: ["CMD", "/usr/bin/mysql", "--host=127.0.0.1", "--user=root", "--password=${STANDALONE_MYSQL_ROOT_PASSWORD}", "--execute", "SHOW DATABASES;"]
      interval: 1s
      timeout: 5s
      retries: 60

  nginx:
    build:
      context: .
      dockerfile: docker/production/nginx/Dockerfile
    platform: linux/amd64   # <-- platform tweak here
    ports:
      - "${STANDALONE_WEB_PORT:?Missing STANDALONE_WEB_PORT}:443"

  migrate:
    image: dragory/zeppelin
    platform: linux/amd64   # <-- platform tweak here
    depends_on:
      mysql:
        condition: service_healthy
    environment: &env
      - NODE_ENV=production
      - DB_HOST=mysql
      - DB_PORT=3306
      - DB_USER=zeppelin
      - DB_PASSWORD=${STANDALONE_MYSQL_PASSWORD}
      - DB_DATABASE=zeppelin
      - API_PATH_PREFIX=/api
    env_file:
      - .env
    working_dir: /zeppelin/backend
    command: ["npm", "run", "migrate-prod"]

  api:
    image: dragory/zeppelin
    platform: linux/amd64   # <-- platform tweak here
    depends_on:
      migrate:
        condition: service_completed_successfully
    restart: on-failure
    environment: *env
    env_file:
      - .env
    working_dir: /zeppelin/backend
    command: ["npm", "run", "start-api-prod"]

  bot:
    image: dragory/zeppelin
    platform: linux/amd64   # <-- platform tweak here
    depends_on:
      migrate:
        condition: service_completed_successfully
    restart: on-failure
    environment: *env
    env_file:
      - .env
    working_dir: /zeppelin/backend
    command: ["npm", "run", "start-bot-prod"]

  dashboard:
    image: dragory/zeppelin
    platform: linux/amd64   # <-- platform tweak here
    depends_on:
      migrate:
        condition: service_completed_successfully
    restart: on-failure
    environment: *env
    env_file:
      - .env
    working_dir: /zeppelin/dashboard
    command: ["node", "serve.js"]`

  #What does this do? It makes Docker emulate linux environments, and if you don't change the code, it will say platform errors, because MacOS is ARM-64 but the code is expecting AMD-86.
 **NOW, LOGIN TO DOCKER DESKTOP**
**now go to `docker/production/nginx/default.conf` and paste this code**
`server {
listen 80;
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  server_name _;

  # Using a variable here stops nginx from crashing if the dev container is restarted or becomes otherwise unavailable
  set $backend_upstream "http://api:3001";
  set $dashboard_upstream "http://dashboard:3002";

  location / {
    # Using a variable in proxy_pass also requires resolver to be set.
    # This is the address of the internal docker compose DNS server.
    resolver 127.0.0.11;
    proxy_pass $dashboard_upstream$uri$is_args$args;
  }

  location /api {
    resolver 127.0.0.11;
    proxy_pass $backend_upstream$uri$is_args$args;
    proxy_redirect off;

    client_max_body_size 200M;
  }

  ssl_certificate /etc/ssl/certs/localhost-cert.pem;
  ssl_certificate_key /etc/ssl/private/localhost-cert.key;

  ssl_session_timeout 1d;
  ssl_session_cache shared:MozSSL:10m;
  ssl_session_tickets off;

  ssl_protocols TLSv1.3;
  ssl_prefer_server_ciphers off;
}
