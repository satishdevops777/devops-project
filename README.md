# Devops-project

## 1. Frontend
- The Frontend is a web service responsible for serving the UI (webframe) of the RoboShop application to end users.
- It uses Nginx to:
  - Serve static web content
  - Act as a reverse proxy to backend microservices
  - Provide a health endpoint for monitoring
 
### Responsibilities of the Frontend Service
1️⃣ Serve Web Content
  - HTML
  - CSS
  - JavaScript
  - Images
- These files form the webframe (UI) that users interact with in their browser.
  ```test
  /index.html
  /css/style.css
  /js/app.js
  ```
2️⃣ Reverse Proxy to Backend Services
  - The frontend does not contain business logic.
  - Instead, it forwards API calls to backend services like:
    ```test
    catalogue
    user
    cart
    shipping
    payment
    ```
    ```test
    Browser → Nginx (Frontend) → Catalogue Service → MongoDB
    ```
 3️⃣ Central Entry Point (Single URL)
    - Users access the application using one URL:
      ```test
      https://roboshop.example.com
      ```
    - Nginx internally routes requests:
      ```test
      /api/catalogue → catalogue service
      /api/cart       → cart service
      ```
    This avoids exposing backend services directly.

### 1. install Nginx
    ```
    dnf install nginx -y
    ```
### 2. Start & Enable Nginx service
    ```
    systemctl enable nginx  
    systemctl start nginx
    ```
### 3. Remove the default content that web server is serving.
    ```
    rm -rf /usr/share/nginx/html/*
    ```
### 4. Download the frontend content
    ```
    curl -o /tmp/frontend.zip https://roboshop-artifacts.s3.amazonaws.com/frontend.zip 
    ```
### 5. Extract the frontend content.
    ```
    cd /usr/share/nginx/html 
    unzip /tmp/frontend.zip
    ```
### 6. Create Nginx Reverse Proxy Configuration.
    ```
    vim /etc/nginx/default.d/roboshop.conf #Create Nginx Reverse Proxy Configuration.
    ```

### 1️⃣ proxy_http_version 1.1;
    - proxy_http_version 1.1; #Forces Nginx to use HTTP/1.1 when talking to backend services.
    - WHY THIS IS IMPORTANT
      - HTTP/1.0:
        - No keep-alive
        - New connection for every request ❌
    - HTTP/1.1:
      - Connection reuse (keep-alive) ✅
      - Faster
      - Required for modern APIs
   - Without HTTP/1.1:
      ```
      Browser → Nginx → Backend
      (NEW connection every request)
      ```
  - With HTTP/1.1:
    ```
    Browser → Nginx → Backend
    (One connection reused)
    ```
### 📌 SRE Impact
  - Lower latency, Less CPU, Better scalability under load

### 2️⃣ location /images/ {} – Static Images
  - Any request starting with: /images/
    | URL                    | Match? |
    | ---------------------- | ------ |
    | /images/logo.png       | ✅      |
    | /images/products/1.jpg | ✅      |
    | /css/style.css         | ❌      |
  - expires 5s; Tells the browser: “Cache this image for 5 seconds”
    - REAL BROWSER BEHAVIOR
      - User loads /images/logo.png
      - Browser caches it
      - Next request within 5s → no server call
      - After 5s → browser re-fetches
    📌 WHY ONLY 5 SECONDS?
    - Product images may change
    - Prevents stale UI after deployment
    
    📌 Production Note
    - CDN may cache longer
    - Frontend keeps it short


  - root /usr/share/nginx/html;
    - Nginx looks for files here: root /usr/share/nginx/html/images;
      ```test
      /usr/share/nginx/html/
       ├── images/
       │   ├── logo.png
       │   ├── product1.jpg
       │   └── placeholder.jpg
      ```
  - try_files $uri /images/placeholder.jpg;
    - WHAT IT DOES
      - Try to find requested image
      - If not found, serve a fallback image
      -  User requests: /images/product999.jpg
      - If file exists: ✅ served normally
      - If file does NOT exist:
        - ❌ 404 avoided
        - ✅ placeholder.jpg served
      - 📌 WHY THIS IS IMPORTANT
        - UI never breaks
        - No broken image icons
        - Better user experience
  ```nginx
  location /images/ {
      expires 5s;
      root   /usr/share/nginx/html;
      try_files $uri /images/placeholder.jpg;
    }
  ```

