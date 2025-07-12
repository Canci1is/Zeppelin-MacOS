# Zeppelin-MacOS-zeppelin-guide
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
