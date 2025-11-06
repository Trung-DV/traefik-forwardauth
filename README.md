# Traefik ForwardAuth Examples

This project provides examples of different authentication setups using Traefik's ForwardAuth middleware.

## Overview

Forward authentication is a powerful feature in Traefik that delegates authentication decisions to an external service. This allows for flexible and robust authentication mechanisms without building them into your own services.

All setups in this repository share a common architecture:
- **Traefik** acts as the gateway and reverse proxy.
- **Excalidraw** is used as an example service that requires authentication/SSO before it can be accessed.
- A **third-party service** (like Nginx or Authentik) provides **external authentication**.

This repository includes the following examples:

- [Nginx HTTP Basic Authentication](./nginx)
- Authentik (Coming Soon)

## 1. Nginx HTTP Basic Authentication

This example demonstrates a simple and effective authentication setup using an Nginx service providing HTTP Basic Authentication.

The setup includes:
- **Traefik** as a reverse proxy and load balancer.
- **Nginx** as a lightweight, internal authentication service.
- **Excalidraw** as an example protected service.
- A straightforward authentication flow where Traefik uses Nginx to protect a service with a username and password.

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

For more details on the Nginx example, see the [Nginx README](./nginx/README.md).

## 2. Authentik with GitHub OAuth2

This example demonstrates a comprehensive authentication solution using Authentik, a flexible open-source Identity Provider that supports SSO and social login.

The setup includes:
- **Traefik** as a reverse proxy and load balancer.
- **Authentik** as a feature-rich identity provider supporting SSO and social login (e.g., GitHub, Google, etc.).
- **Excalidraw** as an example service protected by Authentik.
- A sophisticated authentication flow where users can log in using social providers like GitHub, and Traefik uses Authentik's embedded outpost to protect services.

### System Components

```mermaid
graph TB
   subgraph "Client Layer"
       Client[Client/Browser]
   end

   subgraph "External Providers"
       GitHub[GitHub OAuth]
   end
   
   subgraph "Traefik Reverse Proxy"
       Traefik[Traefik]
       subgraph "Middlewares"
           ForwardAuth[authentik Middleware]
       end
       subgraph "Routers"
           ExcalidrawRouter[excalidraw.traefik.authentik.local]
           AuthentikRouter[server.traefik.authentik.local]
           TraefikRouter[traefik.authentik.local]
       end
   end
   
   subgraph "Services"
       subgraph "Identity Provider"
           Authentik[Authentik Service]
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
   Traefik --> AuthentikRouter
   Traefik --> TraefikRouter
   
   %% Router to service mapping
   ExcalidrawRouter --> ForwardAuth
   TraefikRouter --> Dashboard
   
   %% ForwardAuth flow
   ForwardAuth -- "Forwards auth request to" --> Authentik
   Authentik -- "If not logged in, redirects to login page" --> Client
   Authentik -- "Authenticates against" --> GitHub
   ForwardAuth -- "If auth succeeds, forwards original request to" --> Excalidraw
   
   AuthentikRouter --> Authentik
   Client -- "Logs in via" --> Authentik
   
   %% Styling
   classDef client fill:#e1f5fe
   classDef traefik fill:#fff3e0
   classDef service fill:#f3e5f5
   classDef middleware fill:#fff9c4
   classDef external fill:#e8f5e9
   
   class Client client
   class Traefik,ExcalidrawRouter,AuthentikRouter,TraefikRouter traefik
   class Authentik,Excalidraw,Dashboard service
   class ForwardAuth middleware
   class GitHub external
```

For more details on the Authentik example, see the [Authentik README](./authentik/README.md).

