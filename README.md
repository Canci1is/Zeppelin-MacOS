# ZeppelinBot-MacOS

Use This Guide to self-host Zeppelin STANDALONE on your Mac

1. Go to https://code.visualstudio.com/download and download for Mac  
2. Go to https://www.docker.com/products/docker-desktop/ and download for Apple Silicon  
3. Type `git clone https://github.com/ZeppelinBot/Zeppelin` (Make sure you can use GitHub on your VSCode terminal.)  
4. Then type:  
`cd Zeppelin`  
5. Go to `.env.example`, right-click the file, click Rename and type in `.env` (erase `.example`)  
6. Fill in the values under **Standalone** and **General**:

==========================
GENERAL OPTIONS  
==========================

**32 character encryption key**  
KEY=  # Paste output from `openssl rand -hex 16`

Values from the Discord Developer Portal:  
CLIENT_ID=  # Get the Client ID  
CLIENT_SECRET=  # Reset and get the Client Secret  
BOT_TOKEN=  # Get the Bot Token  

The defaults here automatically work for the development environment.  
For production, change localhost:3300 to your domain.  
DASHBOARD_URL=http://localhost:80  
API_URL=http://localhost:80/api  

Comma-separated list of user IDs who should have access to the bot's global commands  
STAFF=  # Add user IDs who will have access to global commands  

Comma-separated list of server IDs that should be allowed by default  
DEFAULT_ALLOWED_SERVERS=  # Server IDs to use Zeppelin in  

Only required if relevant features are used:  
#FISHFISH_API_KEY=  
#DEFAULT_SUCCESS_EMOJI=  
#DEFAULT_ERROR_EMOJI=  

==========================
PRODUCTION - STANDALONE  
NOTE: You only need to fill in these values for running the standalone production environment  
==========================

STANDALONE_WEB_PORT=80  

The MySQL database running in the container is exposed to the host on this port,  
allowing access with database tools such as DBeaver  
STANDALONE_MYSQL_PORT=3356  
STANDALONE_MYSQL_PASSWORD=  # Set to any secure password  
STANDALONE_MYSQL_ROOT_PASSWORD=  # Same as above

Now we need to alter `docker-compose.standalone.yml` to make it work for macOS.  
Just copy and paste this:

```
version: '3'
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
      - "${STANDALONE_WEB_PORT:?Missing STANDALONE_WEB_PORT}:80"

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
    command: ["node", "serve.js"]
```

# What does this do?  
It makes Docker emulate a Linux environment.  
If you don’t change the platform, it will show errors — because macOS is ARM64 but Zeppelin expects AMD64.

**NOW, LOGIN TO DOCKER DESKTOP**

Now go to `docker/production/nginx/default.conf` and paste this code:

```
server {
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
```

### NEXT:  
Go to https://discord.com/developers/applications → your bot → OAuth2 → Add Redirect →  
`http://localhost:80/api/auth/oauth-callback`

### Make sure Docker is open, click to start the app. if you close it, the code will stop.

### NOW: Run the code.

Type this in your terminal:

```
docker-compose -f docker-compose.standalone.yml up -d --build
```

Wait until everything says `Healthy`, `Created`, or `Running`.  
(Make sure Safari allows insecure websites or HTTP if it doesn't load.)

Then go to:  
http://localhost:80

If it doesn’t work, allow Docker through macOS firewall.

Now go to:  
https://setuptoolzep.github.io — fill in the values, copy the config output, and paste into your dashboard server config.
See more command setup at:  
https://zeppelin.wiki/  
and  
https://zeppelin.gg/docs

If you want to run your code without your parents noticing, first set brightness to lowest, though it still has little bit of brightness on, so type ```caffeinate -d -i -m -u & sleep 4; pmset displaysleepnow``` IN YOUR TERMINAL APP, NOT VSCode TERMINAL. Quickly, turn off your keyboard and mouse, and it wikk automatically black out the screen, but dont worrym the -d is for Detached, which runs in the background, and this allows it to run in background. If you press anything or move your cursor, the screen will turn on again, BUT THIS IS ONLY SCREEN SLEEPING WITH CODE RUNNING. THE SYSTEM ISN'T OFF.
---

That's it.  
Good luck, and for help, join the support server:  
https://discord.com/invite/UeTCfaK


Made with a lot of time by `invalidsyntaxerror`. 
