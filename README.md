# devops-project

## 1. Frontend
- The Frontend is a web service responsible for serving the UI (webframe) of the RoboShop application to end users.
- It uses Nginx to:
  - Serve static web content
  - Act as a reverse proxy to backend microservices
  - Provide a health endpoint for monitoring
 
Responsibilities of the Frontend Service

1️⃣ Serve Web Content
- HTML
- CSS
- JavaScript
- Images
These files form the webframe (UI) that users interact with in their browser.
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
    Nginx internally routes requests:
    ```test
    /api/catalogue → catalogue service
    /api/cart       → cart service
    ```
    This avoids exposing backend services directly.

1. install Nginx
```
dnf install nginx -y
```
2. Start & Enable Nginx service
```
systemctl enable nginx  
systemctl start nginx
```
3. Remove the default content that web server is serving.
```
rm -rf /usr/share/nginx/html/*
```
4. Download the frontend content
```
curl -o /tmp/frontend.zip https://roboshop-artifacts.s3.amazonaws.com/frontend.zip 
```
5. Extract the frontend content.
```
cd /usr/share/nginx/html 
unzip /tmp/frontend.zip
```
6. Create Nginx Reverse Proxy Configuration.
```
vim /etc/nginx/default.d/roboshop.conf #Create Nginx Reverse Proxy Configuration.
```

1️⃣ proxy_http_version 1.1;
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
      ```test
      Browser → Nginx → Backend
      (NEW connection every request)
      ```
  - With HTTP/1.1:
    ```test
    Browser → Nginx → Backend
    (One connection reused)
    ```
📌 SRE Impact
- Lower latency, Less CPU, Better scalability under load

2️⃣ location /images/ {} – Static Images
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
🧠 What is a Health Endpoint?
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

🔹 stub_status on;
  - What is stub_status?
  - A built-in Nginx module
  - Shows internal Nginx metrics
  - No backend calls involved
  - sample example
    ```test
    Active connections: 2
    server accepts handled requests
    1200 1200 4500
    Reading: 0 Writing: 1 Waiting: 1
    ```
    📌 Healthy system
      - accepts ≈ handled
    📌  Problem
      - accepts ≠ handled → dropped connections

    
🎯 Who Uses /health?
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

📌 SRE Debugging
    - High Writing → slow backend
    - High Reading → slow clients
    - High Waiting → healthy idle state


What /health DOES check

✅ Nginx process is alive
✅ Nginx can accept connections

What /health DOES NOT check

❌ Backend services
❌ Database health
❌ API correctness

📌 This is a shallow health check

| Endpoint   | Purpose                 |
| ---------- | ----------------------- |
| `/health`  | Is Nginx alive?         |
| `/ready`   | Are backends reachable? |
| `/metrics` | Prometheus metrics      |


access_log off;
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
