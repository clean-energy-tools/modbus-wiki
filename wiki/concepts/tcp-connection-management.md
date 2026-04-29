---
title: TCP Connection Management
Summary: TCP connection lifecycle management for MODBUS/TCP including establishment, maintenance, pooling, and error handling.
Sources:
  - raw/MODBUS/messagingimplementationguide.md
Categories:
  - tcp-ip
  - networking
  - connection-lifecycle
type: concept
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:24+03:00
---

TCP connection management for MODBUS/TCP covers how to open, keep alive, and close Ethernet connections for MODBUS communication (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)).

## What Connection Management Does

MODBUS/TCP needs proper connection handling to work reliably and efficiently over Ethernet networks (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)).

### Connection Features

| Feature | Details |
|----------|-------|
| Port number | 502 (normal), 802 (encrypted) |
| Pattern | Client asks, server answers |
| How long to keep open | Keep connections open (don't close after each message) |
| Multiple clients | Yes, servers can handle many clients at once |
| Multiple messages | Yes, send many messages on same connection |

## Ways to Manage Connections

### Manual Connection Management

**What it means:** Your program handles all the connection details yourself.

**Good things:**
- Complete control over how connections work
- Can customize for your specific needs
- Most flexible

**Challenges:**
- You need to understand TCP/IP networking
- More code to write and maintain
- Takes more development time

**What you need to do:**
- Use socket programming (BSD sockets)
- Open, maintain, and close connections yourself
- Keep track of all your connections

### Automatic Connection Management

**Description:** Connection management module handles all TCP operations transparently.

**Advantages:**
- Simpler application code
- No TCP expertise required
- Automatic connection pooling and management

**Disadvantages:**
- Less flexibility
- Less control over connection behavior
- May not optimize for all use cases

**Features:**
- Automatic connection establishment
- Connection pooling
- Automatic connection cleanup
- Transparent to application

## Connection Establishment

### Client Connection

**Process:**
1. Create TCP socket
2. Connect to server IP:502
3. Configure socket options
4. Use connection for MODBUS transactions

**Code example (BSD sockets):**
```c
int create_modbus_tcp_client(const char *server_ip) {
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) return -1;

    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(502);
    inet_pton(AF_INET, server_ip, &server_addr.sin_addr);

    if (connect(sockfd, (struct sockaddr*)&server_addr,
               sizeof(server_addr)) < 0) {
        close(sockfd);
        return -1;
    }

    // Configure socket options
    int opt = 1;
    setsockopt(sockfd, IPPROTO_TCP, TCP_NODELAY, &opt, sizeof(opt));
    setsockopt(sockfd, SOL_SOCKET, SO_KEEPALIVE, &opt, sizeof(opt));

    return sockfd;
}
```

**Local port selection:**
- Client uses ephemeral port (> 1024)
- Different local port for each connection
- TCP stack assigns automatically

### Server Connection

**Process:**
1. Create TCP socket
2. Bind to port 502
3. Listen for connections
4. Accept incoming connections
5. Spawn handlers for each connection

**Code example (BSD sockets):**
```c
int create_modbus_tcp_server() {
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (listen_fd < 0) return -1;

    int opt = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(502);

    if (bind(listen_fd, (struct sockaddr*)&server_addr,
             sizeof(server_addr)) < 0) {
        close(listen_fd);
        return -1;
    }

    if (listen(listen_fd, MAX_CONNECTIONS) < 0) {
        close(listen_fd);
        return -1;
    }

    return listen_fd;
}

int accept_connection(int listen_fd) {
    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);

    int client_fd = accept(listen_fd,
                           (struct sockaddr*)&client_addr,
                           &client_len);

    if (client_fd >= 0) {
        int opt = 1;
        setsockopt(client_fd, IPPROTO_TCP, TCP_NODELAY,
                   &opt, sizeof(opt));
        setsockopt(client_fd, SOL_SOCKET, SO_KEEPALIVE,
                   &opt, sizeof(opt));
    }

    return client_fd;
}
```

## Socket Options

### TCP_NODELAY

**Purpose:** Disable Nagle algorithm for immediate transmission

**Usage:**
```c
int opt = 1;
setsockopt(sockfd, IPPROTO_TCP, TCP_NODELAY, &opt, sizeof(opt));
```

**Benefits:**
- Sends small packets immediately
- Better real-time performance
- No delay waiting for ACKs

**When to use:**
- Always recommended for MODBUS/TCP
- Critical for real-time control applications

### SO_KEEPALIVE

**Purpose:** Enable TCP keep-alive to detect dead connections

**Usage:**
```c
int opt = 1;
setsockopt(sockfd, SOL_SOCKET, SO_KEEPALIVE, &opt, sizeof(opt));
```

**Parameters (platform-specific):**
- Idle time before first probe: 2 hours (default)
- Probe interval: 75 seconds
- Maximum probes: 8

**Benefits:**
- Detects crashed/absent endpoints
- Automatic connection cleanup
- Prevents hung connections

**When to use:**
- Always recommended for MODBUS/TCP
- Critical for long-lived connections

### SO_REUSEADDR

**Purpose:** Allow immediate port reuse after connection close

**Usage:**
```c
int opt = 1;
setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```

**Benefits:**
- Bypasses TIME-WAIT state
- Faster server restart
- Allows immediate port reuse

**When to use:**
- Recommended for server sockets
- Helpful for quick restarts

### SO-RCVBUF, SO-SNDBUF

**Purpose:** Adjust socket buffer sizes for flow control

**Usage:**
```c
int recv_buf_size = 900;  // 3 frames of 300 bytes
int send_buf_size = 900;
setsockopt(sockfd, SOL_SOCKET, SO_RCVBUF,
           &recv_buf_size, sizeof(recv_buf_size));
setsockopt(sockfd, SOL_SOCKET, SO_SNDBUF,
           &send_buf_size, sizeof(send_buf_size));
```

**Benefits:**
- Adjust buffer sizes for application needs
- Balance performance vs resource usage
- Optimize for typical frame sizes

## Connection Lifecycle

### Establishment Phase

**Client:**
1. Create socket
2. Configure socket options
3. Connect to server:502
4. Connection ready for MODBUS transactions

**Server:**
1. Create listening socket on port 502
2. Configure socket options
3. Listen for connections
4. Accept incoming connections
5. Create handler for each connection

### Active Phase

**Client behavior:**
- Send MODBUS requests on existing connection
- Receive responses
- Support concurrent transactions (unique Transaction IDs)
- Handle timeouts and errors
- Keep connection open for multiple transactions

**Server behavior:**
- Process incoming requests
- Send responses
- Handle multiple clients simultaneously
- Manage connection pool
- Detect dead connections (keep-alive)

### Termination Phase

**Client-initiated close:**
1. Client sends FIN
2. Client waits for server ACK
3. Server sends FIN
4. Client sends ACK
5. Connection closed

**Server-initiated close:**
1. Server sends FIN
2. Server waits for client ACK
3. Client sends FIN
4. Server sends ACK
5. Connection closed

**Graceful shutdown:**
```c
void shutdown_connection(int sockfd) {
    shutdown(sockfd, SHUT_RDWR);  // Disable send/receive
    close(sockfd);
}
```

## Connection Pool Management

### Two-Pool Architecture

**Priority Connection Pool:**
- Connections for "marked" devices (specific IP addresses)
- Never closed on local initiative
- Configurable maximum connections per device
- Used for critical devices

**Non-priority Connection Pool:**
- Connections for non-marked devices
- Close oldest unused connection when pool full
- Configurable maximum connections
- Used for regular devices

### Pool Strategy

**Connection request from marked device:**
1. Check priority pool for available connection
2. If available connection exists → use it
3. If no connection but pool has capacity → create new
4. If pool full → reject or close oldest from non-priority

**Connection request from non-marked device:**
1. Check non-priority pool for available connection
2. If available connection exists → use it
3. If no connection but pool has capacity → create new
4. If pool full → close oldest unused, create new

**Marked device configuration:**
```
Device IP: 192.168.1.100
Max connections: 2
Marked: true
Pool: Priority
```

### Connection Pooling Benefits

**Performance:**
- Reuse existing connections
- Avoid connection establishment overhead
- Lower latency for transactions

**Resource Management:**
- Control total connections
- Prevent resource exhaustion
- Prioritize critical devices

**Reliability:**
- Established connections are more reliable
- Avoid frequent connection/disconnection cycles
- Better stability for continuous operation

## Implementation Rules

### Recommended Practices (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md))

1. **Automatic connection management** - Use automatic management without explicit user requirement

2. **Keep connections open** - Do not open/close connection per MODBUS transaction
   - Use single connection for multiple transactions
   - Close only when communications complete

3. **Minimum connections** - One connection per application to same server
   - Reduces resource usage
   - Simplifies management

4. **Concurrent transactions** - Multiple MODBUS transactions on same TCP connection
   - Must use unique Transaction IDs
   - Supports parallel request processing

5. **Separate connections** for bi-directional communication
   - Each device acting as both client and server needs separate connections

6. **One ADU per TCP frame** - Send only one MODBUS request/response per TCP send
   - Do not batch multiple ADUs in single TCP write

### Connection Timeout Management

**Response timeout:**
- Application-specific (typically 1-5 seconds)
- Use application-level timers
- Handle timeout with retry or error

**Keep-alive timeout:**
- Default: 2 hours idle time
- Probe interval: 75 seconds
- Maximum probes: 8
- Adjust based on application requirements

**Connection establishment timeout:**
- Default: 75 seconds on Berkeley-derived systems
- Adapt to application real-time requirements

## Error Handling

### Connection Errors

**Connection failure:**
- Unable to establish TCP connection
- Check server availability
- Verify network connectivity
- Retry with exponential backoff

**Connection reset:**
- Connection closed by remote
- Handle gracefully
- Re-establish if needed

**Connection timeout:**
- No response within timeout period
- Implement retry logic
- Consider connection dead

### Transaction Errors

**Request failure:**
- Unable to send request
- Check connection status
- Re-establish connection if needed

**Response timeout:**
- No response within timeout period
- Retry transaction
- Check connection health

**Exception response:**
- Server returned MODBUS exception
- Parse exception code
- Handle according to exception type

### Failure Recovery

**Reconnection strategy:**
```c
int send_with_retry(int sockfd, uint8_t *data, int len,
                   int max_retries, int retry_delay_ms) {
    for (int i = 0; i < max_retries; i++) {
        if (send(sockfd, data, len, 0) == len) {
            return 0;  // Success
        }
        if (i < max_retries - 1) {
            usleep(retry_delay_ms * 1000);
        }
    }
    return -1;  // Failed after all retries
}
```

**Connection health check:**
```c
int check_connection_health(int sockfd) {
    int error = 0;
    socklen_t len = sizeof(error);
    getsockopt(sockfd, SOL_SOCKET, SO_ERROR, &error, &len);
    return error;  // 0 = healthy, non-zero = error
}
```

## Related pages

- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)
- [MBAP Header](/wiki/concepts/mbap-header.md)
- [Function Codes](/wiki/concepts/function-codes.md)

## Backlinks

- [Connecting with MODBUS TCP Security](/wiki/answers/connecting-with-modbus-tcp-security.md)
- [MODBUS TCP Message Format](/wiki/answers/modbus-tcp-message-format.md)
- [What is MBAP](/wiki/answers/what-is-mbap.md)
- [Function Codes](/wiki/concepts/function-codes.md)
- [MBAP Header](/wiki/concepts/mbap-header.md)
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)
- [Summary of messagingimplementationguide](/wiki/summaries/MODBUS/messagingimplementationguide.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