3️⃣ API Reverse Proxy (Core Logic)
    ```nginx
    location /api/catalogue/ { proxy_pass http://localhost:8080/; }
    ```
  - WHAT THIS MEANS
    - Any request starting with /api/catalogue/
    - Forward it to backend service running on port 8080
    - User requests GET /api/cart/add
    -   What Nginx does proxy_pass → http://localhost:8080/api/cart/add
    -   BACKEND RESPONDS
      ```arduino
      200 OK
    { "status": "item added" }
      ```
  ```nginx
  location /api/catalogue/ { proxy_pass http://localhost:8080/; }
  location /api/user/ { proxy_pass http://localhost:8080/; }
  location /api/cart/ { proxy_pass http://localhost:8080/; }
  location /api/shipping/ { proxy_pass http://localhost:8080/; }
  location /api/payment/ { proxy_pass http://localhost:8080/; }
  ```
  
4️⃣ /health – Health Check Endpoint
  - creates a health and monitoring endpoint in Nginx
  ```nginx
  location /health {
    stub_status on;
    access_log off;
  }
  ```
### 🧠 What is a Health Endpoint?
  - A health endpoint is a URL that:
    - Returns basic service status
    - Is checked automatically
    - Helps systems decide:
      👉 “Is this service healthy enough to receive traffic?”
      ```test
      http://frontend:80/health
      ```
  
      ```test
      curl http://localhost/health
      ```
  - If Nginx is running → response comes back
  - If Nginx is down → request fails

### 🔹 stub_status on;
    - What is stub_status?
      - A built-in Nginx module
      - Shows internal Nginx metrics
      - No backend calls involved
      - sample example
        ```
        Active connections: 2
        server accepts handled requests
        1200 1200 4500
        Reading: 0 Writing: 1 Waiting: 1
        ```
    📌 Healthy system
      - accepts ≈ handled
    📌  Problem
      - accepts ≠ handled → dropped connections

    
  ### 🎯 Who Uses /health?
  | Component        | Why                                   |
  | ---------------- | ------------------------------------- |
  | Load Balancer    | Route traffic only to healthy servers |
  | Kubernetes       | Restart unhealthy pods                |
  | Monitoring tools | Alert on failures                     |
  | SREs             | Quick sanity checks                   |

  | State   | Meaning                 |
  | ------- | ----------------------- |
  | Reading | Reading request headers |
  | Writing | Sending response        |
  | Waiting | Idle keep-alive         |

### 📌 SRE Debugging
    - High Writing → slow backend
    - High Reading → slow clients
    - High Waiting → healthy idle state


### What /health DOES check
✅ Nginx process is alive
✅ Nginx can accept connections

### What /health DOES NOT check
❌ Backend services
❌ Database health
❌ API correctness
📌 This is a shallow health check

| Endpoint   | Purpose                 |
| ---------- | ----------------------- |
| `/health`  | Is Nginx alive?         |
| `/ready`   | Are backends reachable? |
| `/metrics` | Prometheus metrics      |


### access_log off;
  - By default, Nginx records every HTTP request in an access log file.
    ```test
    10.0.1.15 - - [06/Jan/2026:10:12:01 +0530] "GET /api/cart HTTP/1.1" 200 512 "-" "Mozilla/5.0"
    ```
  - This happens for:
    - Page loads
    - API calls
    - Images
    - Health checks

Meaning (simple words)
“Do NOT log HTTP requests for this location.”
- When placed inside a location block, it:
- Disables access logging only for that URL
- Other URLs are still logged normally

Without access_log off;
- Logs grow fast
- Disk fills up
- Nginx crashes
- Entire frontend goes DOWN ❌
📌 This has caused real outages

With access_log off; (Correct Way)
- Health checks still work ✅
- Logs stay clean ✅
- Disk usage controlled ✅
- Real user issues easier to debug ✅

- Note
    - Ensure you replace the localhost with the actual ip address of those component server. Word localhost is just used to avoid the failures on the Nginx Server.

7. Restart Nginx Service to load the changes of the configuration.
  ```
  systemctl restart nginx
  ```


## 2. MongoDB

