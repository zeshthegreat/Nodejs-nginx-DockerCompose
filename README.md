# Nginx with Docker and Docker Compose

This project runs a full-stack application behind an Nginx reverse proxy using Docker Compose. The stack includes a frontend client, three load-balanced Node.js API instances, a MongoDB database, Mongo Express for database management, and an Nginx container handling SSL termination, rate limiting, caching, and compression.


## Technologies Used

- React 19
- Node.js & Express
- MongoDB & Mongo Express
- Nginx (Reverse Proxy, Load Balancing, Caching, SSL)
- Docker & Docker Compose

## Project Structure

```
├── backend/
│   ├── Dockerfile          # Node.js backend container configuration
│   ├── index.js            # Express API server and MongoDB connection
│   └── package.json        # Backend dependencies
├── frontend/
│   ├── Dockerfile          # React frontend container configuration
│   ├── package.json        # Frontend dependencies
│   ├── public/             # Static assets (HTML template, icons)
│   └── src/
│       ├── components/     # React UI components (UserForm, UserList, Input)
│       ├── services/       # API service and Axios client
│       ├── App.js          # Main application component
│       └── index.js        # React entry point
├── nginx/
│   └── default.conf        # Nginx reverse proxy, SSL, and load balancer configuration
├── docker-compose.yaml     # Docker Compose orchestration
└── README.md               # Project documentation and guide
```

## Step 1: Move Nginx into Docker Compose

In a standard host setup, Nginx runs directly on the operating system through systemd and proxies requests to local ports:

```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://localhost:3000;
    }
    location /api/ {
        proxy_pass http://localhost:3001/;
    }
    location /mongo/ {
        proxy_pass http://localhost:8081;
    }
}
```

To run Nginx inside a container alongside the application services:

1. Stop and disable the host Nginx service:
   ```bash
   sudo systemctl stop nginx && sudo systemctl disable nginx
   ```

2. Create a folder named `nginx` in the project root:
   ```bash
   mkdir -p ~/nginx-docker-compose/nginx
   ```

