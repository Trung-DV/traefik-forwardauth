# Traefik ForwardAuth with Authentik and GitHub OAuth2

This project demonstrates how to set up a comprehensive authentication solution using Traefik's ForwardAuth middleware with Authentik, a flexible open-source Identity Provider. This guide uses GitHub as an example for OAuth2 authentication.

![](./docs/excalidraw-traefik-forwardauth.webp)

## Overview

The setup includes:
- **Traefik** as a reverse proxy and load balancer.
- **Authentik** as a feature-rich, internal identity provider supporting SSO and social login (e.g., GitHub, Google, etc.).
- **Excalidraw** as an example service protected by Authentik.

This example shows how to protect a service so that users must log in via Authentik (which can use a social login provider like GitHub for authentication) before they can access it.

## Architecture

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
       end
   end
   
   subgraph "Services"
       subgraph "Identity Provider"
           Authentik[Authentik Service]
       end
       
       subgraph "Protected Services"
           Excalidraw[Excalidraw Service]
       end
   end

   %% Client connections
   Client --> Traefik
   
   %% Traefik routing
   Traefik --> ExcalidrawRouter
   Traefik --> AuthentikRouter
   
   %% Router to service mapping
   ExcalidrawRouter --> ForwardAuth
   AuthentikRouter --> Authentik
   
   %% ForwardAuth flow (correct numbered order)
   Authentik -- "2. If not logged in, redirects to login page" --> Client
   Client -- "3. Logs in via" --> Authentik
   Authentik -- "4. Authenticates against" --> GitHub
   ForwardAuth -- "5. If auth succeeds, forwards original request to" --> Excalidraw
   ForwardAuth -- "1. Forwards auth request to" --> Authentik
   
   %% Styling
   classDef client fill:#e1f5fe
   classDef traefik fill:#fff3e0
   classDef service fill:#f3e5f5
   classDef middleware fill:#fff9c4
   classDef external fill:#e8f5e9
   
   class Client client
   class Traefik,ExcalidrawRouter,AuthentikRouter traefik
   class Authentik,Excalidraw service
   class ForwardAuth middleware
   class GitHub external
```

## Quick Start

Authentik is installed using Docker Compose, follow this guide https://docs.goauthentik.io/install-config/install/docker-compose/.

1.  **Generate secrets and create `.env` file:**
    Before starting the services, you need to generate a secret key for `AUTHENTIK_SECRET_KEY` and a password for `PG_PASS`. These are stored in an `.env` file in the `authentik/` directory.

    Run the following commands to generate the secrets and create the `.env` file:
    ```bash
    cd authentik/
    echo "PG_PASS=$(openssl rand -base64 36 | tr -d '\n')" >> .env
    echo "AUTHENTIK_SECRET_KEY=$(openssl rand -base64 60 | tr -d '\n')" >> .env
    ```

2.  **Start the services:**
    ```bash
    docker compose up -d
    ```

![Authentik domain](./docs/traefik.png "Traefik Dashboard")

3.  **Access Authentik:**
    Open `https://server.traefik.authentik.local/if/flow/initial-setup/` and complete the initial setup to create an admin account.

![](./docs/authentik-initial-setup.1.png)
![](./docs/authentik-initial-setup.2.png)

## Configuration: Using GitHub as a Provider

Follow these steps to allow users to log in with their GitHub accounts.

