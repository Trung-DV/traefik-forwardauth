# Traefik ForwardAuth with Nginx HTTP Basic Authentication

This project demonstrates a simple and effective authentication setup using Traefik's ForwardAuth middleware with an Nginx service providing HTTP Basic Authentication.

## Overview

The setup includes:
- **Traefik** as a reverse proxy and load balancer.
- **Nginx** as a lightweight, internal authentication service.
- **Excalidraw** as an example protected service.
- A straightforward authentication flow where Traefik uses Nginx to protect a service with a username and password.

## Architecture

### System Components

```mermaid
graph TB
    subgraph "Client Layer"
        Client[Client/Browser]
    end
    
    subgraph "Traefik Reverse Proxy"
        Traefik[Traefik]
        subgraph "Middlewares"
            ForwardAuth[ForwardAuth Middleware]
        end
        subgraph "Routers"
            ExcalidrawRouter[excalidraw.traefik.local]
            TraefikRouter[traefik.local]
        end
    end
    
    subgraph "Services"
        subgraph "Internal Authentication"
            NginxAuth[Nginx Auth Service]
        end
        
        subgraph "Protected Services"
            Excalidraw[Excalidraw Service]
        end
        
        subgraph "Management"
            Dashboard[Traefik Dashboard]
        end
    end

    %% Client connections
    Client --> Traefik
    
    %% Traefik routing
    Traefik --> ExcalidrawRouter
    Traefik --> TraefikRouter
    
    %% Router to service mapping
    ExcalidrawRouter --> ForwardAuth
    TraefikRouter --> Dashboard
    
    %% ForwardAuth flow
    ForwardAuth -- Forwards auth request to --> NginxAuth
    ForwardAuth -- If auth succeeds, forwards original request to --> Excalidraw
    
    %% Styling
    classDef client fill:#e1f5fe
    classDef traefik fill:#fff3e0
    classDef service fill:#f3e5f5
    classDef middleware fill:#fff9c4
    
    class Client client
    class Traefik,ExcalidrawRouter,TraefikRouter traefik
    class NginxAuth,Excalidraw,Dashboard service
    class ForwardAuth middleware
```

### Request Flow Sequence

```mermaid
sequenceDiagram
    participant Client
    participant Traefik
    participant NginxAuth as Nginx Auth (Internal)
    participant Excalidraw

    Note over Client,Excalidraw: Initial Request (No Credentials)

    Client->>Traefik: 1. GET https://excalidraw.traefik.local
    Traefik->>NginxAuth: 2. Forward auth request
    NginxAuth->>Traefik: 3. 401 Unauthorized
    Traefik->>Client: 4. 401 Unauthorized (triggers browser prompt)

    Note over Client: Browser displays login prompt for "Restricted Area"

    Note over Client,Excalidraw: Authenticated Request

    Client->>Traefik: 5. GET https://excalidraw.traefik.local<br/>(with 'Authorization: Basic ...' header)
    Traefik->>NginxAuth: 6. Forward auth request (with credentials)
    NginxAuth->>Traefik: 7. 204 No Content<br/>(with 'X-Forwarded-User: admin' header)
    Traefik->>Excalidraw: 8. Forward original request<br/>(with 'X-Forwarded-User' header)
    Excalidraw->>Traefik: 9. Service response
    Traefik->>Client: 10. 200 OK + Response
```

### Flow Explanation

1.  **Initial Request**: The client makes a request to the protected service (`excalidraw.traefik.local`).
2.  **ForwardAuth**: Traefik intercepts the request and, because of the `nginx-auth` middleware, sends an authentication request to the internal `nginx-auth` service.
3.  **Authentication Challenge**: Since the initial request has no credentials, the Nginx service returns a `401 Unauthorized` response. Traefik forwards this to the client, causing the browser to display a username/password prompt.
4.  **Authenticated Request**: The user enters their credentials (`admin`/`admin`). The browser sends the request again, this time with an `Authorization: Basic <credentials>` header.
5.  **Validation**: Traefik again forwards the auth request to Nginx. Nginx validates the credentials against its configured `.htpasswd` file.
6.  **Success and Forward**: Upon successful validation, Nginx returns a `204 No Content` status and adds the `X-Forwarded-User` header. Traefik receives this successful response and forwards the original client request to the `excalidraw` service, along with the `X-Forwarded-User` header.
7.  **Final Response**: The `excalidraw` service responds, and Traefik passes the response back to the client.

## User Experience

1. **Login Required**: When first accessing the site, the browser will prompt for credentials.
![Login Prompt](docs/require-login.png)

2. **Authentication**: Enter the username and password (`admin`/`admin`).
![Login](docs/login.png)

3. **Access Granted**: After successful authentication, the user can access the protected Excalidraw service.
![Logged In](docs/logged-in.png)

## Services

### 1. Traefik (Reverse Proxy)
- **URL**: `https://traefik.local`
- **Purpose**: The main reverse proxy and dashboard. It manages incoming traffic and coordinates with the authentication service.

### 2. Nginx Auth Service
- **Purpose**: Provides HTTP Basic Authentication internally for Traefik. It has no public URL. It validates credentials and returns a success or failure status to Traefik.
- **Credentials**: The user is `admin` with password `admin`, as configured in `compose.yml`.

### 3. Excalidraw Service (Protected)
- **URL**: `https://excalidraw.traefik.local`
- **Purpose**: An example service that is protected by the `nginx-auth` middleware. Access is only granted after successful authentication.

## Quick Start

1.  **Start the services:**
    ```bash
    docker compose up -d
    ```

2.  **Access the protected service:**
    Open your browser and navigate to `https://excalidraw.traefik.local`. You will be prompted for a username and password.

    Alternatively, use `curl`:
    ```bash
    # This will fail with a 401
    curl https://excalidraw.traefik.local -k

    # This will succeed
    curl -u admin:admin https://excalidraw.traefik.local -k
    ```
    The `-u admin:admin` flag tells curl to use HTTP Basic Authentication.
