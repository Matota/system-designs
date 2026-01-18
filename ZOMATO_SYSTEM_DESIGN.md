# System Design: Online Food Delivery System (Zomato/Uber Eats)

This document outlines the architecture for a three-sided online food delivery marketplace connecting Customers, Restaurants, and Delivery Partners. The system is designed for high concurrency, real-time tracking, and reliability.

## 1. Core Functional Requirements

*   **Customers**: Can browse restaurants, view menus, place orders, make payments, and track their order in real-time.
*   **Restaurants**: Can manage their menu, pricing, and availability. They can accept or reject incoming orders.
*   **Delivery Partners**: Can view and accept available delivery tasks, see pickup and drop-off locations, and track their earnings.

## 2. High-Level Architecture (Microservices)

The system is a classic three-sided marketplace. A microservices architecture is ideal for separating the distinct concerns of each user type and the core order lifecycle.

### Architectural Diagram

```mermaid
graph TD
    subgraph "User-Facing Clients"
        A[Customer App]
        B[Restaurant Dashboard]
        C[Delivery Partner App]
    end

    subgraph Backend
        D[API Gateway]
        
        subgraph "Core Services"
            E[Order Service]
            F[Restaurant Service]
            G[Delivery Partner Service]
            H[Payment Service]
            I[Notification Service]
        end

        subgraph "Data Stores"
            J["PostgreSQL DB"]
            K["Redis (Cache & Geospatial)"]
            L["Elasticsearch (Search)"]
        end
    end

    A --> D; B --> D; C --> D

    D --> E; D --> F; D --> G; D --> H; D --> I

    E -- Manages --> J
    F -- Manages --> J
    G -- Manages --> J
    
    G -- "Uses for Location" --> K
    F -- "Uses for Cache" --> K
    
    F -- "Indexes data in" --> L
```

## 3. Core Services & Responsibilities

*   **API Gateway**: A single entry point that handles requests from all three client types. It manages authentication, routing, and WebSocket connections for real-time updates.
*   **Restaurant Service**: Manages all restaurant-related data, including menus, operating hours, locations, and cuisines. It also provides an interface for restaurants to update their information. Data is indexed in **Elasticsearch** for search and discovery.
*   **Order Service**: The central orchestrator of the entire system. It manages the state machine for an order: `CREATED`, `PAID`, `RESTAURANT_ACCEPTED`, `PREPARING`, `READY_FOR_PICKUP`, `ASSIGNED_TO_PARTNER`, `PICKED_UP`, `DELIVERED`, `CANCELED`.
*   **Delivery Partner Service**: Manages profiles, availability, and real-time location of delivery partners. It uses **Redis** with geospatial indexes to efficiently find available partners near a specific restaurant.
*   **Payment Service**: Integrates with payment gateways like Stripe or Adyen to process customer payments and handle payouts to restaurants and delivery partners.
*   **Notification Service**: A crucial service that sends real-time updates to all parties. It uses multiple channels:
    *   **Push Notifications (FCM/APNS)**: For critical alerts (e.g., "New Order," "Order Delivered").
    *   **WebSockets**: For pushing continuous location updates of the delivery partner to the customer's app.
    *   **SMS/Email**: For receipts and non-urgent communication.

## 4. Detailed Data Flows

### A. Order Placement and Restaurant Confirmation

```mermaid
sequenceDiagram
    participant Customer
    participant Gateway
    participant OrderService
    participant PaymentService
    participant NotificationService
    participant Restaurant

    Customer->>Gateway: Place Order (items, restaurantId)
    Gateway->>OrderService: CreateOrder(data)
    OrderService->>PaymentService: ProcessPayment()
    PaymentService-->>OrderService: Payment Successful
    
    Note over OrderService: Order status: PAID
    
    OrderService->>NotificationService: Notify Restaurant (New Order)
    NotificationService->>Restaurant: Push Notification/WebSocket Message
    
    Restaurant->>Gateway: Accept Order(orderId)
    Gateway->>OrderService: Handle Restaurant Acceptance
    
    Note over OrderService: Order status: RESTAURANT_ACCEPTED
    
    OrderService->>NotificationService: Notify Customer (Order Confirmed)
    NotificationService->>Customer: Push Notification
```

### B. Delivery Partner Assignment & Real-Time Tracking

This flow begins right after the restaurant accepts the order.

```mermaid
sequenceDiagram
    participant OrderService
    participant DeliveryPartnerService as PartnerService
    participant Redis
    participant NotificationService
    participant DeliveryPartner as Partner
    participant Customer

    Note over OrderService: Restaurant is preparing the food. Time to find a driver.
    
    OrderService->>PartnerService: FindPartner(restaurantLocation)
    PartnerService->>Redis: GEOSEARCH partners BYRADIUS [long lat] 10 km
    Redis-->>PartnerService: List of available partners
    PartnerService-->>OrderService: Recommended Partner
    
    OrderService->>NotificationService: Offer Job to Partner
    NotificationService->>Partner: Push Notification (New Delivery Task)
    
    Partner->>OrderService: Accept Job(orderId)
    Note over OrderService: Order Status: ASSIGNED_TO_PARTNER
    
    loop Real-time Tracking
        Partner->>PartnerService: SendLocation(lat, long)
        PartnerService->>Redis: SET partner:location:[partnerId] [lat, long]
        
        Note over Customer: Map screen is open.
        Customer->>PartnerService: Subscribe to location updates via WebSocket
        PartnerService->>Customer: Push location from Redis every few seconds
    end
    
    Partner->>OrderService: Mark as DELIVERED
    Note over OrderService: Order Status: DELIVERED
    OrderService->>NotificationService: Notify Customer (Order Delivered)
```

## 5. Key Challenges & Design Choices

*   **The Three-Way Coordination**: The biggest challenge is orchestrating the workflow between three different actors who all have different states and applications. The `Order Service` acts as the central state machine and source of truth, and the `Notification Service` is the communication backbone that keeps everyone in sync.
*   **Real-Time Location Tracking**: Pushing location data from thousands of delivery partners every few seconds is write-heavy. Using Redis for this is ideal, as it's an in-memory data store designed for extremely fast reads and writes. The customer app can then poll or use a WebSocket to get this location data without hitting the primary database.
*   **Restaurant Availability**: Restaurants can go offline or become too busy to accept new orders. The `Restaurant Service` needs to manage this state, and the customer-facing app must reflect it in real-time to avoid user frustration.
*   **Scalability**: Each service is designed to be independently scalable. If order volume surges, we can scale up the `Order Service` and `Delivery Partner Service` instances without impacting the `Restaurant Service`.
