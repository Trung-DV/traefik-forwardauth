# Traefik ForwardAuth with Wiremock Authentication Service

This project demonstrates a complete authentication and authorization setup using Traefik's ForwardAuth middleware with a Wiremock-based authentication service.

## Overview

The setup includes:
- **Traefik** as a reverse proxy and load balancer
- **Wiremock** as a mock authentication service
- **Echo service** (traefik/whoami) as a protected service demonstrating ForwardAuth
- Complete JWT-based authentication flow with user information forwarding

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
            AuthRouter[auth.traefik.local]
            EchoRouter[echo.traefik.local]
            TraefikRouter[traefik.local]
        end
    end
    
    subgraph "Services"
        subgraph "Authentication"
            Auth[Auth Service<br/>Wiremock:8080]
            subgraph "Endpoints"
                Login[POST /auth/login]
                Validate[GET /auth/validate]
            end
        end
        
        subgraph "Protected Services"
            Echo[Echo Service<br/>traefik/whoami]
        end
        
        subgraph "Management"
            Dashboard[Traefik Dashboard]
        end
    end
    
    subgraph "Storage"
        Mappings[Wiremock Mappings<br/>JSON Files]
    end

    %% Client connections
    Client --> Traefik
    
    %% Traefik routing
    Traefik --> AuthRouter
    Traefik --> EchoRouter
    Traefik --> TraefikRouter
    
    %% Router to service mapping
    AuthRouter --> Auth
    EchoRouter --> ForwardAuth
    TraefikRouter --> Dashboard
    
    %% ForwardAuth flow
    ForwardAuth --> Auth
    ForwardAuth --> Echo
    
    %% Auth service endpoints
    Auth --> Login
    Auth --> Validate
    
    %% Data storage
    Auth --> Mappings
    
    %% Styling
    classDef client fill:#e1f5fe
    classDef traefik fill:#fff3e0
    classDef service fill:#f3e5f5
    classDef storage fill:#e8f5e8
    classDef middleware fill:#fff9c4
    
    class Client client
    class Traefik,AuthRouter,EchoRouter,TraefikRouter traefik
    class Auth,Echo,Dashboard service
    class Mappings storage
    class ForwardAuth middleware
```

### Request Flow Sequence

```mermaid
sequenceDiagram
    participant Client
    participant Traefik as Traefik Proxy
    participant Auth as Auth Service<br/>(Wiremock)
    participant Echo as Protected Service<br/>(Echo)

    Note over Client,Echo: ForwardAuth Flow
    
    Client->>Traefik: 1. Request to protected service<br/>GET https://echo.traefik.local<br/>Authorization: Bearer <token>
    
    Traefik->>Auth: 2. ForwardAuth validation<br/>GET /auth/validate<br/>Authorization: Bearer <token>
    
    alt Valid Token
        Auth->>Traefik: 3a. 200 OK + Auth Headers<br/>X-Auth-User: admin<br/>X-Auth-Email: admin@example.com<br/>X-Auth-Role: admin
        Traefik->>Echo: 4a. Forward request + Auth Headers<br/>GET / + X-Auth-* headers
        Echo->>Traefik: 5a. Service response
        Traefik->>Client: 6a. 200 OK + Response
    else Invalid/Missing Token
        Auth->>Traefik: 3b. 401 Unauthorized
        Traefik->>Client: 4b. 401 Unauthorized
    end

    Note over Client,Echo: Authentication Flow
    
    Client->>Traefik: Login Request<br/>POST https://auth.traefik.local/auth/login
    Traefik->>Auth: Forward login request
    Auth->>Traefik: JWT Token + User Info
    Traefik->>Client: JWT Token + User Info
```

### Flow Explanation

1. **Client Request**: Client sends request to protected service with JWT token
2. **ForwardAuth**: Traefik intercepts and validates token with auth service  
3. **Validation**: Auth service returns user headers if token is valid
4. **Forward**: Traefik forwards original request with authentication context
5. **Response**: Protected service receives request with user information

## Services

### 1. Traefik (Reverse Proxy)
- **URL**: https://traefik.local
- **Purpose**: Main reverse proxy with dashboard
- **Features**: 
  - HTTPS with automatic certificates
  - ForwardAuth middleware configuration
  - Request routing to services

### 2. Auth Service (Wiremock)
- **URL**: https://auth.traefik.local
- **Purpose**: Mock authentication service
- **Endpoints**:
  - `POST /auth/login` - User authentication
  - `GET /auth/validate` - Token validation for ForwardAuth

### 3. Echo Service (Protected)
- **URL**: https://echo.traefik.local
- **Purpose**: Demonstrates ForwardAuth protection
- **Protection**: Requires valid JWT token
- **Features**: Shows forwarded authentication headers

## Quick Start

1. **Start the services:**
   ```bash
   docker compose up -d
   ```

2. **Login to get a token:**
   ```bash
   curl -X POST https://auth.traefik.local/auth/login \
     -H "Content-Type: application/json" \
     -d '{"username":"admin","password":"password"}' \
     -k
   ```

3. **Access protected service:**
   ```bash
   curl -X GET https://echo.traefik.local \
     -H "Authorization: Bearer YOUR_TOKEN_HERE" \
     -k
   ```

## Authentication Flow

### 1. Login Process
- **Endpoint**: `POST /auth/login`
- **Valid Credentials**: `{"username":"admin","password":"password"}`
- **Response**: JWT token with user information

**Example:**
```bash
curl -X POST https://auth.traefik.local/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"password"}' \
  -k