3. Create `~/nginx-docker-compose/nginx/default.conf` and update proxy targets from `localhost` to the Docker Compose service names:
   - `http://localhost:3000` becomes `http://web:3000`
   - `http://localhost:3001/` becomes `http://api:3001/`
   - `http://localhost:8081` becomes `http://mongoExpress:8081`

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    location / {
        proxy_pass http://web:3000;
    }
    location /api/ {
        proxy_pass http://api:3001/;
    }
    location /mongo/ {
        proxy_pass http://mongoExpress:8081;
    }
}
```

## Step 2: Set up HTTPS with Let's Encrypt

Certbot runs on the host machine to obtain SSL certificates, which are then mounted into the Nginx container as read-only volumes.

1. Install Certbot on the host:
   ```bash
   sudo apt install snapd -y
   sudo snap install core
   sudo snap refresh core
   sudo apt remove certbot -y
   sudo snap install --classic certbot
   sudo ln -s /snap/bin/certbot /usr/bin/certbot
   ```

2. Generate certificates for your domain:
   ```bash
   sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
   ```

3. Expose port 443 and mount certificates in `~/nginx-docker-compose/docker-compose.yaml`:
   ```yaml
   nginx:
     image: nginx:1.30.4-alpine3.24
     ports:
       - 80:80
       - 443:443
     volumes:
       - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
       - /etc/letsencrypt:/etc/letsencrypt:ro
   ```

4. Configure HTTP to HTTPS redirection and SSL certificates in `~/nginx-docker-compose/nginx/default.conf`:
   ```nginx
   server {
       listen 80;
       server_name yourdomain.com www.yourdomain.com;
       return 301 https://$host$request_uri;
   }

   server {
       listen 443 ssl;
       server_name yourdomain.com www.yourdomain.com;

       ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
       ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

       location /api/ {
           proxy_pass http://api:3001/;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
       }

       location /mongo/ {
           proxy_pass http://mongoExpress:8081/;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
       }

       location / {
           proxy_pass http://web:3000;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
       }
   }
   ```

5. Test the Nginx container configuration:
   ```bash
   docker exec -it <container_name_or_id> nginx -t
   ```

## Step 3: Configure load balancing

To distribute backend traffic across multiple containers, add multiple API instances to Docker Compose and define an upstream block in Nginx.

1. In `~/nginx-docker-compose/docker-compose.yaml`, define multiple API services (`api1`, `api2`, `api3`):
   ```yaml
   api1:
     build:
       context: backend
     expose:
       - 3001
     environment:
       - API_NAME=api1
       - DB_URL=mongodb://admin:pass123@db:27017/?authSource=admin
     restart: always
     depends_on:
       - db
     command: ["node", "index.js"]

   api2:
     build:
       context: backend
     expose:
       - 3001
     environment:
       - API_NAME=api2
       - DB_URL=mongodb://admin:pass123@db:27017/?authSource=admin
     restart: always
     depends_on:
       - db
     command: ["node", "index.js"]

   api3:
     build:
       context: backend
     expose:
       - 3001
     environment:
       - API_NAME=api3
       - DB_URL=mongodb://admin:pass123@db:27017/?authSource=admin
     restart: always
     depends_on:
       - db
     command: ["node", "index.js"]
   ```

2. Make sure backend code reads the `API_NAME` environment variable if you want to inspect which instance handled a request.

3. In `~/nginx-docker-compose/nginx/default.conf`, add an upstream group using the `least_conn` algorithm:
   ```nginx
   upstream backend_servers {
       least_conn;
       server api1:3001;
       server api2:3001;
       server api3:3001;
   }
   ```

4. Route API requests to the upstream block:
   ```nginx
   location /api/ {
       proxy_pass http://backend_servers/;
       proxy_set_header Host $host;
       proxy_set_header X-Real-IP $remote_addr;
   }
   ```

## Step 4: Configure HTTP caching

Caching responses directly in Nginx reduces repeated database and backend calculations.

1. Define the cache zone at the top level of `~/nginx-docker-compose/nginx/default.conf`:
   ```nginx
   proxy_cache_path /data/nginx/cache 
                   keys_zone=api_cache:10m
                   max_size=1g
                   inactive=30m
                   use_temp_path=off;
   ```

2. Add a persistent volume for the cache in `~/nginx-docker-compose/docker-compose.yaml`:
   ```yaml
   nginx:
     volumes:
       - nginx_cache:/data/nginx/cache
       - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
       - /etc/letsencrypt:/etc/letsencrypt:ro

   volumes:
     mongodb-sarvin:
     nginx_cache:
   ```

3. Enable caching for selected endpoints while keeping other routes uncached:
   ```nginx
   # Cached endpoint
   location = /api/proccessData {
       proxy_cache api_cache;
       proxy_cache_valid 200 10m;
       proxy_pass http://backend_servers/proccessData;
   }

   # Uncached endpoints
   location /api/ {
       proxy_pass http://backend_servers/;
   }
   ```

## Step 5: Enable HTTP compression

Gzip compresses text-based assets like HTML, CSS, JavaScript, and JSON before sending them to the browser, reducing bandwidth usage.

Add these directives inside `~/nginx-docker-compose/nginx/default.conf`:

```nginx
gzip on;
gzip_min_length 1000;

gzip_types
    text/plain
    text/css
    application/json
    application/javascript
    text/xml
    application/xml
    image/svg+xml;

gzip_comp_level 5;
```

## Step 6: Add security headers

Hardening headers protect the site against common browser-side exploits and hide server version details.

Add these headers to `~/nginx-docker-compose/nginx/default.conf`:

```nginx
server_tokens off;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```

## Step 7: Apply rate limiting

Rate limiting prevents abusive traffic patterns by restricting how many requests a client IP can make per second.

1. Define a 10 MB memory zone tracking binary IP addresses at 5 requests per second in `~/nginx-docker-compose/nginx/default.conf`:
   ```nginx
   limit_req_zone $binary_remote_addr zone=api_limit:10m rate=5r/s;
   ```

2. Apply the limit zone and return status 429 when a client exceeds the threshold:
   ```nginx
   location /api/ {
       limit_req zone=api_limit;
       limit_req_status 429;
       proxy_pass http://backend_servers/;
   }
   ```

## Step 8: Block direct IP address access

To prevent scanners and unauthorized traffic from accessing services via raw IP addresses, configure default server blocks that drop connections without returning data.

Add these blocks to `~/nginx-docker-compose/nginx/default.conf`:

```nginx
# Drop requests to raw IP over HTTP
server {
    listen 80 default_server;
    server_name _;
    return 444;
}

# Drop requests to raw IP over HTTPS
server {
    listen 443 default_server;
    server_name _;
    return 444;
}
```

## Running the project

1. Start all containers in detached mode:
   ```bash
   docker compose up -d --build
   ```

2. View container status:
   ```bash
   docker compose ps
   ```

3. Follow logs across services:
   ```bash
   docker compose logs -f
   ```

4. Stop all services:
   ```bash
   docker compose down
   ```
