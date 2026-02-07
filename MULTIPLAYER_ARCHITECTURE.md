# SeriousProton Multiplayer Architecture

This document describes the complete multiplayer architecture of SeriousProton, providing all the information needed to create a custom client that can communicate with a SeriousProton server.

## Table of Contents

1. [Overview](#overview)
2. [Network Protocol](#network-protocol)
3. [Connection and Authentication](#connection-and-authentication)
4. [Object Replication](#object-replication)
5. [Command System](#command-system)
6. [Proxy Architecture](#proxy-architecture)
7. [ECS Entity Replication](#ecs-entity-replication)
8. [Audio Streaming](#audio-streaming)
9. [Master Server Protocol](#master-server-protocol)
10. [Example Implementation](#example-implementation)

## Overview

SeriousProton uses a **client-server architecture** with the following key features:

- **TCP-based communication** for reliable data transfer
- **Command-based packet protocol** with binary serialization
- **Object replication** with delta compression (only changed values are sent)
- **Optional proxy servers** for geographically distributed clients
- **Master server registry** for server discovery
- **Steam P2P support** (optional, requires Steam SDK)

### Key Components

- **GameServer**: The authoritative server that manages game state
- **GameClient**: Client that connects to the server
- **GameServerProxy**: Optional relay server for distributed clients
- **MultiplayerObject**: Base class for replicated game objects
- **Master Server**: PHP-based server registry (optional)

### Network Topology

```
[Client 1] ──┐
[Client 2] ──┼─── [GameServer] ─── [Master Server]
[Client 3] ──┘

OR with proxies:

[Client A] ──┐                    
[Client B] ──┼─ [Proxy 1] ──┐
             │              │
[Client C] ──┤              ├─── [GameServer] ─── [Master Server]
[Client D] ──┘              │
                     [Proxy 2] ── [Client E]
                                  [Client F]
```

## Network Protocol

### Packet Structure

All packets use the following structure:

```
[Command (2 bytes)] [Payload (variable length)]
```

- **Command**: A 16-bit unsigned integer (`command_t`) identifying the packet type
- **Payload**: Binary data serialized using `sp::io::DataBuffer`

### Data Serialization

SeriousProton uses `sp::io::DataBuffer` for binary serialization with **Variable-Length Quantity (VLQ)** encoding for integers to save bandwidth:

#### Supported Data Types

| Type | Encoding |
|------|----------|
| `bool` | 1 byte (0 or 1) |
| `int8_t`, `uint8_t` | 1 byte raw |
| `int16_t`, `int32_t` | VLQ signed (zigzag encoding) |
| `uint16_t`, `uint32_t` | VLQ unsigned |
| `uint64_t` | VLQ unsigned 64-bit |
| `float` | 4 bytes (IEEE 754) |
| `double` | 8 bytes (IEEE 754) |
| `string` | VLQ length + UTF-8 bytes |
| `enum` | As `uint16_t` |
| `glm::vec2`, `glm::vec3` | Component-wise serialization |

#### VLQ Encoding

**Unsigned VLQ** (`writeVLQu`):
- Uses 7 bits per byte for data
- Bit 7 (0x80) indicates continuation
- Most significant bytes first
- Example: `127` → `0x7F`, `128` → `0x81 0x00`

**Signed VLQ** (`writeVLQs`):
- Zigzag encoding: `(v < 0) ? (-v << 1) | 1 : (v << 1)`
- Then encoded as unsigned VLQ
- Example: `-1` → `0x01`, `1` → `0x02`

### Command Reference

All commands are defined in `src/multiplayer_internal.h`:

#### Object Management Commands

| Command | Value | Direction | Description |
|---------|-------|-----------|-------------|
| `CMD_CREATE` | 0x0001 | Server → Client | Create a new replicated object |
| `CMD_UPDATE_VALUE` | 0x0002 | Server → Client | Update specific member values of an object |
| `CMD_DELETE` | 0x0003 | Server → Client | Delete a replicated object |

#### Connection Commands

| Command | Value | Direction | Description |
|---------|-------|-----------|-------------|
| `CMD_SET_CLIENT_ID` | 0x0004 | Server → Client | Assign client ID after successful auth |
| `CMD_REQUEST_AUTH` | 0x0009 | Server → Client | Request authentication (version + password) |
| `CMD_CLIENT_SEND_AUTH` | 0x0010 | Client → Server | Send authentication credentials |

#### Game State Commands

| Command | Value | Direction | Description |
|---------|-------|-----------|-------------|
| `CMD_SET_GAME_SPEED` | 0x0005 | Server → Client | Set game simulation speed multiplier |

#### Command Execution

| Command | Value | Direction | Description |
|---------|-------|-----------|-------------|
| `CMD_CLIENT_COMMAND` | 0x0006 | Client → Server | Client sends command to specific object |
| `CMD_SERVER_COMMAND` | 0x0011 | Server → Client | Server broadcasts command from object |

#### Keepalive

| Command | Value | Direction | Description |
|---------|-------|-----------|-------------|
| `CMD_ALIVE` | 0x0007 | Client → Server | Heartbeat keepalive ping |
| `CMD_ALIVE_RESP` | 0x0012 | Server → Client | Heartbeat response for RTT calculation |

#### Proxy Commands

| Command | Value | Direction | Description |
|---------|-------|-----------|-------------|
| `CMD_NEW_PROXY_CLIENT` | 0x000a | Proxy → Server | Notify server of new client connecting through proxy |
| `CMD_SET_PROXY_CLIENT_ID` | 0x000b | Server → Proxy | Assign client ID to proxy-connected client |
| `CMD_DEL_PROXY_CLIENT` | 0x000c | Proxy → Server | Notify server of client disconnecting from proxy |
| `CMD_PROXY_CLIENT_COMMAND` | 0x000d | Proxy → Server | Forward client command through proxy |
| `CMD_PROXY_TO_CLIENTS` | 0x000e | Server → Proxy | Indicate which proxy clients should receive data |
| `CMD_SERVER_CONNECT_TO_PROXY` | 0x000f | Server → Proxy | Server initiating proxy connection |

#### Audio Commands

| Command | Value | Direction | Description |
|---------|-------|-----------|-------------|
| `CMD_AUDIO_COMM_START` | 0x0020 | Client ↔ Server | Start audio stream |
| `CMD_AUDIO_COMM_DATA` | 0x0021 | Client ↔ Server | Audio data packet (Opus encoded) |
| `CMD_AUDIO_COMM_STOP` | 0x0022 | Client ↔ Server | Stop audio stream |

#### ECS Commands

| Command | Value | Direction | Description |
|---------|-------|-----------|-------------|
| `CMD_ECS_UPDATE` | 0x0030 | Server → Client | ECS entity/component update |

**ECS Sub-commands** (byte following `CMD_ECS_UPDATE`):

| Sub-command | Value | Description |
|-------------|-------|-------------|
| `CMD_ECS_ENTITY_CREATE` | 0x00 | Create a new ECS entity |
| `CMD_ECS_ENTITY_DESTROY` | 0x01 | Destroy an ECS entity |
| `CMD_ECS_SET_COMPONENT` | 0x02 | Add or update a component |
| `CMD_ECS_DEL_COMPONENT` | 0x03 | Remove a component |

## Connection and Authentication

### Connection Flow

#### 1. Client Initiates Connection

```cpp
// Client creates TCP socket connection
socket = new TcpSocket();
socket->connect(server_address, port);
socket->setBlocking(false);
```

**State**: `Connecting`

#### 2. Server Accepts Connection

```cpp
// Server accepts incoming connection
ClientInfo info;
info.socket = listen_socket.accept();
info.receive_state = CRS_Auth;
```

#### 3. Server Sends Authentication Request

```
Packet: CMD_REQUEST_AUTH
Payload:
  - version_number (int32_t): Server version number
  - password_required (bool): Whether password is needed
```

**Client State**: `Authenticating`

#### 4. Client Sends Authentication

```
Packet: CMD_CLIENT_SEND_AUTH
Payload:
  - version_number (int32_t): Client version number
  - password (string): Password (empty if no password)
```

#### 5. Server Validates

Server checks:
- Version number matches
- Password is correct (if required)

**Success**:
```
Packet: CMD_SET_CLIENT_ID
Payload:
  - client_id (int32_t): Assigned client ID
```

**Failure**:
- Server closes connection
- Client receives disconnect with appropriate reason

**Client State**: `Connected`

#### 6. Server Replicates Initial State

After authentication, server sends:
1. `CMD_CREATE` packets for all existing objects
2. `CMD_ECS_UPDATE` packets for all ECS entities
3. `CMD_SET_GAME_SPEED` if game speed is not 1.0

### Disconnection

**Client-initiated**:
- Client closes socket
- No explicit disconnect packet

**Server-initiated**:
- Server closes socket
- Client detects via timeout or socket error

**Disconnect Reasons**:
```cpp
enum class DisconnectReason : uint8_t
{
    None = 0,
    FailedToConnect,      // TCP connection failed
    VersionMismatch,      // Version numbers don't match
    BadCredentials,       // Wrong password
    TimedOut,             // No data for 20 seconds
    ClosedByServer,       // Normal server shutdown
    Unknown
};
```

### Heartbeat / Keepalive

To maintain the connection and measure latency:

**Client sends** (every 0.5 seconds if no other data sent):
```
Packet: CMD_ALIVE
Payload: (empty)
```

**Server responds**:
```
Packet: CMD_ALIVE_RESP
Payload: (empty)
```

**Timeout**: If no data received for **20 seconds**, connection is considered dead.

## Object Replication

### Multiplayer Object System

Objects derive from `MultiplayerObject` and register members for replication:

```cpp
class MyObject : public MultiplayerObject
{
public:
    int health;
    float position_x;
    float position_y;
    string name;
    
    MyObject() : MultiplayerObject("MyObject")
    {
        registerMemberReplication(&health);
        registerMemberReplication(&position_x);
        registerMemberReplication(&position_y);
        registerMemberReplication(&name);
    }
};

REGISTER_MULTIPLAYER_CLASS(MyObject, "MyObject");
```

### Object Creation (CMD_CREATE)

When a new object is created on the server:

```
Packet: CMD_CREATE
Payload:
  - object_id (int32_t): Unique object identifier
  - class_name (string): Class identifier (e.g., "MyObject")
  - member_count (uint16_t): Number of replicated members
  
  For each member:
    - member_index (uint16_t): Index in registration order
    - member_value: Serialized value (type-specific)
```

**Example packet for MyObject**:
```
CMD_CREATE (0x0001)
object_id: 42
class_name: "MyObject"
member_count: 4
  [0]: 100        (health)
  [1]: 10.5       (position_x)
  [2]: 20.3       (position_y)
  [3]: "Player1"  (name)
```

**Client handling**:
1. Look up class name in registry
2. Create instance using registered factory function
3. Read all member values
4. Store object with ID in object map

### Object Updates (CMD_UPDATE_VALUE)

Only **changed** members are sent:

```
Packet: CMD_UPDATE_VALUE
Payload:
  - object_id (int32_t): Object to update
  - update_count (uint16_t): Number of members being updated
  
  For each update:
    - member_index (uint16_t): Index of member
    - member_value: New serialized value
```

**Example** (only health changed):
```
CMD_UPDATE_VALUE (0x0002)
object_id: 42
update_count: 1
  [0]: 75  (health decreased from 100 to 75)
```

### Change Detection

The server automatically tracks changes:

```cpp
struct MemberReplicationInfo {
    void* ptr;                    // Pointer to member variable
    uint64_t prev_data;          // Previous value
    float update_delay;          // Throttle: min time between updates
    float update_timeout;        // Time until next allowed update
    
    bool(*isChangedFunction)(void* data, void* prev_data_ptr);
    void(*sendFunction)(void* data, DataBuffer& packet);
    void(*receiveFunction)(void* data, DataBuffer& packet);
};
```

**Update cycle** (server):
1. Check each object's registered members
2. Compare current value with `prev_data`
3. If changed AND `update_timeout <= 0`:
   - Add to update packet
   - Update `prev_data`
   - Reset `update_timeout = update_delay`
4. Send `CMD_UPDATE_VALUE` if any changes

### Update Delays

Members can have **update delays** to throttle bandwidth:

```cpp
registerMemberReplication(&position_x, 0.1f);  // Max 10 updates/sec
registerMemberReplication(&health);             // No delay (instant)
```

### Object Deletion (CMD_DELETE)

When object is destroyed on server:

```
Packet: CMD_DELETE
Payload:
  - object_id (int32_t): Object to delete
```

**Client handling**:
1. Look up object by ID
2. Call destructor / cleanup
3. Remove from object map

### Vector Replication

Vectors are replicated as a whole when changed:

```
Packet: CMD_UPDATE_VALUE
Payload:
  object_id: 42
  update_count: 1
    member_index: 5
    vector_count (uint16_t): Number of elements
    element[0]: value
    element[1]: value
    ...
```

## Command System

### Client Commands (Client → Server)

Clients can send custom commands to specific objects:

**Client code**:
```cpp
sp::io::DataBuffer packet;
packet << some_data;
my_object->sendClientCommand(packet);
```

**Protocol**:
```
Packet: CMD_CLIENT_COMMAND
Payload:
  - object_id (int32_t): Target object
  - command_data: Custom payload
```

**Server handling**:
```cpp
void MyObject::onReceiveClientCommand(int32_t client_id, sp::io::DataBuffer& packet)
{
    // Read custom data
    // Execute command
    // Optionally broadcast result to all clients
}
```

### Server Commands (Server → Clients)

Server can broadcast commands from objects to all clients:

**Server code**:
```cpp
sp::io::DataBuffer packet;
packet << result_data;
my_object->broadcastServerCommand(packet);
```

**Protocol**:
```
Packet: CMD_SERVER_COMMAND
Payload:
  - object_id (int32_t): Source object
  - command_data: Custom payload
```

**Client handling**:
```cpp
void MyObject::onReceiveServerCommand(sp::io::DataBuffer& packet)
{
    // Read custom data
    // Update client-side state
}
```

### Typical Command Flow

1. **User Input** → Client sends `CMD_CLIENT_COMMAND` to server
2. **Server Validates** → Processes command in `onReceiveClientCommand`
3. **Server Updates State** → Modifies object members (automatically replicated)
4. **Server Broadcasts** → Optionally sends `CMD_SERVER_COMMAND` for immediate feedback
5. **Clients Update** → Receive state via `CMD_UPDATE_VALUE` and/or `CMD_SERVER_COMMAND`

## Proxy Architecture

Proxies act as relay servers for geographically distributed clients, reducing latency to a nearby proxy rather than to the main server.

### Proxy Connection

#### Proxy → Server

1. **Proxy initiates** TCP connection to server
2. **Proxy authenticates** as a regular client
3. **Proxy listens** for local client connections on its own port

#### Client → Proxy

1. **Client connects** to proxy (same protocol as connecting to server)
2. **Proxy forwards** authentication to server
3. **Server assigns** client ID via `CMD_SET_PROXY_CLIENT_ID`
4. **Proxy maintains** mapping of client IDs

### Proxy Command Forwarding

#### Client Command → Server

```
Client → Proxy:
  CMD_CLIENT_COMMAND
  - object_id
  - command_data

Proxy → Server:
  CMD_PROXY_CLIENT_COMMAND
  - object_id
  - client_id (from proxy)
  - command_data
```

#### Server State → Clients

```
Server → Proxy:
  CMD_PROXY_TO_CLIENTS
  - client_id_count (uint16_t)
  - client_id[0]
  - client_id[1]
  - ...
  - actual_packet (CMD_CREATE/UPDATE_VALUE/DELETE/SERVER_COMMAND)

Proxy → Clients:
  Unpacks and sends actual_packet to specified clients
```

### Proxy Client Lifecycle

**New Client**:
```
Proxy → Server:
  CMD_NEW_PROXY_CLIENT
  - temp_client_id (int32_t): Temporary ID used by proxy

Server → Proxy:
  CMD_SET_PROXY_CLIENT_ID
  - temp_client_id
  - actual_client_id (int32_t): Real ID assigned by server
```

**Disconnecting Client**:
```
Proxy → Server:
  CMD_DEL_PROXY_CLIENT
  - client_id (int32_t): ID of disconnected client
```

### Proxy State Replication

- Proxies maintain **full game state** (all objects, all members)
- When a client connects, proxy sends full state from its cache
- Reduces load on main server for initial synchronization

## ECS Entity Replication

SeriousProton supports Entity-Component-System (ECS) replication:

### Entity Creation

```
Packet: CMD_ECS_UPDATE
Payload:
  - sub_command (uint8_t): CMD_ECS_ENTITY_CREATE (0x00)
  - entity_index (uint32_t): Server-side entity index
  - entity_version (uint32_t): Entity version number
```

**Client handling**:
1. Create new local ECS entity
2. Attach `ServerIndex` component with server's index/version
3. Store in `entity_mapping` array for lookup

### Entity Destruction

```
Packet: CMD_ECS_UPDATE
Payload:
  - sub_command (uint8_t): CMD_ECS_ENTITY_DESTROY (0x01)
  - entity_index (uint32_t): Server-side entity index
```

**Client handling**:
1. Look up entity by server index
2. Destroy local entity
3. Clear mapping

### Component Set

```
Packet: CMD_ECS_UPDATE
Payload:
  - sub_command (uint8_t): CMD_ECS_SET_COMPONENT (0x02)
  - entity_index (uint32_t): Target entity
  - component_type_id (uint32_t): Component type identifier
  - component_data: Serialized component (type-specific)
```

**Client handling**:
1. Look up entity by server index
2. Add or update component
3. Deserialize component data

### Component Deletion

```
Packet: CMD_ECS_UPDATE
Payload:
  - sub_command (uint8_t): CMD_ECS_DEL_COMPONENT (0x03)
  - entity_index (uint32_t): Target entity
  - component_type_id (uint32_t): Component type to remove
```

### Entity References

ECS entities can be serialized in DataBuffer:

```cpp
sp::io::DataBuffer& operator<<(DataBuffer& packet, const sp::ecs::Entity& e);
sp::io::DataBuffer& operator>>(DataBuffer& packet, sp::ecs::Entity& e);
```

**Server** sends entity index; **Client** resolves via `entity_mapping`.

## Audio Streaming

SeriousProton supports real-time voice communication using Opus codec:

### Starting Audio Stream

**Client**:
```
Packet: CMD_AUDIO_COMM_START
Payload:
  - target_identifier (int32_t): Target for audio (e.g., team ID, channel)
```

**Server** validates via `onVoiceChat(client_id, target_identifier)` callback:
- Returns `std::unordered_set<int32_t>` of client IDs that should receive audio
- Can implement proximity chat, team chat, etc.

### Audio Data Packets

**Client**:
```
Packet: CMD_AUDIO_COMM_DATA
Payload:
  - opus_packet (variable): Encoded audio frame
```

**Server**:
- Receives audio data
- Forwards to target clients determined by `onVoiceChat`

**Target Clients**:
```
Packet: CMD_AUDIO_COMM_DATA
Payload:
  - source_client_id (int32_t): Who is speaking
  - opus_packet (variable): Encoded audio frame
```

### Stopping Audio Stream

**Client**:
```
Packet: CMD_AUDIO_COMM_STOP
Payload: (empty)
```

**Server** notifies target clients:
```
Packet: CMD_AUDIO_COMM_STOP
Payload:
  - source_client_id (int32_t): Who stopped speaking
```

### Audio Implementation Notes

- Audio is **Opus encoded** (low-latency, voice-optimized)
- Sample rate: 48 kHz
- Frame size: Typically 20ms (960 samples)
- Bitrate: Adjustable (typically 16-24 kbps for voice)

## Master Server Protocol

The master server is a **PHP-based HTTP service** for server registration and discovery.

### Server Registration

Servers periodically POST to `/register.php` to stay listed:

**Request**:
```http
POST /register.php HTTP/1.1
Content-Type: application/x-www-form-urlencoded

port=35666&name=My%20Server&version=20240101
```

**Parameters**:
- `port` (required): Server's listen port
- `name` (required): Human-readable server name
- `version` (required): Server version number

**Server IP** is detected from HTTP request.

**Response**:
- `OK` - Registration successful
- `CONNECT FAILED: <ip>:<port>` - Master server couldn't reach game server

**Verification**:
Master server performs a **TCP connection test** to verify the server is reachable before listing.

### Registration Frequency

Servers must register **every 5 minutes** to stay listed (or more frequently).

### Server Discovery

Clients GET from `/list.php`:

**Request**:
```http
GET /list.php HTTP/1.1
```

**Response**:
```
127.0.0.1:35666:20240101:My Server
192.168.1.100:35666:20240101:Another Server
```

**Format** (colon-delimited):
```
<ip>:<port>:<version>:<name>
```

One server per line.

**Empty list**: No response body.

### Master Server Database

Master server uses **MySQL** to store:
- Server IP address
- Port
- Version
- Name
- Last update timestamp

Servers not updated in **5 minutes** are automatically removed from listings.

## Example Implementation

### Minimal Client Implementation

Here's a minimal example of connecting to a SeriousProton server:

```cpp
#include <iostream>
#include "io/network/tcpSocket.h"
#include "io/dataBuffer.h"
#include "multiplayer_internal.h"

class MinimalClient
{
    sp::io::network::TcpSocket socket;
    int32_t client_id = -1;
    int version_number;
    
public:
    MinimalClient(int version) : version_number(version) {}
    
    bool connect(const std::string& host, int port)
    {
        if (!socket.connect(sp::io::network::Address(host), port))
            return false;
        socket.setBlocking(false);
        return true;
    }
    
    void update()
    {
        // Receive packets
        sp::io::DataBuffer receive_buffer;
        std::vector<uint8_t> temp_buffer(1024);
        
        while (true)
        {
            size_t received = socket.receive(temp_buffer.data(), temp_buffer.size());
            if (received == 0)
                break;
            
            receive_buffer.appendRaw(temp_buffer.data(), received);
        }
        
        // Process packets
        while (receive_buffer.available() >= 2)
        {
            command_t cmd;
            receive_buffer >> cmd;
            
            switch (cmd)
            {
            case CMD_REQUEST_AUTH:
                handleAuthRequest(receive_buffer);
                break;
            case CMD_SET_CLIENT_ID:
                receive_buffer >> client_id;
                std::cout << "Connected! Client ID: " << client_id << std::endl;
                break;
            case CMD_CREATE:
                handleCreateObject(receive_buffer);
                break;
            case CMD_UPDATE_VALUE:
                handleUpdateObject(receive_buffer);
                break;
            case CMD_DELETE:
                handleDeleteObject(receive_buffer);
                break;
            case CMD_ALIVE_RESP:
                // Keepalive response
                break;
            default:
                std::cout << "Unknown command: " << cmd << std::endl;
                break;
            }
        }
    }
    
private:
    void handleAuthRequest(sp::io::DataBuffer& packet)
    {
        int32_t server_version;
        bool password_required;
        
        packet >> server_version >> password_required;
        
        std::cout << "Server version: " << server_version << std::endl;
        std::cout << "Password required: " << password_required << std::endl;
        
        // Send authentication
        sp::io::DataBuffer auth_packet;
        auth_packet << CMD_CLIENT_SEND_AUTH;
        auth_packet << version_number;
        auth_packet << std::string("");  // Empty password
        
        socket.send(auth_packet.getData(), auth_packet.getDataSize());
    }
    
    void handleCreateObject(sp::io::DataBuffer& packet)
    {
        int32_t object_id;
        std::string class_name;
        uint16_t member_count;
        
        packet >> object_id >> class_name >> member_count;
        
        std::cout << "Create object: " << class_name 
                  << " (ID: " << object_id << ", "
                  << member_count << " members)" << std::endl;
        
        // Read and discard member data for this example
        for (uint16_t i = 0; i < member_count; i++)
        {
            uint16_t member_index;
            packet >> member_index;
            // Would need to know member types to properly deserialize
        }
    }
    
    void handleUpdateObject(sp::io::DataBuffer& packet)
    {
        int32_t object_id;
        uint16_t update_count;
        
        packet >> object_id >> update_count;
        
        std::cout << "Update object " << object_id 
                  << " (" << update_count << " members)" << std::endl;
    }
    
    void handleDeleteObject(sp::io::DataBuffer& packet)
    {
        int32_t object_id;
        packet >> object_id;
        
        std::cout << "Delete object " << object_id << std::endl;
    }
};

int main()
{
    MinimalClient client(20240101);
    
    if (!client.connect("localhost", 35666))
    {
        std::cerr << "Failed to connect" << std::endl;
        return 1;
    }
    
    std::cout << "Connected to server" << std::endl;
    
    // Main loop
    while (true)
    {
        client.update();
        // Sleep briefly to avoid busy-waiting
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
    
    return 0;
}
```

### Python Client Example

For reference, here's how you might implement a basic client in Python:

```python
import socket
import struct
import time

# Commands
CMD_REQUEST_AUTH = 0x0009
CMD_CLIENT_SEND_AUTH = 0x0010
CMD_SET_CLIENT_ID = 0x0004
CMD_CREATE = 0x0001
CMD_UPDATE_VALUE = 0x0002
CMD_DELETE = 0x0003
CMD_ALIVE = 0x0007
CMD_ALIVE_RESP = 0x0012

class DataBuffer:
    def __init__(self):
        self.data = bytearray()
        self.pos = 0
    
    def write_uint16(self, value):
        # VLQ unsigned encoding
        if value >= (1 << 14):
            self.data.append(((value >> 14) & 0x7F) | 0x80)
        if value >= (1 << 7):
            self.data.append(((value >> 7) & 0x7F) | 0x80)
        self.data.append(value & 0x7F)
    
    def write_int32(self, value):
        # VLQ signed encoding (zigzag)
        if value < 0:
            unsigned = ((-value) << 1) | 1
        else:
            unsigned = value << 1
        
        if unsigned >= (1 << 28):
            self.data.append(((unsigned >> 28) & 0x7F) | 0x80)
        if unsigned >= (1 << 21):
            self.data.append(((unsigned >> 21) & 0x7F) | 0x80)
        if unsigned >= (1 << 14):
            self.data.append(((unsigned >> 14) & 0x7F) | 0x80)
        if unsigned >= (1 << 7):
            self.data.append(((unsigned >> 7) & 0x7F) | 0x80)
        self.data.append(unsigned & 0x7F)
    
    def write_string(self, value):
        encoded = value.encode('utf-8')
        self.write_uint32(len(encoded))
        self.data.extend(encoded)
    
    def read_uint16(self):
        result = 0
        while True:
            b = self.data[self.pos]
            self.pos += 1
            result = (result << 7) | (b & 0x7F)
            if not (b & 0x80):
                break
        return result
    
    def read_int32(self):
        unsigned = self.read_uint32()
        if unsigned & 1:
            return -(unsigned >> 1)
        return unsigned >> 1
    
    def read_bool(self):
        b = self.data[self.pos]
        self.pos += 1
        return b != 0
    
    def read_string(self):
        length = self.read_uint32()
        s = self.data[self.pos:self.pos+length].decode('utf-8')
        self.pos += length
        return s

class SeriousProtonClient:
    def __init__(self, host, port, version):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.setblocking(False)
        self.host = host
        self.port = port
        self.version = version
        self.client_id = None
        self.recv_buffer = bytearray()
        
    def connect(self):
        try:
            self.sock.connect((self.host, self.port))
        except BlockingIOError:
            pass  # Non-blocking connect
    
    def update(self):
        # Receive data
        try:
            data = self.sock.recv(4096)
            if data:
                self.recv_buffer.extend(data)
        except BlockingIOError:
            pass
        
        # Process packets
        while len(self.recv_buffer) >= 2:
            buf = DataBuffer()
            buf.data = self.recv_buffer
            
            try:
                cmd = buf.read_uint16()
                
                if cmd == CMD_REQUEST_AUTH:
                    self.handle_auth_request(buf)
                elif cmd == CMD_SET_CLIENT_ID:
                    self.client_id = buf.read_int32()
                    print(f"Connected! Client ID: {self.client_id}")
                elif cmd == CMD_CREATE:
                    self.handle_create(buf)
                elif cmd == CMD_ALIVE_RESP:
                    pass
                
                # Remove processed data
                self.recv_buffer = self.recv_buffer[buf.pos:]
            except:
                break  # Not enough data yet
    
    def handle_auth_request(self, buf):
        server_version = buf.read_int32()
        password_required = buf.read_bool()
        
        print(f"Server version: {server_version}")
        
        # Send authentication
        packet = DataBuffer()
        packet.write_uint16(CMD_CLIENT_SEND_AUTH)
        packet.write_int32(self.version)
        packet.write_string("")  # No password
        
        self.sock.send(packet.data)
    
    def handle_create(self, buf):
        object_id = buf.read_int32()
        class_name = buf.read_string()
        member_count = buf.read_uint16()
        
        print(f"Create object: {class_name} (ID: {object_id})")
    
    def send_keepalive(self):
        packet = DataBuffer()
        packet.write_uint16(CMD_ALIVE)
        self.sock.send(packet.data)

# Usage
client = SeriousProtonClient("localhost", 35666, 20240101)
client.connect()

last_keepalive = time.time()
while True:
    client.update()
    
    # Send keepalive every 0.5 seconds
    if time.time() - last_keepalive > 0.5:
        client.send_keepalive()
        last_keepalive = time.time()
    
    time.sleep(0.01)
```

## Implementation Notes

### Version Compatibility

- Version numbers must **match exactly** between client and server
- No backward compatibility is enforced by the protocol
- Applications should use a versioning scheme (e.g., YYYYMMDD format)

### Bandwidth Optimization

- Use **update delays** on high-frequency data (positions, velocities)
- Only changed members are sent (delta compression)
- VLQ encoding reduces integer sizes
- Consider using proxies for distant clients

### Security Considerations

- **No encryption**: TCP traffic is unencrypted
  - Use VPN or implement TLS wrapper for secure communication
- **Password authentication**: Simple plaintext password
  - Not suitable for untrusted networks
- **Command validation**: Always validate client commands on server
- **DOS protection**: Server should rate-limit clients

### Performance

- **Non-blocking sockets**: All network I/O is non-blocking
- **Single-threaded update loop**: Server processes all clients in one thread
- **Update throttling**: Members can specify minimum update intervals
- **Object pooling**: Reuse DataBuffer objects to reduce allocations

### Thread Safety

- SeriousProton multiplayer is **not thread-safe**
- All network operations should occur on the main game thread
- Master server updates run in a separate thread but use thread-safe communication

## Troubleshooting

### Connection Fails

1. Verify server is running and listening on correct port
2. Check firewall rules
3. Ensure no version mismatch
4. Verify password if required

### Objects Not Updating

1. Check that members are registered with `registerMemberReplication`
2. Verify object is created on server (has valid ID)
3. Check update delays aren't too restrictive
4. Ensure change detection is working (comparison operator for custom types)

### High Bandwidth Usage

1. Increase update delays on frequently changing members
2. Reduce precision (quantize floats)
3. Use proxies for distant clients
4. Consider delta encoding for vectors

### Master Server Not Listing Server

1. Verify server is publicly reachable (port forwarding)
2. Check registration is sent every < 5 minutes
3. Ensure master server can connect back to game server
4. Verify MySQL database is configured correctly

## Conclusion

This document provides a complete specification of the SeriousProton multiplayer protocol. With this information, you should be able to:

- Implement custom clients in any language
- Understand how game state is synchronized
- Debug network issues
- Optimize bandwidth usage
- Extend the protocol for custom applications

For more details, refer to the source code:
- `src/multiplayer_internal.h` - Command definitions
- `src/multiplayer_server.h/cpp` - Server implementation
- `src/multiplayer_client.h/cpp` - Client implementation
- `src/multiplayer.h/cpp` - Object replication
- `src/io/dataBuffer.h` - Binary serialization
- `masterserver/` - Master server PHP implementation
