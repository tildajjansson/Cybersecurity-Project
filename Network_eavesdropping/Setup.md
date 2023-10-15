## Steps required to set up NodeBB with Redis and NGINX on a Ubuntu subsystem within Windows 10

### Configuring NodeBB to use Redis as the database:

1. **Install Redis Server Version 6.0.16**
    ```bash
    sudo apt-get install redis-server
    ```

2. **Start Redis**
    ```bash
    redis-server
    ```

3. **Download and Install NodeBB v3.4.3**
    ```bash
    git clone https://github.com/NodeBB/NodeBB.git
    ```

4. **Run the NodeBB setup command**
    - Go with the default options, but input "redis" when prompted with database type.
    - Host IP or address of the Redis instance: `127.0.0.1` since Redis is installed on the same machine as NodeBB.
    - Host port of the Redis instance: `6379`.
    - Password of the Redis database: leave this empty.
    - Which database to use: `0`

    When prompted for administrator info we entered:
    - Administrator username: `Tildaja`
    - Administrator email: `tildaja@kth.se`
    - Administrator password: `Password123`

### Install and Configure NGINX:

1. **Install NGINX version: nginx/1.18.0 (Ubuntu)**
    ```bash
    sudo apt-get install nginx
    ```

2. **Create the directory and generate the self-signed certificate**
    ```bash
    sudo mkdir -p /etc/nginx/ssl/
    sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/nginx.key -out /etc/nginx/ssl/nginx.crt
    ```
    - This will create a self-signed certificate valid for one year. When prompted for info, leave those empty.

3. **Create a new configuration file**
    ```bash
    sudo nano /etc/nginx/sites-available/nodebb
    ```
    - Add the following to the NGINX file `nodebb`, see [nodebb file](./nodebb):
    ```nginx
    # HTTP Server
    server {
        source_charset utf-8;
        listen 80;
        server_name localhost;

        # Redirecting all HTTP traffic to HTTPS using this line:
        rewrite ^ https://$server_name$request_uri permanent;
    }

    server {
        source_charset utf-8;
        listen 443 ssl;
        server_name localhost;
        
        # Using self-signed certificate
        ssl_certificate /etc/nginx/ssl/nginx.crt;
        ssl_certificate_key /etc/nginx/ssl/nginx.key;

        location / {
                    proxy_set_header X-Real-IP $remote_addr;
                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header X-Forwarded-Proto $scheme;
                    proxy_set_header Host $http_host;
                    proxy_set_header X-NginX-Proxy true;

                    proxy_pass http://127.0.0.1:4567;
                    proxy_redirect off;

                    # Socket.IO Support
                    proxy_http_version 1.1;
                    proxy_set_header Upgrade $http_upgrade;
                    proxy_set_header Connection "upgrade";
        }
    }
    ```

4. **Create a symbolic link to the 'sites-enabled' directory**
    ```bash
    sudo ln -s /etc/nginx/sites-available/nodebb /etc/nginx/sites-enabled/
    ```

5. **Modify the NodeBB `config.json`**
    - Update the URL field to reflect the change from HTTP to HTTPS in the forum's URL as it ensures that NodeBB recognizes the secure URL and operates accordingly, see [config.json](./config.json).
    ```json
    {
        "url": "https://localhost",
        "secret": "a25a18f4-71eb-445c-9d32-c270985c010b",
        "database": "redis",
        "port": "4567",
        "redis": {
            "host": "127.0.0.1",
            "port": "6379",
            "password": "",
            "database": "0"
        }
    }
    ```

### Final Steps:

1. **After running `./nodebb setup` and ensuring it completed successfully, navigate to the NodeBB directory and run:**
    ```bash
    ./nodebb stop
    sudo service nginx stop
    ./nodebb build
    sudo service nginx start
    ./nodebb start
    ```

2. **Now you should be able to access NodeBB via `https://localhost` or `http://localhost`**
    - Since it's a self-signed certificate, your browser will warn you about the connection not being private. Proceed to the site.

3. **To stop NodeBB, Redis and NGINX use:**
    ```bash
    ./nodebb stop
    sudo service nginx stop
    redis-cli shutdown
    ```