```

**Success Response (200):**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires_in": 3600,
  "token_type": "Bearer",
  "user": {
    "id": "1",
    "username": "admin",
    "email": "admin@example.com",
    "role": "admin"
  }
}
```

**Failed Login (401):**
```json
{
  "error": "Invalid credentials",
  "message": "The provided username or password is incorrect"
}
```

### 2. ForwardAuth Process
When accessing a protected service like `https://echo.traefik.local`:

1. **Traefik** receives the request
2. **ForwardAuth middleware** calls `GET /auth/validate` with the Authorization header
3. **Auth service** validates the token and returns user information in headers
4. **Traefik** forwards the request to the target service with authentication headers
5. **Protected service** receives the request with user context

## ForwardAuth Headers

When authentication is successful, the following headers are forwarded to protected services:

- `X-Auth-User`: Username (e.g., "admin")
- `X-Auth-Email`: User email (e.g., "admin@example.com")
- `X-Auth-Role`: User role (e.g., "admin")

## Testing

### Test Unauthorized Access
```bash
curl -v https://echo.traefik.local -k
# Expected: 401 Unauthorized
```

### Test Authorized Access
```bash
# 1. Get token
TOKEN=$(curl -s -X POST https://auth.traefik.local/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"password"}' \
  -k | grep -o '"token":"[^"]*"' | cut -d'"' -f4)

# 2. Access protected service
curl -X GET https://echo.traefik.local \
  -H "Authorization: Bearer $TOKEN" \
  -k
```

### Practical Examples

Here are the actual curl commands we tested during implementation:

#### 1. Login and Get JWT Token
```bash
# Detailed login request with verbose output
curl -v -X POST https://auth.traefik.local/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"password"}' \
  -k 2>&1

# Expected response:
# HTTP/2 200 
# content-type: application/json
# {"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkFkbWluIFVzZXIiLCJpYXQiOjE1MTYyMzkwMjIsImV4cCI6MTUxNjI0MjYyMn0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c","expires_in":3600,"token_type":"Bearer","user":{"id":"1","username":"admin","email":"admin@example.com","role":"admin"}}
```

#### 2. Test Protected Service Without Token (Blocked)
```bash
# This should return 401 Unauthorized
curl -v -X GET https://echo.traefik.local -k 2>&1

# Expected response:
# HTTP/2 401 
# content-type: application/json
# {"error":"Unauthorized","message":"Valid authorization token required"}
```

#### 3. Test Protected Service With Valid Token (Success)
```bash
# Access protected service with JWT token
curl -v -X GET https://echo.traefik.local \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkFkbWluIFVzZXIiLCJpYXQiOjE1MTYyMzkwMjIsImV4cCI6MTUxNjI0MjYyMn0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c" \
  -k 2>&1

# Expected response will show the forwarded auth headers:
# HTTP/2 200 
# X-Auth-Email: admin@example.com
# X-Auth-Role: admin
# X-Auth-User: admin
# [... echo service response with all headers ...]
```

#### 4. Test Auth Service Validation Endpoint Directly
```bash
# Test the validation endpoint that ForwardAuth uses
curl -X GET https://auth.traefik.local/auth/validate \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkFkbWluIFVzZXIiLCJpYXQiOjE1MTYyMzkwMjIsImV4cCI6MTUxNjI0MjYyMn0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c" \
  -k

# Expected response:
# {"valid":true,"user":{"id":"1","username":"admin","email":"admin@example.com","role":"admin"}}
```

#### 5. Test Failed Login
```bash
# Test with wrong credentials
curl -X POST https://auth.traefik.local/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"wrong","password":"credentials"}' \
  -k

# Expected: Returns 401 with error message
```

#### 6. Complete Authentication Flow Test Script
```bash
#!/bin/bash

echo "=== Testing Complete ForwardAuth Flow ==="

# Step 1: Try accessing protected service without token (should fail)
echo -e "\n1. Testing unauthorized access..."
curl -s -X GET https://echo.traefik.local -k | head -1

# Step 2: Login to get token
echo -e "\n2. Logging in to get JWT token..."
TOKEN=$(curl -s -X POST https://auth.traefik.local/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"password"}' \
  -k | grep -o '"token":"[^"]*"' | cut -d'"' -f4)

echo "Token received: ${TOKEN:0:50}..."

# Step 3: Use token to access protected service
echo -e "\n3. Accessing protected service with token..."
curl -s -X GET https://echo.traefik.local \
  -H "Authorization: Bearer $TOKEN" \
  -k | grep -E "(X-Auth-User|X-Auth-Email|X-Auth-Role)"

echo -e "\n=== Test Complete ==="
```

### Expected Protected Service Response
```
Hostname: 692c5f26b897
IP: 127.0.0.1
IP: ::1
IP: 192.168.148.4
RemoteAddr: 192.168.148.2:41966
GET / HTTP/1.1
Host: echo.traefik.local
User-Agent: curl/8.7.1
Accept: */*
Accept-Encoding: gzip
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkFkbWluIFVzZXIiLCJpYXQiOjE1MTYyMzkwMjIsImV4cCI6MTUxNjI0MjYyMn0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
X-Auth-Email: admin@example.com
X-Auth-Role: admin
X-Auth-User: admin
X-Forwarded-For: 192.168.148.1
X-Forwarded-Host: echo.traefik.local
X-Forwarded-Port: 443
X-Forwarded-Proto: https
X-Forwarded-Server: eefe628cd6e8
X-Real-Ip: 192.168.148.1
```