-  Developer has chosen the database MongoDB. Hence, we are trying to install it up and configure it.
-  Versions of the DB Software you will get context from the developer, Meaning we need to check with developer. Developer has shared the version information as MongoDB-4.x
-  Setup the MongoDB repo file
   /etc/yum.repos.d/mongo.repo
    ```mongodb
    [mongodb-org-4.2]  #This is the repository ID & Must be unique on the system
    name=MongoDB Repository #A human-readable name
    baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.2/x86_64/
    gpgcheck=0
    enabled=1
    ```
    | Part                       | Meaning                     |
    | -------------------------- | --------------------------- |
    | `https://repo.mongodb.org` | Official MongoDB repository |
    | `/yum/redhat/`             | For RHEL-based OS           |
    | `$releasever`              | OS version (auto-detected)  |
    | `mongodb-org/4.2/`         | MongoDB version             |
    | `x86_64/`                  | 64-bit architecture         |

  - gpgcheck=0
   - What is GPG check?
     - Verifies package authenticity
     - Ensures packages are not tampered
  - enabled=1
    - #Repository is ACTIVE
       - enabled=1 → works ✅
       - enabled=0 → repo ignored ❌
    
- Install MongoDB :
  ```
    dnf install mongodb-org -y
  ```
- Start & Enable MongoDB Service
  ```
  systemctl enable mongod 
  systemctl start mongod
  ```
  ⚠️ Common Production Issues
    | Issue               | Cause               |
    | ------------------- | ------------------- |
    | 404 error           | Wrong `$releasever` |
    | Repo unreachable    | Network / proxy     |
    | Install fails       | OS version mismatch |
    | Security audit fail | `gpgcheck=0`        |

  
  - Usually MongoDB opens the port only to localhost(127.0.0.1), meaning this service can be accessed by the application that is hosted on this server only. However, we need to access this service to be accessed by   another server, So we need to change the config accordingly.
  - Update listen address from ```127.0.0.1 to 0.0.0.0``` in ```/etc/mongod.conf```
  - Restart the service to make the changes effected.
    ```shell
    systemctl restart mongod
    ```

  ## 03. Catalogue
  - Catalogue is a microservice that is responsible for serving the list of items that displays in roboshop application.
  - Developer has chosen NodeJs, Check with developer which version of NodeJS is needed. Developer has set a context that it can work with NodeJS >18
  - Install NodeJS, By default NodeJS 10 is available, We would like to enable 18 version and install list.
  - You can list modules by using dnf module list
    ```
    dnf module disable nodejs -y
    dnf module enable nodejs:18 -y
    ```
  - Install NodeJS
    ```
    dnf install nodejs -y
    ```
  - We already discussed in Linux basics section that applications should run as nonroot user.
  - Add application User
    ```
    useradd roboshop
    ```
  - User roboshop is a function / daemon user to run the application. Apart from that we dont use this user to login to server.
  - Also, username roboshop has been picked because it more suits to our project name.
  - We keep application in one standard location. This is a usual practice that runs in the organization.
  - Lets setup an app directory.
    ```
    mkdir /app
    ```
  - Download the application code to created app directory.
    ```
    curl -o /tmp/catalogue.zip https://roboshop-artifacts.s3.amazonaws.com/catalogue.zip 
    cd /app 
    unzip /tmp/catalogue.zip
    ```

  # RoboShop – Catalogue Microservice Setup

This document explains **how to set up the Catalogue microservice** of the RoboShop application **step by step**, with clear explanations of *why* each step is required. This README is suitable for **GitHub**, **interviews**, and **real-world DevOps/SRE practice**.

---

## 📌 What is the Catalogue Service?

The **Catalogue** service is a backend microservice responsible for:

* Serving the **list of products** displayed in the RoboShop UI
* Fetching product data from **MongoDB**
* Responding to frontend API requests like `/api/catalogue`

---

## 🧱 Technology Stack

| Component       | Technology                  |
| --------------- | --------------------------- |
| Language        | NodeJS (> 18)               |
| Database        | MongoDB                     |
| Service Manager | systemd                     |
| OS              | RHEL / CentOS / Rocky Linux |

---

## ⚠️ Important Notes

* The application is **custom-built**, not available as an RPM
* All setup steps are done **manually**
* The service must run as a **non-root user**
* MongoDB is a **remote dependency**

