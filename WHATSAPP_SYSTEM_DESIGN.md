# System Design: Real-Time Messaging App (WhatsApp)

This document outlines the architecture for a large-scale, end-to-end encrypted (E2EE) messaging application like WhatsApp. The design prioritizes security, reliability, and low-latency communication.

## 1. Core Functional Requirements
*   **1-on-1 Chat**: Users can send and receive messages in real-time.
*   **Group Chat**: Support for conversations with multiple participants.
*   **Message Status**: Display message status: `sending`, `sent` (to server), `delivered` (to recipient), and `read`.
*   **End-to-End Encryption (E2EE)**: The server must never be able to read message content.
*   **Media Sharing**: Users can send images, videos, and other files.
*   **Offline Support**: Users can view past messages and send new ones even when offline.

## 2. High-Level Architecture

The system uses a combination of WebSockets for real-time communication and a REST API for less frequent actions. The core security is provided by the **Signal Protocol** for E2EE.

### Architectural Diagram
```mermaid
graph TD
    A[Client App] <--> B{API Gateway};
    
    subgraph "Backend Services"
        C[User Service]
        D[Chat Service]
        E[WebSocket Manager]
        F[Media Service]
    end
    
    subgraph "Data & Messaging"
        G["User DB (PostgreSQL)"]
        H["Chat DB (Cassandra)"]
        I["Message Broker (RabbitMQ)"]
        J["Media Storage (S3)"]
    end
    
    B --> C; B --> D; B --> F;
    
    A -- WebSocket Connection --> E;
    
    C -- Manages --> G;
    D -- Manages --> H;
    F -- Manages --> J;
    
    D -- Pub/Sub --> I;
    E -- Pub/Sub --> I;
```

## 3. Core Components & Logic

*   **API Gateway**: Handles standard HTTP requests for actions like user login, profile updates, and fetching contacts.
*   **WebSocket Manager**: Manages persistent, low-latency WebSocket connections for millions of concurrent users. It's responsible for routing real-time messages between clients.
*   **Chat Service**: The main business logic for messaging. It receives messages, determines the recipient(s), and pushes them into a message broker for delivery. It also persists messages to a database.
*   **Media Service**: Handles the storage and retrieval of media files. It generates pre-signed URLs for clients to upload/download files directly to/from S3, offloading bandwidth from the service itself.
*   **Message Broker (RabbitMQ/Kafka)**: Decouples the incoming messages from the WebSocket delivery system. This adds resilience and allows services to scale independently. The `Chat Service` publishes a message, and the `WebSocket Manager` subscribes to relevant topics/queues to deliver it to the correct client.
*   **Databases**:
    *   **Cassandra** is used for the chat history due to its high write throughput and horizontal scalability, which is perfect for time-series data like messages.
    *   **PostgreSQL** is used for relational data like user profiles and contact lists.

## 4. API Design

The API is split between a standard REST API for management tasks and a real-time API over WebSockets for messaging.

### REST API Endpoints (Gateway)

#### 1. User Registration
```http
POST /api/v1/users/register
Content-Type: application/json

Request:
{
  "phone_number": "+14155552671",
  "name": "Jane Doe",
  "public_keys": {
    "identity_key": "...",
    "pre_keys": ["...", "..."]
  }
}

Response (201 Created):
{
  "user_id": "u-a1b2c3d4",
  "status": "Verification required"
}
```

#### 2. Create Group Chat
```http
POST /api/v1/groups
Content-Type: application/json
Authorization: Bearer <user_token>

Request:
{
  "group_name": "Project Team",
  "members": ["u-a1b2c3d4", "u-b2c3d4e5", "u-c3d4e5f6"]
}

Response (201 Created):
{
  "group_id": "g-xyz-789",
  "group_name": "Project Team",
  "members": [],
  "created_at": "2026-01-18T11:00:00Z"
}
```

#### 3. Get Pre-signed URL for Media Upload
```http
POST /api/v1/media/upload
Content-Type: application/json
Authorization: Bearer <user_token>

Request:
{
  "file_name": "image.jpg",
  "file_type": "image/jpeg",
  "is_encrypted": true
}

Response (200 OK):
{
  "media_id": "m-12345",
  "upload_url": "https://whatsapp-media.s3.amazonaws.com/..."
}
```

### Real-time API (WebSockets)

(Events sent and received by the client over a persistent WebSocket connection)

#### Client -> Server: Send Message
```json
{
  "event": "send_message",
  "payload": {
    "recipient_id": "u-b2c3d4e5", 
    "encrypted_content": "..." 
  }
}
```

#### Server -> Client: Receive Message
```json
{
  "event": "receive_message",
  "payload": {
    "sender_id": "u-a1b2c3d4",
    "encrypted_content": "...",
    "timestamp": "2026-01-18T11:05:00Z"
  }
}
```

