
### Run MySQL in Docker alongside your app
### Step 1: Create a shared Docker network

Containers can't talk to each other by default unless they are on the same virtual network.
```
docker network create app-network
```
### Step 2: Spin up a MySQL container

```
docker run -d --name mysql-db --network app-network -e MYSQL_DATABASE=order_db -e MYSQL_ROOT_PASSWORD=root -p 3307:3306 mysql:8.0
```

#### Running in Docker with Environment Variables (`-e`)
**Spring Property Placeholders with Default Values**. It gives you the best of both worlds: local execution out of the box and flexible Docker overrides without changing a single line of code.
`jdbc:mysql://${DB_HOST:localhost}:${DB_PORT:3306}/${DB_NAME:order_db}?useSSL=false`
-   **`${DB_HOST:localhost}`**: Spring looks for an environment variable named `DB_HOST`
    -   If it **finds** `DB_HOST`, it uses that value (e.g., `mysql-db` or `host.docker.internal`).
    -   If it **doesn't find** `DB_HOST` (like when you run locally in IntelliJ/Eclipse), it gracefully falls back to `localhost`.
-   **`${DB_NAME:order_db}`**: Works the exact same way—uses `DB_NAME` if passed, otherwise falls back to `order_db`.

Example:
```
spring:
  datasource:
    url: jdbc:mysql://${DB_HOST:localhost}:${DB_PORT:3306}/${DB_NAME:order_db}?useSSL=false&allowPublicKeyRetrieval=true
    username: ${DB_USER:root}
    password: ${DB_PASS:root}
    driver-class-name: com.mysql.cj.jdbc.Driver
```

#### Option 1: Your MySQL is running inside a Docker Container
#### How the Connection Works

```
[ Windows Host ] ─── (Connects via localhost:3307) ───┐
                                                     ▼
┌─ Docker Network: app-network ───────────────────────────────┐
│                                                             │
│  [ order-service ] ── (Connects via mysql-db:3306) ──► [ mysql-db ]
│                                                             │
└─────────────────────────────────────────────────────────────┘

```

Because `order-service` and `mysql-db` are on the **same Docker network** (`app-network`), `order-service` talks to `mysql-db` **directly inside Docker**. It doesn't step outside to your Windows laptop, so it doesn't touch port `3307`.
```
docker run -d -p 8083:8080 ^
  --name spring-order-service ^
  --network app-network ^
  -e DB_HOST=mysql-db ^
  -e DB_NAME=order_db ^
  -e DB_PORT=3306 ^
  vamshivoore/spring-order-service:latest

  docker run -d -p 8085:8080 ^
  --name springboot-crud-k8s ^
  --network app-network ^
  -e DB_HOST=mysql-db ^
  -e DB_NAME=vmc_order ^
  -e DB_PORT=3306 ^
  vamshivoore/springboot-crud-k8s:latest
```

-   **Use this if:** You ran `docker run --name mysql-db ...` earlier to spin up MySQL as a container.
-   **Why it looks like this:**
    -   `--network app-network` connects your app container directly to your MySQL container.
    -   `DB_HOST=mysql-db` tells Spring Boot to talk to the MySQL container by its name.
    -   `DB_PORT=3306` uses MySQL's internal container port (not 3307!).
        

### Option 2: Your MySQL is installed directly on your Windows Lapt

```
docker run -d -p 8083:8080 ^
  --name order-service ^
  -e DB_HOST=host.docker.internal ^
  vamshivoore/spring-order-service:latest
```

-   **Use this if:** You are **NOT** using a MySQL Docker container, but instead using the MySQL Server installed natively on your Windows OS (e.g., via MySQL Installer or XAMPP).
    
-   **Why it looks like this:**
    
    -   `host.docker.internal` is a special Docker DNS name that lets a container reach out to your physical laptop's `localhost:3306`.
        
    -   You don't need `--network app-network` because MySQL isn't on a Docker network.


#### Quick Checklist of What This Setup Achieves:

-   **No Local Port Conflict:** Map `-p 3307:3306` so your laptop's existing MySQL stays on `3306` while this container uses `3307`.
-   **Spring Boot Container Connection:** When running `order-service` inside Docker on `app-network`, use:
    -   **`DB_HOST=mysql-db`** 
    -   **`DB_PORT=3306`** (containers talk to each other directly via internal port `3306`).
-   **IDE / Local Connection:** If you run Spring Boot or MySQL Workbench directly on your laptop, point it to **`localhost:3307`**.
`useSSL=false` tells the MySQL JDBC driver to connect to the database **without using Transport Layer Security (TLS/SSL) encryption**.

### What it does in plain English

-   **When set to `false`:** Data sent between your Spring Boot application and MySQL (including queries, table data, and credentials) is sent in **unencrypted plain text**.
    
-   **When set to `true`:** All communication is encrypted using an SSL/TLS certificate, protecting it from being intercepted on the network.
    

### Why we use `useSSL=false` in Docker / Local Dev

1.  **Avoids Setup Complexity:** Local development databases and test containers (like your MySQL Docker image) don't have SSL certificates installed by default. Turning SSL off prevents connection errors like `SSLHandshakeException` or `javax.net.ssl.SSLException`.
    
2.  **Speed & Simplicity:** On your local laptop or inside a private Docker bridge network (`app-network`), traffic never leaves your machine, so encryption overhead isn't necessary during testing.
    

### Production vs. Local Standard

| **Environment** | **Setting** | **Why?** |
|---|---|---|
| **Local / Docker Sandbox** | `useSSL=false` | Prevents SSL certificate errors and simplifies local development and testing. |
| **Production (AWS RDS, Cloud Databases)** | `useSSL=true` | Encrypts database communication and protects sensitive data over public or cloud networks. |


** allowPublicKeyRetrieval=true **
if you are connecting to **MySQL 8.0+** using default password authentication (like `caching_sha2_password`).

Here is why:

-   **What it does:** MySQL 8.0+ encrypts passwords using RSA keys for security. When a client (like Spring Boot's JDBC driver) connects without SSL (`useSSL=false`), MySQL requires the client to request the server's public key to safely encrypt the password.
    
-   **Why you get errors without it:** By default, the Java driver disables public key retrieval to protect against man-in-the-middle attacks. If you don't explicitly set `allowPublicKeyRetrieval=true` during non-SSL connections, Spring Boot will throw:
    
    > `java.sql.SQLException: Public Key Retrieval is not allowed`
 
```
url: jdbc:mysql://${DB_HOST:localhost}:${DB_PORT:3306}/${DB_NAME:order_db}?useSSL=false&allowPublicKeyRetrieval=true
```

** Note: **  Adding `?createDatabaseIfNotExist=true` ensures MySQL automatically creates `order_db` if it hasn't been created yet.
```
docker run -d -p 8083:8080 ^
  --name spring-order-service ^
  --network app-network ^
  -e SPRING_DATASOURCE_URL=jdbc:mysql://mysql-db:3306/order_db?createDatabaseIfNotExist=true ^
  vamshivoore/spring-order-service:latest

  docker run -d -p 8085:8000 ^
  --name springboot-crud-k8s ^
  --network app-network ^
  -e DB_HOST=mysql-db ^
  -e DB_PORT=3306 ^
  -e DB_NAME=vmc_order ^
  -e DB_USERNAME=root ^
  -e DB_PASSWORD=root ^
  vamshivoore/springboot-crud-k8s:latest
```