---

## 1️⃣ Install NodeJS 18

### Why?

The developer has specified that the application works only with **NodeJS version 18 or higher**. Older versions (like NodeJS 10) are unsupported and insecure.

### List available NodeJS modules

```bash
dnf module list nodejs
```

### Disable default NodeJS module

```bash
dnf module disable nodejs -y
```

### Enable NodeJS 18

```bash
dnf module enable nodejs:18 -y
```

### Install NodeJS

```bash
dnf install nodejs -y
```

### Verify installation

```bash
node -v
```

Expected output:

```text
v18.x.x
```

---

## 2️⃣ Create Application User

### Why?

Running applications as **root is insecure**. A dedicated system user improves isolation and security.

### Create user

```bash
useradd roboshop
```

> ℹ️ `roboshop` is a **daemon user**, not intended for interactive login.

---

## 3️⃣ Create Application Directory

### Why `/app`?

Organizations usually standardize application locations for maintainability and automation.

```bash
mkdir /app
```

---

## 4️⃣ Download Application Code

### Download the catalogue application

```bash
curl -o /tmp/catalogue.zip https://roboshop-artifacts.s3.amazonaws.com/catalogue.zip
```

### Extract into `/app`

```bash
cd /app
unzip /tmp/catalogue.zip
```

This creates the NodeJS project structure, including:

* `server.js`
* `package.json`
* `schema/`

---

## 5️⃣ Install Application Dependencies

### Why?

NodeJS applications depend on external libraries defined in `package.json`.

```bash
cd /app
npm install
```

This downloads all required dependencies into `node_modules/`.

---

## 6️⃣ Configure systemd Service

### Why systemd?

Using systemd provides:

* Auto-start on reboot
* Crash recovery
* Centralized logging

---

### Create service file

```bash
vim /etc/systemd/system/catalogue.service
```

### Service definition

```ini
[Unit]
Description=Catalogue Service

[Service]
User=roboshop
Environment=MONGO=true
Environment=MONGO_URL="mongodb://<MONGODB-SERVER-IPADDRESS>:27017/catalogue"
ExecStart=/bin/node /app/server.js
SyslogIdentifier=catalogue

[Install]
WantedBy=multi-user.target
```

> ⚠️ Replace `<MONGODB-SERVER-IPADDRESS>` with the actual MongoDB server IP.

---

## 7️⃣ Load and Start the Service

### Reload systemd

```bash
systemctl daemon-reload
```

### Enable service at boot

```bash
systemctl enable catalogue
```

### Start service

```bash
systemctl start catalogue
```

### Verify status

```bash
systemctl status catalogue
```

---

## 8️⃣ Install MongoDB Client

### Why?

The catalogue service requires product data to be loaded into MongoDB. We only need the **MongoDB client**, not the server.

### Create MongoDB repo

```bash
vim /etc/yum.repos.d/mongo.repo
```

```ini
[mongodb-org-4.2]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.2/x86_64/
gpgcheck=0
enabled=1
```

### Install MongoDB shell

```bash
dnf install mongodb-org-shell -y
```

---

## 9️⃣ Load Catalogue Schema

### Why?

Without schema data:
* MongoDB is empty
* Catalogue API returns no products

### Load schema

```bash
mongo --host <MONGODB-SERVER-IPADDRESS> </app/schema/catalogue.js
```

This inserts product data into the `catalogue` database.

---

## 🔟 Update Frontend Configuration

### Why?

The frontend (Nginx) must know where the catalogue service is running.

### Config file

```text
/etc/nginx/default.d/roboshop.conf
```

### Update catalogue route

```nginx
location /api/catalogue/ {
  proxy_pass http://<CATALOGUE-SERVER-IP>:8080;
}
```

Reload Nginx after changes.

---

## 🔁 End-to-End Request Flow

```text
Browser
 → Nginx Frontend
 → /api/catalogue
 → Catalogue NodeJS Service
 → MongoDB
 → Product List
 → Browser
```

---

## ✅ Final Checklist

* [x] NodeJS 18 installed
* [x] roboshop user created
* [x] Application code deployed
* [x] Dependencies installed
* [x] systemd service running
* [x] MongoDB schema loaded
* [x] Frontend routing updated

---