#### Server -> Client: Message Status Update
```json
{
  "event": "status_update",
  "payload": {
    "message_id": "msg-abc-123",
    "status": "DELIVERED"
  }
}
```

## 5. Database Design

### Cassandra Schema (For Messages)

(Cassandra is chosen for its extreme write throughput and horizontal scalability, making it perfect for storing trillions of messages.)

#### messages_by_chat Table
(A "chat" can be a 1-on-1 chat or a group chat. The `chat_id` could be a composite of the two user IDs for 1-on-1, or the group ID.)
```cql
CREATE TABLE messages_by_chat (
    chat_id UUID,
    message_id TIMEUUID,
    sender_id UUID,
    encrypted_content BLOB,
    PRIMARY KEY (chat_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

### PostgreSQL Schema (For Users and Groups)

#### users Table
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone_number VARCHAR(20) UNIQUE NOT NULL,
    name VARCHAR(100),
    status_text TEXT,
    profile_picture_url TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

#### user_public_keys Table
```sql
CREATE TABLE user_public_keys (
    user_id UUID PRIMARY KEY REFERENCES users(id),
    identity_key TEXT NOT NULL,
    pre_keys LIST<TEXT>,
    last_updated TIMESTAMP WITH TIME ZONE
);
```

#### groups Table
```sql
CREATE TABLE groups (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    icon_url TEXT,
    creator_id UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

#### group_members Table
```sql
CREATE TABLE group_members (
    group_id UUID NOT NULL REFERENCES groups(id),
    user_id UUID NOT NULL REFERENCES users(id),
    role TEXT, -- 'admin' or 'member'
    joined_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (group_id, user_id)
);
```

## 6. End-to-End Encryption (E2EE) Deep Dive

E2EE ensures that only the sender and intended recipient(s) can read the message content.

1.  **Key Generation**: Each user generates a set of public and private keys on their device upon registration. The public keys are uploaded to the server.
2.  **Session Establishment (Signal Protocol)**: When User A wants to message User B for the first time, User A's app requests User B's public keys from the server. It uses these keys to establish a shared secret key for an encrypted session.
3.  **Double Ratchet Algorithm**: For subsequent messages, this algorithm is used to generate a new encryption key for *every single message*. This provides forward secrecy (if a key is compromised, past messages remain secure) and post-compromise security.
4.  **The Server's Role**: The server only sees encrypted ciphertext. Its job is to store and forward these encrypted blobs to the correct recipients. It has no ability to decrypt them.

## 7. Detailed Data Flows

### A. Sending a Message

```mermaid
sequenceDiagram
    participant AliceApp as Alice's App
    participant Server
    participant BobApp as Bob's App

    AliceApp->>AliceApp: Encrypt message for Bob using session key
    AliceApp->>Server: Send(encrypted_blob) via WebSocket
    Server->>Server: Persist encrypted_blob to Cassandra
    
    alt Bob is Online
        Server->>BobApp: Push(encrypted_blob) via WebSocket
    else Bob is Offline
        Server->>BobApp: Send Push Notification (FCM/APNS)
    end
    
    BobApp->>BobApp: Decrypt message with session key
    BobApp->>Server: Acknowledge(message_delivered)
    Server->>AliceApp: Notify(message_delivered)
```

### B. Media Sharing Flow

Media files are also end-to-end encrypted.

1.  **Client-Side Encryption**: The sending client generates a random symmetric key (e.g., AES-256) and encrypts the media file with it.
2.  **Upload Encrypted File**: The client gets a pre-signed URL from the `Media Service` and uploads the *encrypted* file to S3.
3.  **Send Key in a Message**: The client sends a regular chat message to the recipient containing two things: the URL of the encrypted file and the symmetric key used to encrypt it. This message itself is E2E encrypted with the Signal Protocol.
4.  **Download and Decrypt**: The recipient's app receives the message, decrypts it to get the URL and the key, downloads the encrypted file from S3, and then decrypts the file locally for viewing.

## 8. Key Challenges

*   **Scalability of Connections**: Maintaining millions of persistent WebSocket connections is a major engineering challenge. The `WebSocket Manager` needs to be a distributed, load-balanced fleet of servers with connection state often shared via a fast cache like Redis.
*   **E2EE in Group Chats**: E2EE in group chats is more complex. A shared group encryption key is established. When a member is added or removed, this key must be securely renegotiated for all remaining members.
*   **Offline Delivery**: When a user is offline, the server stores the encrypted messages and delivers them as soon as the user reconnects. Push notifications are used to wake the client app if it's in the background.
*   **Data Storage**: Storing trillions of messages requires a a highly scalable, write-optimized database. Cassandra's architecture is an excellent fit for this use case.