**Guidelines:** [GitHub Configuration - Authentik Docs](https://docs.goauthentik.io/users-sources/sources/social-logins/github/#github-configuration)

### 1. Create a GitHub OAuth App

1.  Go to your GitHub account settings.
2.  Navigate to **Developer settings** > **OAuth Apps** > **New OAuth App**.
3.  Fill in the details:
   -   **Application name**: `LocalAuthentik` (or any name you prefer).
   -   **Homepage URL**: `https://server.traefik.authentik.local`
   -   **Authorization callback URL**: `https://server.traefik.authentik.local/source/oauth/callback/github-authentik`
4.  Click **Register application**.
5.  On the next page, generate a **new client secret**. Copy the **Client ID** and the **Client Secret**.

![](./docs/github-app.png)

### 2. Configure Authentik Source

1.  Log in to Authentik as an admin.
2.  Navigate to **Directory** > **Federation & Social login** > **Create**.
3.  Choose **GitHub OAuth Source** from the list of types.
4.  Fill in the form:
   -   **Name**: `github-authentik`
   -   **Slug**: `github-authentik` *(this slug must match the one used in your GitHub OAuth callback URL)*
   -   **Consumer Key**: Paste the **Client ID** from your GitHub OAuth App.
   -   **Consumer Secret**: Paste the **Client Secret** from your GitHub OAuth App.
5.  Click **Finish**.

<details>

<summary>Screenshots: Configure GitHub as a federated identity provider via OAuth2 </summary>

![](./docs/authentik-github.1.png)
![](./docs/authentik-github.2.png)
![](./docs/authentik-github.3.png)
![](./docs/authentik-github.4.png)
![](./docs/authentik-github.5.png)
![](./docs/authentik-github.6.png)

</details>

### 3. Update the Login Flow

1.  Navigate to **Flows & Stages** > **Stages**.
2.  Select the `default-authentication-identification` stage.
3.  Click the **Stage Bindings** tab.
4.  Click **Edit Stage** of `default-authentication-identification`.
5.  In the **Sources** field, select `github-authentik` and use the arrow button (`>`) to move it to the "Selected sources" box on the right.
6.  Click **Update**.

Now, the Authentik login page will show a "Sign in with GitHub" button.

<details>

<summary>Screenshots: Allow users to authenticate with Authentik using GitHub as an identity provider</summary>

![](./docs/authentik-login-by-github.1.png)
![](./docs/authentik-login-by-github.2.png)
![](./docs/authentik-login-by-github.3.png)
![](./docs/authentik-login-by-github.4.png)
![](./docs/authentik-login-by-github.5.png)
![](./docs/authentik-login-by-github.6.png)
![](./docs/authentik-login-by-github.7.png)
![](./docs/authentik-login-by-github.8.png)
</details>

![](./docs/authentik-login-by-github.9.png)

### 4. Create a Provider and Application

To protect a service with Authentik, you need to create a Provider and link it to an Application. This involves creating a **Provider** (which defines the authentication method) and an **Application** (which represents the service you are protecting).

#### Create an Application with Provider

1.  **Navigate to Applications**:
    -   Go to **Applications** > **Applications** > **Create with Provider**.

2.  **Configure the Application**:
    -   **Name**: `Exca`
    -   **Slug**: `exca`
    -   Click **Next**.

3.  **Choose Provider Type**:
    -   Select **Proxy Provider**.
    -   Click **Next**.

4.  **Configure the Provider**:
    -   **Name**: `Provider for Exca`
    -   **Authorization flow**: `default-provider-authorization-explicit-consent`
    -   **Forward auth (single application)**: Select this mode
    -   **External host**: `https://excalidraw.traefik.authentik.local`
    -   Click **Finish**.

The provider is now ready to use with Authentik's **Embedded Outpost**.

<details>

<summary>Screenshots: Creating Provider and Application</summary>

![](./docs/auth-for-excalidraw.1.png)
![](./docs/auth-for-excalidraw.2.png)
![](./docs/auth-for-excalidraw.3.png)
![](./docs/auth-for-excalidraw.4.png)
![](./docs/auth-for-excalidraw.5.png)
![](./docs/auth-for-excalidraw.6.png)
![](./docs/auth-for-excalidraw.7.png)
![](./docs/auth-for-excalidraw.8.png)

</details>

### 5. Configure Traefik ForwardAuth with Embedded Outpost

Authentik includes an **Embedded Outpost** that runs inside the main `server` container. This outpost handles ForwardAuth requests without requiring a separate outpost container deployment.

To make it aware of your new application, you need to edit the embedded outpost and link it to the `Exca` application you just created.

1.  Navigate to **Applications** > **Outposts**.
2.  Click the **edit icon** of `authentik Embedded Outpost` to edit it.
3.  In the **Applications** field, select the `Exca` application and use the arrow button (`>`) to move it to the "Selected Applications" box on the right.
4.  Click **Update** to save the changes.

<details>

<summary>Screenshots: Config Outpost</summary>


![](./docs/auth-for-excalidraw.9.png)
![](./docs/auth-for-excalidraw.10.png)
![](./docs/auth-for-excalidraw.11.png)
![](./docs/auth-for-excalidraw.12.png)
![](./docs/auth-for-excalidraw.13.png)

</details>

#### Understanding the Embedded Outpost

The Embedded Outpost is automatically available at:
```
http://server:9000/outpost.goauthentik.io/auth/traefik
```

This endpoint validates authentication requests from Traefik and returns authentication headers for authorized users.

#### Configure Traefik Middleware

Define a ForwardAuth middleware in your Traefik configuration to integrate with Authentik:

```yaml
labels:
  - traefik.http.middlewares.fwauth.forwardauth.address=http://server:9000/outpost.goauthentik.io/auth/traefik
  - traefik.http.middlewares.fwauth.forwardauth.authresponseheaders=X-authentik-username,X-authentik-groups,X-authentik-email,X-authentik-name,X-authentik-uid
```

**Key configuration parameters:**
-   `address`: Points to the Embedded Outpost endpoint
-   `authresponseheaders`: Headers that Authentik returns to the protected service (user info, groups, etc.)

#### Apply Middleware to Protected Services

Add the middleware to any service you want to protect. Example for Excalidraw:

```yaml
services:
  excalidraw:
    image: excalidraw/excalidraw
    labels:
      - traefik.enable=true
      - traefik.http.routers.excalidraw.rule=Host(`excalidraw.traefik.authentik.local`)
      - traefik.http.routers.excalidraw.middlewares=fwauth  # Apply authentication middleware
```

#### Test the Authentication Flow

1.  Navigate to `https://excalidraw.traefik.authentik.local`
2.  You will be redirected to the Authentik login page
3.  Log in using GitHub (or any configured authentication method)
4.  After successful authentication, you'll be redirected back to Excalidraw

The Embedded Outpost automatically validates your session and injects authentication headers into requests to the protected service.

## Services

### 1. Traefik (Reverse Proxy)
- **URL**: `https://traefik.authentik.local`
- **Purpose**: The main reverse proxy and dashboard. It manages incoming traffic and coordinates with Authentik for authentication.

### 2. Authentik (Identity Provider)
- **URL**: `https://server.traefik.authentik.local`
- **Purpose**: Provides identity and access management. Users log in here to gain access to protected services.

### 3. Excalidraw (Protected Service)
- **URL**: `https://excalidraw.traefik.authentik.local`
- **Purpose**: An example service that is protected by the `authentik` middleware. Access is only granted after successful authentication via Authentik.