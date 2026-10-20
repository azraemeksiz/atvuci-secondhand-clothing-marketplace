# Architecture

Second-hand clothing marketplace built as four independent FastAPI microservices. Each service owns its own PostgreSQL database, and no service accesses another service's database.

## 1. System Overview

Clients send every request through the Nginx API Gateway. Services talk to each other over REST/HTTP. Each service validates the JWT itself using a shared secret key, so none of them calls the User Service to check a token.

```mermaid
flowchart TB
    Client["Client (Web / Mobile)"]
    Gateway["Nginx API Gateway"]

    subgraph Services["FastAPI Microservices"]
        direction LR
        User["User Service<br/>register, login, JWT"]
        Listing["Listing Service<br/>listings: size, price, status"]
        Order["Order Service<br/>purchases, order history"]
        Message["Message Service<br/>live chat"]
    end

    UserDB[("User DB<br/>PostgreSQL")]
    ListingDB[("Listing DB<br/>PostgreSQL")]
    OrderDB[("Order DB<br/>PostgreSQL")]
    MessageDB[("Message DB<br/>PostgreSQL")]

    Client -->|"HTTPS"| Gateway
    Gateway -->|"REST"| User
    Gateway -->|"REST"| Listing
    Gateway -->|"REST"| Order
    Gateway -.->|"WebSocket"| Message

    Order -->|"GET listing: available? price?"| Listing
    Order -->|"PATCH listing: mark as sold"| Listing
    Listing -->|"GET seller name"| User

    User --- UserDB
    Listing --- ListingDB
    Order --- OrderDB
    Message --- MessageDB
```

## 2. Purchase Flow

When a buyer presses "Buy", the Order Service verifies the JWT locally, checks the listing, and either completes the purchase or returns an error.

```mermaid
sequenceDiagram
    actor Buyer
    participant Client
    participant Gateway as API Gateway (Nginx)
    participant Order as Order Service
    participant Listing as Listing Service
    participant OrderDB as Order DB

    Buyer->>Client: Press "Buy"
    Client->>Gateway: POST /orders (JWT, listing_id)
    Gateway->>Order: Forward request
    Order->>Order: Validate JWT with shared secret key
    Order->>Listing: GET /listings/{id}
    Listing-->>Order: Listing (status, price)

    alt Listing is active
        Order->>Listing: PATCH /listings/{id} (status = sold)
        Listing-->>Order: 200 OK
        Order->>OrderDB: Save order
        Order-->>Gateway: 201 Created (order)
        Gateway-->>Client: 201 Created (order)
        Client-->>Buyer: Purchase successful
    else Listing is already sold
        Order-->>Gateway: 409 Conflict (listing already sold)
        Gateway-->>Client: 409 Conflict
        Client-->>Buyer: Listing is no longer available
    end
```
