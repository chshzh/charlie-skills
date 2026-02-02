# NCS Project Feature Selection Guide

Comprehensive guide for selecting and configuring features in NCS projects.

---

## Overview

This guide provides detailed information for each feature that can be included in an NCS project. Use this during project planning and PDR creation to understand requirements, dependencies, and configurations.

---

## Wi-Fi Features

### Wi-Fi Shell

**Description**: Interactive shell commands for Wi-Fi management, scanning, configuration, and debugging.

**Use Cases**:
- Development and debugging
- Runtime Wi-Fi configuration
- Network diagnostics
- Quick testing without custom code

**Required Kconfig**:
```conf
# Wi-Fi Shell Support
CONFIG_WIFI=y
CONFIG_WIFI_NRF70=y
CONFIG_WIFI_MGMT_EXT=y
CONFIG_NET_L2_WIFI_SHELL=y

# Shell Framework
CONFIG_SHELL=y
CONFIG_SHELL_BACKEND_SERIAL=y
CONFIG_SHELL_STACK_SIZE=4096

# Memory (combined with other features)
CONFIG_HEAP_MEM_POOL_SIZE=81920  # Minimum 80KB for Wi-Fi
```

**Shell Commands**:
```bash
wifi scan                           # Scan for networks
wifi connect <SSID> <password>      # Connect to AP
wifi disconnect                     # Disconnect from AP
wifi status                         # Show connection status
wifi statistics                     # Show statistics
wifi ap enable <SSID> [password]    # Start SoftAP
wifi ap disable                     # Stop SoftAP
```

**Memory Requirements**:
- Flash: +10KB
- RAM: +4KB (shell stack)
- Heap: 80KB+ (Wi-Fi requirement)

**Dependencies**:
- At least one Wi-Fi mode (STA/SoftAP/P2P)
- Network stack

**Example Implementation**:
```c
// Shell is automatic - no code needed
// Just enable configs and use shell commands
```

**Overlay File**: `overlay-wifi-shell.conf`

---

### Wi-Fi STA (Station Mode)

**Description**: Connect to existing Wi-Fi access points as a client device.

**Use Cases**:
- IoT devices connecting to home/office networks
- Cloud connectivity
- Internet access for protocols (MQTT, HTTP, etc.)

**Required Kconfig**:
```conf
# Wi-Fi Core
CONFIG_WIFI=y
CONFIG_WIFI_NRF70=y
CONFIG_WIFI_READY_LIB=y

# Networking
CONFIG_NETWORKING=y
CONFIG_NET_SOCKETS=y
CONFIG_NET_L2_ETHERNET=y
CONFIG_NET_IPV4=y
CONFIG_NET_IPV6=n  # Optional, enable if needed
CONFIG_NET_DHCPV4=y

# Connection Manager
CONFIG_NET_CONNECTION_MANAGER=y
CONFIG_L2_WIFI_CONNECTIVITY=y

# Memory
CONFIG_HEAP_MEM_POOL_SIZE=81920
CONFIG_NET_BUF_RX_COUNT=16
CONFIG_NET_BUF_TX_COUNT=16
CONFIG_NET_PKT_RX_COUNT=16  
CONFIG_NET_PKT_TX_COUNT=16

# Security (recommended)
CONFIG_WIFI_NM_WPA_SUPPLICANT=y
CONFIG_WIFI_NM_WPA_SUPPLICANT_WPA2=y
CONFIG_WIFI_NM_WPA_SUPPLICANT_WPA3=y  # Optional

# Power Management (disable for development)
CONFIG_NRF_WIFI_LOW_POWER=n
```

**Code Example**:
```c
#include <zephyr/net/wifi_mgmt.h>
#include <zephyr/net/net_if.h>
#include <zephyr/net/net_event.h>

static struct net_mgmt_event_callback wifi_cb;
static struct net_mgmt_event_callback net_cb;

static void wifi_mgmt_event_handler(struct net_mgmt_event_callback *cb,
                                     uint32_t mgmt_event, struct net_if *iface)
{
    switch (mgmt_event) {
    case NET_EVENT_WIFI_CONNECT_RESULT:
        LOG_INF("Wi-Fi connected");
        break;
    case NET_EVENT_WIFI_DISCONNECT_RESULT:
        LOG_INF("Wi-Fi disconnected");
        break;
    }
}

static void net_mgmt_event_handler(struct net_mgmt_event_callback *cb,
                                    uint32_t mgmt_event, struct net_if *iface)
{
    if (mgmt_event == NET_EVENT_IPV4_ADDR_ADD) {
        char addr_str[NET_IPV4_ADDR_LEN];
        struct net_if_config *cfg = net_if_get_config(iface);
        
        net_addr_ntop(AF_INET, &cfg->ip.ipv4->unicast[0].address.in_addr,
                      addr_str, sizeof(addr_str));
        LOG_INF("IP Address: %s", addr_str);
    }
}

int wifi_connect(const char *ssid, const char *psk)
{
    struct wifi_connect_req_params params = {0};
    
    params.ssid = ssid;
    params.ssid_length = strlen(ssid);
    params.psk = psk;
    params.psk_length = strlen(psk);
    params.channel = WIFI_CHANNEL_ANY;
    params.security = WIFI_SECURITY_TYPE_PSK;
    params.mfp = WIFI_MFP_OPTIONAL;
    params.band = WIFI_FREQ_BAND_2_4_GHZ;
    
    return net_mgmt(NET_REQUEST_WIFI_CONNECT, 
                    net_if_get_default(), 
                    &params, 
                    sizeof(params));
}

void wifi_init(void)
{
    net_mgmt_init_event_callback(&wifi_cb, wifi_mgmt_event_handler,
                                  NET_EVENT_WIFI_CONNECT_RESULT |
                                  NET_EVENT_WIFI_DISCONNECT_RESULT);
    
    net_mgmt_init_event_callback(&net_cb, net_mgmt_event_handler,
                                  NET_EVENT_IPV4_ADDR_ADD);
    
    net_mgmt_add_event_callback(&wifi_cb);
    net_mgmt_add_event_callback(&net_cb);
}
```

**Memory Requirements**:
- Flash: +80KB
- RAM: +40KB
- Heap: 80KB minimum

**Dependencies**:
- None (standalone)

**Overlay File**: `overlay-wifi-sta.conf`

---

### Wi-Fi SoftAP (Software Access Point)

**Description**: Create a Wi-Fi access point that other devices can connect to.

**Use Cases**:
- Configuration portals
- Local device communication
- Peer-to-peer networks
- Standalone Wi-Fi networks

**Required Kconfig**:
```conf
# Wi-Fi Core
CONFIG_WIFI=y
CONFIG_WIFI_NRF70=y
CONFIG_WIFI_READY_LIB=y

# Access Point Mode
CONFIG_WIFI_NRF70_AP_MODE=y

# Networking
CONFIG_NETWORKING=y
CONFIG_NET_SOCKETS=y
CONFIG_NET_L2_ETHERNET=y
CONFIG_NET_IPV4=y
CONFIG_NET_DHCPV4=y

# DHCP Server (to assign IPs to clients)
CONFIG_NET_DHCPV4_SERVER=y
CONFIG_NET_DHCPV4_SERVER_INSTANCES=1

# Memory
CONFIG_HEAP_MEM_POOL_SIZE=102400  # 100KB recommended for AP mode
CONFIG_NET_BUF_RX_COUNT=24
CONFIG_NET_BUF_TX_COUNT=24

# Security
CONFIG_WIFI_NM_HOSTAPD=y
CONFIG_WIFI_NM_HOSTAPD_WPA2=y
CONFIG_WIFI_NM_HOSTAPD_WPA3=y  # Optional

# Power Management (must disable for AP)
CONFIG_NRF_WIFI_LOW_POWER=n
```

**Code Example**:
```c
#include <zephyr/net/wifi_mgmt.h>

int wifi_ap_enable(const char *ssid, const char *psk, uint8_t channel)
{
    struct wifi_connect_req_params params = {0};
    
    params.ssid = ssid;
    params.ssid_length = strlen(ssid);
    params.psk = psk;
    params.psk_length = strlen(psk);
    params.channel = channel;  // e.g., 6
    params.security = WIFI_SECURITY_TYPE_PSK;
    params.band = WIFI_FREQ_BAND_2_4_GHZ;
    
    return net_mgmt(NET_REQUEST_WIFI_AP_ENABLE,
                    net_if_get_default(),
                    &params,
                    sizeof(params));
}

int wifi_ap_disable(void)
{
    return net_mgmt(NET_REQUEST_WIFI_AP_DISABLE,
                    net_if_get_default(),
                    NULL, 0);
}
```

**Memory Requirements**:
- Flash: +90KB
- RAM: +50KB
- Heap: 100KB recommended

**Dependencies**:
- None (standalone)

**Overlay File**: `overlay-wifi-softap.conf`

---

### Wi-Fi P2P (Wi-Fi Direct)

**Description**: Peer-to-peer Wi-Fi connections without an access point.

**Use Cases**:
- Direct device-to-device communication
- File transfer
- Gaming/streaming
- IoT mesh networks

**Required Kconfig**:
```conf
# Wi-Fi Core
CONFIG_WIFI=y
CONFIG_WIFI_NRF70=y
CONFIG_WIFI_READY_LIB=y

# P2P Mode
CONFIG_NRF70_P2P_MODE=y

# Networking
CONFIG_NETWORKING=y
CONFIG_NET_SOCKETS=y
CONFIG_NET_L2_ETHERNET=y
CONFIG_NET_IPV4=y

# DHCP (both client and server for P2P)
CONFIG_NET_DHCPV4=y
CONFIG_NET_DHCPV4_SERVER=y
CONFIG_NET_DHCPV4_SERVER_INSTANCES=1

# Memory
CONFIG_HEAP_MEM_POOL_SIZE=102400
CONFIG_NET_BUF_RX_COUNT=24
CONFIG_NET_BUF_TX_COUNT=24

# Security
CONFIG_WIFI_NM_WPA_SUPPLICANT=y
CONFIG_WIFI_NM_HOSTAPD=y

# CRITICAL: Disable low power for P2P
CONFIG_NRF_WIFI_LOW_POWER=n
```

**Code Example**:
```c
#include <zephyr/net/wifi_mgmt.h>

/* P2P GO (Group Owner) - acts like AP */
int wifi_p2p_start_go(const char *ssid, const char *psk)
{
    struct wifi_connect_req_params params = {0};
    
    params.ssid = ssid;
    params.ssid_length = strlen(ssid);
    params.psk = psk;
    params.psk_length = strlen(psk);
    params.channel = WIFI_CHANNEL_ANY;
    params.security = WIFI_SECURITY_TYPE_PSK;
    params.band = WIFI_FREQ_BAND_2_4_GHZ;
    params.p2p_go_intent = 15;  // 15 = strong GO preference
    
    return net_mgmt(NET_REQUEST_WIFI_P2P_START_GO,
                    net_if_get_default(),
                    &params,
                    sizeof(params));
}

/* P2P CLI (Client) - connects to GO */
int wifi_p2p_connect(const char *ssid, const char *psk)
{
    struct wifi_connect_req_params params = {0};
    
    params.ssid = ssid;
    params.ssid_length = strlen(ssid);
    params.psk = psk;
    params.psk_length = strlen(psk);
    params.channel = WIFI_CHANNEL_ANY;
    params.security = WIFI_SECURITY_TYPE_PSK;
    params.band = WIFI_FREQ_BAND_2_4_GHZ;
    params.p2p_go_intent = 0;  // 0 = strong CLI preference
    
    return net_mgmt(NET_REQUEST_WIFI_P2P_CONNECT,
                    net_if_get_default(),
                    &params,
                    sizeof(params));
}
```

**Memory Requirements**:
- Flash: +95KB
- RAM: +55KB
- Heap: 100KB recommended

**Dependencies**:
- None (standalone)

**Special Notes**:
- GO intent: 0-15 (0=prefer client, 15=prefer GO)
- Add timing delays for 4-way handshake and DHCP
- Both devices need P2P enabled

**Overlay File**: `overlay-wifi-p2p.conf`

---

## Network Protocol Features

### UDP (User Datagram Protocol)

**Description**: Connectionless, lightweight network protocol for fast data transmission.

**Use Cases**:
- Real-time data streaming
- Sensor telemetry
- Video/audio transmission
- Low-latency applications

**Required Kconfig**:
```conf
# Networking must be enabled (via Wi-Fi configs)
CONFIG_NETWORKING=y
CONFIG_NET_SOCKETS=y
CONFIG_NET_IPV4=y
CONFIG_NET_UDP=y

# Socket API
CONFIG_POSIX_API=y

# Buffers (adjust based on throughput needs)
CONFIG_NET_BUF_RX_COUNT=16
CONFIG_NET_BUF_TX_COUNT=16
CONFIG_NET_PKT_RX_COUNT=16
CONFIG_NET_PKT_TX_COUNT=16
CONFIG_NET_BUF_DATA_SIZE=512  # Per packet

# Memory
CONFIG_NET_SOCKETS_SOCKOPT_TOS=y
CONFIG_NET_SOCKETS_POLL_MAX=4
```

**Code Example**:
```c
#include <zephyr/net/socket.h>

/* UDP Server */
int udp_server_start(uint16_t port)
{
    int sock;
    struct sockaddr_in addr;
    char buffer[512];
    
    sock = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    if (sock < 0) {
        return -errno;
    }
    
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    addr.sin_addr.s_addr = INADDR_ANY;
    
    if (bind(sock, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        close(sock);
        return -errno;
    }
    
    while (1) {
        struct sockaddr_in client_addr;
        socklen_t client_len = sizeof(client_addr);
        
        ssize_t len = recvfrom(sock, buffer, sizeof(buffer), 0,
                               (struct sockaddr *)&client_addr, &client_len);
        if (len > 0) {
            LOG_INF("Received %d bytes from %s:%d",
                    len,
                    net_addr_ntop(AF_INET, &client_addr.sin_addr, 
                                  addr_str, sizeof(addr_str)),
                    ntohs(client_addr.sin_port));
            
            // Echo back
            sendto(sock, buffer, len, 0,
                   (struct sockaddr *)&client_addr, client_len);
        }
    }
    
    close(sock);
    return 0;
}

/* UDP Client */
int udp_send(const char *server_ip, uint16_t port, 
             const void *data, size_t len)
{
    int sock;
    struct sockaddr_in addr;
    
    sock = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    if (sock < 0) {
        return -errno;
    }
    
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    inet_pton(AF_INET, server_ip, &addr.sin_addr);
    
    ssize_t sent = sendto(sock, data, len, 0,
                          (struct sockaddr *)&addr, sizeof(addr));
    
    close(sock);
    return (sent == len) ? 0 : -EIO;
}
```

**Memory Requirements**:
- Flash: +5KB
- RAM: +8KB (buffers)
- Heap: Minimal

**Dependencies**:
- Wi-Fi (STA, SoftAP, or P2P)
- Network stack

**Overlay File**: `overlay-udp.conf`

---

### TCP (Transmission Control Protocol)

**Description**: Connection-oriented, reliable protocol with guaranteed delivery.

**Use Cases**:
- File transfer
- Streaming data
- HTTP/HTTPS communications
- MQTT (runs over TCP)

**Required Kconfig**:
```conf
# Networking
CONFIG_NETWORKING=y
CONFIG_NET_SOCKETS=y
CONFIG_NET_IPV4=y
CONFIG_NET_TCP=y

# Socket API
CONFIG_POSIX_API=y

# TCP Configuration
CONFIG_NET_TCP_BACKLOG_SIZE=4
CONFIG_NET_MAX_CONN=4
CONFIG_NET_TCP_RETRY_COUNT=9
CONFIG_NET_TCP_TIME_WAIT_DELAY=250

# Buffers
CONFIG_NET_BUF_RX_COUNT=20
CONFIG_NET_BUF_TX_COUNT=20
CONFIG_NET_PKT_RX_COUNT=20
CONFIG_NET_PKT_TX_COUNT=20
CONFIG_NET_BUF_DATA_SIZE=512
```

**Code Example**:
```c
#include <zephyr/net/socket.h>

/* TCP Server */
int tcp_server_start(uint16_t port)
{
    int server_sock, client_sock;
    struct sockaddr_in addr, client_addr;
    socklen_t client_len = sizeof(client_addr);
    char buffer[512];
    
    server_sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (server_sock < 0) {
        return -errno;
    }
    
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    addr.sin_addr.s_addr = INADDR_ANY;
    
    if (bind(server_sock, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        close(server_sock);
        return -errno;
    }
    
    if (listen(server_sock, 4) < 0) {
        close(server_sock);
        return -errno;
    }
    
    LOG_INF("TCP server listening on port %d", port);
    
    while (1) {
        client_sock = accept(server_sock, 
                            (struct sockaddr *)&client_addr, 
                            &client_len);
        if (client_sock < 0) {
            continue;
        }
        
        LOG_INF("Client connected");
        
        while (1) {
            ssize_t len = recv(client_sock, buffer, sizeof(buffer), 0);
            if (len <= 0) {
                break;  // Connection closed or error
            }
            
            // Echo back
            send(client_sock, buffer, len, 0);
        }
        
        close(client_sock);
        LOG_INF("Client disconnected");
    }
    
    close(server_sock);
    return 0;
}

/* TCP Client */
int tcp_connect_and_send(const char *server_ip, uint16_t port,
                         const void *data, size_t len)
{
    int sock;
    struct sockaddr_in addr;
    
    sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (sock < 0) {
        return -errno;
    }
    
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    inet_pton(AF_INET, server_ip, &addr.sin_addr);
    
    if (connect(sock, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        close(sock);
        return -errno;
    }
    
    ssize_t sent = send(sock, data, len, 0);
    
    close(sock);
    return (sent == len) ? 0 : -EIO;
}
```

**Memory Requirements**:
- Flash: +15KB
- RAM: +16KB (buffers + connection state)
- Heap: +4KB per connection

**Dependencies**:
- Wi-Fi (STA, SoftAP, or P2P)
- Network stack

**Overlay File**: `overlay-tcp.conf`

---

### MQTT (Message Queuing Telemetry Transport)

**Description**: Lightweight publish/subscribe messaging protocol for IoT.

**Use Cases**:
- Cloud telemetry (AWS IoT, Azure IoT)
- Sensor data publishing
- Device control/commands
- Real-time messaging

**Required Kconfig**:
```conf
# MQTT Library
CONFIG_MQTT_LIB=y
CONFIG_MQTT_LIB_TLS=y  # For secure MQTT

# Networking (TCP required)
CONFIG_NETWORKING=y
CONFIG_NET_SOCKETS=y
CONFIG_NET_TCP=y

# TLS/DTLS (for secure connections)
CONFIG_NET_SOCKETS_SOCKOPT_TLS=y
CONFIG_MBEDTLS=y
CONFIG_MBEDTLS_BUILTIN=y
CONFIG_MBEDTLS_ENABLE_HEAP=y
CONFIG_MBEDTLS_HEAP_SIZE=40960  # 40KB for TLS

# MQTT Configuration
CONFIG_MQTT_KEEPALIVE=60
CONFIG_MQTT_CLEAN_SESSION=y

# Memory
CONFIG_NET_BUF_RX_COUNT=24
CONFIG_NET_BUF_TX_COUNT=24
CONFIG_MAIN_STACK_SIZE=4096
```

**Code Example**:
```c
#include <zephyr/net/mqtt.h>
#include <zephyr/net/socket.h>

static struct mqtt_client client;
static uint8_t rx_buffer[256];
static uint8_t tx_buffer[256];
static uint8_t payload_buf[128];

static void mqtt_evt_handler(struct mqtt_client *const c,
                              const struct mqtt_evt *evt)
{
    switch (evt->type) {
    case MQTT_EVT_CONNACK:
        if (evt->result == 0) {
            LOG_INF("MQTT connected");
        } else {
            LOG_ERR("MQTT connection failed: %d", evt->result);
        }
        break;
        
    case MQTT_EVT_DISCONNECT:
        LOG_INF("MQTT disconnected");
        break;
        
    case MQTT_EVT_PUBLISH:
        LOG_INF("MQTT publish received");
        // Handle incoming message
        break;
        
    case MQTT_EVT_PUBACK:
        LOG_INF("MQTT publish acknowledged");
        break;
    }
}

int mqtt_connect(const char *broker_host, uint16_t broker_port,
                 const char *client_id)
{
    struct sockaddr_in broker;
    
    mqtt_client_init(&client);
    
    broker.sin_family = AF_INET;
    broker.sin_port = htons(broker_port);
    inet_pton(AF_INET, broker_host, &broker.sin_addr);
    
    client.broker = &broker;
    client.evt_cb = mqtt_evt_handler;
    client.client_id.utf8 = (uint8_t *)client_id;
    client.client_id.size = strlen(client_id);
    client.protocol_version = MQTT_VERSION_3_1_1;
    
    client.rx_buf = rx_buffer;
    client.rx_buf_size = sizeof(rx_buffer);
    client.tx_buf = tx_buffer;
    client.tx_buf_size = sizeof(tx_buffer);
    
    return mqtt_connect(&client);
}

int mqtt_publish(const char *topic, const void *data, size_t len)
{
    struct mqtt_publish_param param;
    
    param.message.topic.qos = MQTT_QOS_1_AT_LEAST_ONCE;
    param.message.topic.topic.utf8 = (uint8_t *)topic;
    param.message.topic.topic.size = strlen(topic);
    param.message.payload.data = data;
    param.message.payload.len = len;
    param.message_id = sys_rand32_get();
    param.dup_flag = 0;
    param.retain_flag = 0;
    
    return mqtt_publish(&client, &param);
}

int mqtt_subscribe(const char *topic)
{
    struct mqtt_topic subscribe_topic = {
        .topic = {
            .utf8 = (uint8_t *)topic,
            .size = strlen(topic)
        },
        .qos = MQTT_QOS_1_AT_LEAST_ONCE
    };
    
    struct mqtt_subscription_list subscription_list = {
        .list = &subscribe_topic,
        .list_count = 1,
        .message_id = sys_rand32_get()
    };
    
    return mqtt_subscribe(&client, &subscription_list);
}
```

**Memory Requirements**:
- Flash: +45KB (with TLS)
- RAM: +30KB
- Heap: 40KB for TLS

**Dependencies**:
- Wi-Fi STA (internet connectivity)
- TCP
- TLS/DTLS (for secure connections)

**Overlay File**: `overlay-mqtt.conf`

---

### HTTP Client

**Description**: HTTP client for RESTful API communication and web requests.

**Use Cases**:
- REST API calls
- Cloud service integration
- Webhook notifications
- File downloads

**Required Kconfig**:
```conf
# HTTP Client Library
CONFIG_HTTP_CLIENT=y

# Networking
CONFIG_NETWORKING=y
CONFIG_NET_SOCKETS=y
CONFIG_NET_TCP=y

# DNS (for hostname resolution)
CONFIG_DNS_RESOLVER=y
CONFIG_DNS_RESOLVER_ADDITIONAL_BUF_CTR=2
CONFIG_DNS_RESOLVER_ADDITIONAL_QUERIES=2

# TLS for HTTPS
CONFIG_NET_SOCKETS_SOCKOPT_TLS=y
CONFIG_MBEDTLS=y
CONFIG_MBEDTLS_BUILTIN=y
CONFIG_MBEDTLS_ENABLE_HEAP=y
CONFIG_MBEDTLS_HEAP_SIZE=40960

# Memory
CONFIG_MAIN_STACK_SIZE=4096
CONFIG_HTTP_CLIENT_MAX_HOSTNAME_LEN=64
CONFIG_HTTP_CLIENT_MAX_URL_LEN=256
```

**Code Example**:
```c
#include <zephyr/net/http/client.h>
#include <zephyr/net/socket.h>

static uint8_t recv_buf[1024];

static void http_response_cb(struct http_response *rsp,
                              enum http_final_call final_data,
                              void *user_data)
{
    if (final_data == HTTP_DATA_MORE) {
        LOG_INF("Partial data received (%zd bytes)", rsp->data_len);
    } else if (final_data == HTTP_DATA_FINAL) {
        LOG_INF("All data received (%zd bytes total)", rsp->data_len);
        LOG_INF("HTTP Status: %d", rsp->http_status_code);
    }
    
    if (rsp->body_found && rsp->data_len > 0) {
        LOG_HEXDUMP_DBG(rsp->recv_buf, rsp->data_len, "Response:");
    }
}

int http_get(const char *url)
{
    struct http_request req;
    int sock;
    int ret;
    
    memset(&req, 0, sizeof(req));
    
    sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (sock < 0) {
        return -errno;
    }
    
    req.method = HTTP_GET;
    req.url = url;
    req.host = "example.com";  // Extract from URL
    req.protocol = "HTTP/1.1";
    req.response = http_response_cb;
    req.recv_buf = recv_buf;
    req.recv_buf_len = sizeof(recv_buf);
    
    ret = http_client_req(sock, &req, 5000, NULL);
    
    close(sock);
    return ret;
}

int http_post_json(const char *url, const char *json_data)
{
    struct http_request req;
    int sock;
    int ret;
    
    memset(&req, 0, sizeof(req));
    
    sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (sock < 0) {
        return -errno;
    }
    
    req.method = HTTP_POST;
    req.url = url;
    req.host = "example.com";
    req.protocol = "HTTP/1.1";
    req.payload = json_data;
    req.payload_len = strlen(json_data);
    req.content_type_value = "application/json";
    req.response = http_response_cb;
    req.recv_buf = recv_buf;
    req.recv_buf_len = sizeof(recv_buf);
    
    ret = http_client_req(sock, &req, 5000, NULL);
    
    close(sock);
    return ret;
}
```

**Memory Requirements**:
- Flash: +40KB (with TLS)
- RAM: +25KB
- Heap: 40KB for TLS

**Dependencies**:
- Wi-Fi STA (internet connectivity)
- TCP
- DNS resolver
- TLS (for HTTPS)

**Overlay File**: `overlay-http-client.conf`

---

### HTTPS Server

**Description**: Secure HTTP server for hosting web interfaces and APIs on the device.

**Use Cases**:
- Configuration web interface
- REST API endpoints on device
- Status/monitoring dashboard
- Secure device management

**Required Kconfig**:
```conf
# HTTP Server Library
CONFIG_HTTP_SERVER=y
CONFIG_HTTP_SERVER_MAX_CLIENTS=2
CONFIG_HTTP_SERVER_MAX_STREAMS=4

# Networking
CONFIG_NETWORKING=y
CONFIG_NET_SOCKETS=y
CONFIG_NET_TCP=y

# TLS for HTTPS
CONFIG_NET_SOCKETS_SOCKOPT_TLS=y
CONFIG_MBEDTLS=y
CONFIG_MBEDTLS_BUILTIN=y
CONFIG_MBEDTLS_ENABLE_HEAP=y
CONFIG_MBEDTLS_HEAP_SIZE=51200  # 50KB
CONFIG_MBEDTLS_SERVER_NAME_INDICATION=y

# File System (for serving files)
CONFIG_FILE_SYSTEM=y
CONFIG_FILE_SYSTEM_LITTLEFS=y

# Memory
CONFIG_MAIN_STACK_SIZE=8192
CONFIG_NET_BUF_RX_COUNT=32
CONFIG_NET_BUF_TX_COUNT=32
CONFIG_HEAP_MEM_POOL_SIZE=131072  # 128KB
```

**Code Example**:
```c
#include <zephyr/net/http/service.h>

static uint8_t recv_buffer[1024];

HTTP_SERVICE_DEFINE(my_http_service, "0.0.0.0", 443, 4, 10, NULL);

static int http_get_handler(struct http_resource_detail_static *detail,
                             struct http_client_ctx *client)
{
    static const char response[] = 
        "HTTP/1.1 200 OK\r\n"
        "Content-Type: application/json\r\n"
        "\r\n"
        "{\"status\":\"ok\",\"device\":\"nRF7002\"}";
    
    return http_server_sendall(client, response, sizeof(response) - 1);
}

HTTP_RESOURCE_DEFINE(res_status, my_http_service, "/api/status",
                      &http_get_handler);

static int http_post_config_handler(struct http_resource_detail_static *detail,
                                     struct http_client_ctx *client)
{
    // Parse POST data and update configuration
    // Send response
    static const char response[] = 
        "HTTP/1.1 200 OK\r\n"
        "Content-Type: application/json\r\n"
        "\r\n"
        "{\"result\":\"configured\"}";
    
    return http_server_sendall(client, response, sizeof(response) - 1);
}

HTTP_RESOURCE_DEFINE(res_config, my_http_service, "/api/config",
                      NULL, &http_post_config_handler);
```

**Memory Requirements**:
- Flash: +80KB
- RAM: +60KB
- Heap: 128KB (for TLS + connections)

**Dependencies**:
- Wi-Fi (STA or SoftAP)
- TCP
- TLS certificates
- File system (optional, for static files)

**Special Notes**:
- Requires TLS certificates (generate or provision)
- Consider using SoftAP mode for local-only access
- Limit concurrent connections to manage memory

**Overlay File**: `overlay-https-server.conf`

---

### CoAP (Constrained Application Protocol)

**Description**: Lightweight protocol designed for constrained devices and networks.

**Use Cases**:
- IoT communication (lightweight alternative to HTTP)
- Resource-constrained devices
- Local device discovery (multicast)
- M2M communication

**Required Kconfig**:
```conf
# CoAP Library
CONFIG_COAP=y
CONFIG_COAP_EXTENDED_OPTIONS_LEN=y
CONFIG_COAP_URI_WILDCARD=y

# Networking (UDP required)
CONFIG_NETWORKING=y
CONFIG_NET_SOCKETS=y
CONFIG_NET_UDP=y

# DTLS (for secure CoAP)
CONFIG_NET_SOCKETS_SOCKOPT_TLS=y
CONFIG_MBEDTLS=y
CONFIG_MBEDTLS_BUILTIN=y
CONFIG_MBEDTLS_ENABLE_HEAP=y
CONFIG_MBEDTLS_HEAP_SIZE=32768

# Memory
CONFIG_COAP_RESOURCE_MAX_NAME_LEN=32
CONFIG_COAP_RESOURCE_MAX_OBSERVERS=4
```

**Code Example**:
```c
#include <zephyr/net/coap.h>
#include <zephyr/net/socket.h>

static int coap_get_handler(struct coap_resource *resource,
                             struct coap_packet *request,
                             struct sockaddr *addr, socklen_t addr_len)
{
    struct coap_packet response;
    uint8_t response_buf[256];
    const char *payload = "{\"temp\":22.5,\"humidity\":45}";
    
    coap_packet_init(&response, response_buf, sizeof(response_buf),
                     COAP_VERSION_1, COAP_TYPE_ACK, 
                     COAP_TOKEN_MAX_LEN, coap_header_get_token(request),
                     COAP_RESPONSE_CODE_CONTENT, coap_header_get_id(request));
    
    coap_packet_append_option(&response, COAP_OPTION_CONTENT_FORMAT,
                               (uint8_t []){COAP_CONTENT_FORMAT_APP_JSON}, 1);
    
    coap_packet_append_payload_marker(&response);
    coap_packet_append_payload(&response, payload, strlen(payload));
    
    return sendto(resource->user_data, response.data, response.offset, 0,
                  addr, addr_len);
}

static const char * const coap_sensor_path[] = {"sensor", "data", NULL};

static struct coap_resource resources[] = {
    {
        .path = coap_sensor_path,
        .get = coap_get_handler,
    },
};

int coap_server_start(uint16_t port)
{
    int sock;
    struct sockaddr_in addr;
    
    sock = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    if (sock < 0) {
        return -errno;
    }
    
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    addr.sin_addr.s_addr = INADDR_ANY;
    
    if (bind(sock, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        close(sock);
        return -errno;
    }
    
    // Store socket in resource user_data
    resources[0].user_data = (void *)(uintptr_t)sock;
    
    // CoAP packet handling loop
    while (1) {
        uint8_t request_buf[256];
        struct sockaddr_in client_addr;
        socklen_t client_len = sizeof(client_addr);
        
        ssize_t len = recvfrom(sock, request_buf, sizeof(request_buf), 0,
                               (struct sockaddr *)&client_addr, &client_len);
        
        if (len > 0) {
            struct coap_packet request;
            
            if (coap_packet_parse(&request, request_buf, len, NULL, 0) == 0) {
                coap_handle_request(&request, resources, 
                                   ARRAY_SIZE(resources),
                                   (struct sockaddr *)&client_addr, 
                                   client_len);
            }
        }
    }
    
    close(sock);
    return 0;
}
```

**Memory Requirements**:
- Flash: +25KB (with DTLS)
- RAM: +15KB
- Heap: 32KB for DTLS

**Dependencies**:
- Wi-Fi (STA, SoftAP, or P2P)
- UDP
- DTLS (for secure CoAP)

**Overlay File**: `overlay-coap.conf`

---

## Additional Features

### Memfault

**Description**: Cloud-based device monitoring, debugging, and OTA updates.

**Use Cases**:
- Remote crash analysis
- Performance monitoring
- OTA firmware updates
- Device fleet management
- Diagnostic data collection

**Required Kconfig**:
```conf
# Memfault SDK
CONFIG_MEMFAULT=y
CONFIG_MEMFAULT_SHELL=y
CONFIG_MEMFAULT_HTTP_ENABLE=y
CONFIG_MEMFAULT_ROOT_CERT_STORAGE_NRF9160_MODEM=n

# Memfault Features
CONFIG_MEMFAULT_COREDUMP_COLLECT_BSS_REGIONS=y
CONFIG_MEMFAULT_COREDUMP_COLLECT_DATA_REGIONS=y
CONFIG_MEMFAULT_LOGGING_ENABLE=y
CONFIG_MEMFAULT_METRICS=y

# Networking (requires HTTP client)
CONFIG_HTTP_CLIENT=y
CONFIG_MBEDTLS=y
CONFIG_MBEDTLS_HEAP_SIZE=40960

# Flash (for storing coredumps)
CONFIG_FLASH=y
CONFIG_FLASH_MAP=y
CONFIG_STREAM_FLASH=y
CONFIG_NVS=y

# Debugging
CONFIG_DEBUG=y
CONFIG_DEBUG_INFO=y
CONFIG_THREAD_NAME=y
CONFIG_THREAD_STACK_INFO=y

# Memory
CONFIG_HEAP_MEM_POOL_SIZE=98304  # 96KB minimum
CONFIG_MAIN_STACK_SIZE=4096
```

**Project Setup**:
1. Sign up at https://memfault.com
2. Create project and get project key
3. Create `config/memfault_platform_config.h`:

```c
#pragma once

#define MEMFAULT_USE_GNU_BUILD_ID 1
#define MEMFAULT_PLATFORM_HAS_LOG_CONFIG 1
```

4. Create `config/memfault_metrics_heartbeat_config.def`:

```c
MEMFAULT_METRICS_KEY_DEFINE(wifi_connected_time_ms, kMemfaultMetricType_Timer)
MEMFAULT_METRICS_KEY_DEFINE(wifi_disconnects, kMemfaultMetricType_Counter)
MEMFAULT_METRICS_KEY_DEFINE(bytes_sent, kMemfaultMetricType_Counter)
MEMFAULT_METRICS_KEY_DEFINE(bytes_received, kMemfaultMetricType_Counter)
```

**Code Example**:
```c
#include <memfault/core/platform/device_info.h>
#include <memfault/core/trace_event.h>
#include <memfault/metrics/metrics.h>
#include <memfault/ports/zephyr/http.h>

void memfault_platform_get_device_info(sMemfaultDeviceInfo *info)
{
    *info = (sMemfaultDeviceInfo) {
        .device_serial = "DEVICE_SERIAL_NUM",
        .software_type = "nrf7002-app",
        .software_version = "1.0.0",
        .hardware_version = "nrf7002dk",
    };
}

int memfault_init(void)
{
    // Initialize metrics
    memfault_metrics_boot();
    
    LOG_INF("Memfault initialized");
    return 0;
}

void wifi_event_handler(enum wifi_event event)
{
    if (event == WIFI_DISCONNECT) {
        // Track disconnect
        MEMFAULT_TRACE_EVENT_WITH_LOG(wifi_disconnect, "Disconnect event");
        memfault_metrics_heartbeat_add(MEMFAULT_METRICS_KEY(wifi_disconnects), 1);
    }
}

void periodic_upload_task(void)
{
    while (1) {
        // Upload data every hour
        k_sleep(K_HOURS(1));
        
        memfault_zephyr_port_post_data();
    }
}

// Trigger manual upload
void memfault_upload_now(void)
{
    memfault_metrics_heartbeat_debug_trigger();
    memfault_zephyr_port_post_data();
}
```

**Shell Commands**:
```bash
mflt get_device_info          # Show device info
mflt export                   # Export data chunks
mflt trigger_logs             # Save logs to coredump
mflt post_chunks              # Upload to Memfault cloud
```

**Memory Requirements**:
- Flash: +120KB (with HTTPS + Memfault SDK)
- RAM: +50KB
- Heap: 96KB minimum
- Flash storage: 64KB+ (for coredumps)

**Dependencies**:
- Wi-Fi STA (internet connectivity)
- HTTP Client
- TLS
- Flash storage

**Special Notes**:
- Set project key in overlay: `CONFIG_MEMFAULT_NCS_PROJECT_KEY="your-key"`
- Build with symbols: `-DCONFIG_DEBUG=y -DCONFIG_DEBUG_INFO=y`
- Upload can be triggered periodically or on events

**Overlay File**: `overlay-memfault.conf`

---

### BLE Provisioning

**Description**: Provision Wi-Fi credentials via Bluetooth Low Energy.

**Use Cases**:
- Secure credential provisioning without hardcoding
- User-friendly device setup (mobile app)
- Credential updates without reflashing
- Initial device onboarding

**Required Kconfig**:
```conf
# Bluetooth
CONFIG_BT=y
CONFIG_BT_PERIPHERAL=y
CONFIG_BT_DEVICE_NAME="WiFi_Device"

# BLE GATT
CONFIG_BT_GATT_DYNAMIC_DB=y
CONFIG_BT_ATT_PREPARE_COUNT=2
CONFIG_BT_ATT_TX_MAX=3
CONFIG_BT_L2CAP_TX_BUF_COUNT=3

# Wi-Fi (must be enabled)
CONFIG_WIFI=y
CONFIG_WIFI_CREDENTIALS=y
CONFIG_WIFI_CREDENTIALS_BACKEND_SETTINGS=y

# Settings (to store credentials)
CONFIG_SETTINGS=y
CONFIG_SETTINGS_RUNTIME=y
CONFIG_NVS=y
CONFIG_FLASH=y
CONFIG_FLASH_MAP=y

# Memory
CONFIG_BT_BUF_ACL_RX_SIZE=255
CONFIG_BT_BUF_ACL_TX_SIZE=251
CONFIG_HEAP_MEM_POOL_SIZE=98304  # Account for both BLE + Wi-Fi
```

**Code Example**:
```c
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/gatt.h>
#include <zephyr/net/wifi_credentials.h>

/* Custom BLE service UUID for Wi-Fi provisioning */
#define BT_UUID_WIFI_PROV_SERVICE \
    BT_UUID_DECLARE_128(BT_UUID_128_ENCODE(0x12345678, 0x1234, 0x5678, \
                                            0x1234, 0x56789abcdef0))

#define BT_UUID_WIFI_SSID \
    BT_UUID_DECLARE_128(BT_UUID_128_ENCODE(0x12345678, 0x1234, 0x5678, \
                                            0x1234, 0x56789abcdef1))

#define BT_UUID_WIFI_PSK \
    BT_UUID_DECLARE_128(BT_UUID_128_ENCODE(0x12345678, 0x1234, 0x5678, \
                                            0x1234, 0x56789abcdef2))

static uint8_t wifi_ssid[32];
static uint8_t wifi_psk[64];
static size_t ssid_len = 0;
static size_t psk_len = 0;

static ssize_t write_ssid(struct bt_conn *conn,
                          const struct bt_gatt_attr *attr,
                          const void *buf, uint16_t len,
                          uint16_t offset, uint8_t flags)
{
    if (offset + len > sizeof(wifi_ssid)) {
        return BT_GATT_ERR(BT_ATT_ERR_INVALID_OFFSET);
    }
    
    memcpy(wifi_ssid + offset, buf, len);
    ssid_len = offset + len;
    
    LOG_INF("SSID received: %.*s", ssid_len, wifi_ssid);
    return len;
}

static ssize_t write_psk(struct bt_conn *conn,
                         const struct bt_gatt_attr *attr,
                         const void *buf, uint16_t len,
                         uint16_t offset, uint8_t flags)
{
    if (offset + len > sizeof(wifi_psk)) {
        return BT_GATT_ERR(BT_ATT_ERR_INVALID_OFFSET);
    }
    
    memcpy(wifi_psk + offset, buf, len);
    psk_len = offset + len;
    
    LOG_INF("PSK received (%d bytes)", psk_len);
    
    // Save credentials and trigger connection
    save_and_connect_wifi();
    
    return len;
}

BT_GATT_SERVICE_DEFINE(wifi_prov_svc,
    BT_GATT_PRIMARY_SERVICE(BT_UUID_WIFI_PROV_SERVICE),
    BT_GATT_CHARACTERISTIC(BT_UUID_WIFI_SSID,
                           BT_GATT_CHRC_WRITE,
                           BT_GATT_PERM_WRITE,
                           NULL, write_ssid, NULL),
    BT_GATT_CHARACTERISTIC(BT_UUID_WIFI_PSK,
                           BT_GATT_CHRC_WRITE,
                           BT_GATT_PERM_WRITE,
                           NULL, write_psk, NULL),
);

static void save_and_connect_wifi(void)
{
    struct wifi_credentials_personal creds = {
        .header = {
            .type = WIFI_SECURITY_TYPE_PSK,
            .ssid_len = ssid_len,
        },
    };
    
    memcpy(creds.header.ssid, wifi_ssid, ssid_len);
    memcpy(creds.password, wifi_psk, psk_len);
    creds.password_len = psk_len;
    
    // Save to persistent storage
    wifi_credentials_set_personal_struct(&creds);
    
    // Connect to Wi-Fi
    wifi_connect((char *)wifi_ssid, (char *)wifi_psk);
    
    // Optionally disconnect BLE
    //bt_le_adv_stop();
}

static const struct bt_data ad[] = {
    BT_DATA_BYTES(BT_DATA_FLAGS, (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),
    BT_DATA(BT_DATA_NAME_COMPLETE, CONFIG_BT_DEVICE_NAME, 
            sizeof(CONFIG_BT_DEVICE_NAME) - 1),
};

int ble_provisioning_start(void)
{
    int err;
    
    err = bt_enable(NULL);
    if (err) {
        LOG_ERR("Bluetooth init failed (err %d)", err);
        return err;
    }
    
    LOG_INF("Bluetooth initialized");
    
    err = bt_le_adv_start(BT_LE_ADV_CONN, ad, ARRAY_SIZE(ad), NULL, 0);
    if (err) {
        LOG_ERR("Advertising failed to start (err %d)", err);
        return err;
    }
    
    LOG_INF("BLE advertising started - Device: %s", CONFIG_BT_DEVICE_NAME);
    return 0;
}
```

**Usage Flow**:
1. Device boots and starts BLE advertising
2. User connects via BLE (nRF Connect app)
3. User writes SSID to SSID characteristic
4. User writes password to PSK characteristic
5. Device stores credentials and connects to Wi-Fi
6. (Optional) BLE advertising stops

**Memory Requirements**:
- Flash: +60KB
- RAM: +20KB
- Heap: Shared with Wi-Fi (total 98KB+)

**Dependencies**:
- Wi-Fi (any mode)
- Bluetooth LE
- Settings/NVS (to store credentials)

**Special Notes**:
- Consider security: use BLE pairing/bonding
- UUIDs shown are examples - generate unique ones
- Mobile app needed (nRF Connect works for basic testing)
- Can disable BLE after provisioning to save power

**Overlay File**: `overlay-ble-prov.conf`

---

## Feature Combinations

### Common Combinations

#### IoT Sensor with Cloud (MQTT)
- Wi-Fi STA ✓
- Wi-Fi Shell ✓
- UDP ✓
- MQTT ✓
- Memfault ✓

**Total Memory**: Flash ~260KB, RAM ~120KB, Heap ~100KB

---

#### Smart Home Device (HTTP + BLE Setup)
- Wi-Fi STA ✓
- BLE Provisioning ✓
- HTTP Client ✓
- UDP ✓
- Memfault ✓

**Total Memory**: Flash ~280KB, RAM ~130KB, Heap ~100KB

---

#### Local Web Server (Configuration Portal)
- Wi-Fi SoftAP ✓
- HTTPS Server ✓
- Wi-Fi Shell ✓
- TCP ✓

**Total Memory**: Flash ~200KB, RAM ~110KB, Heap ~128KB

---

#### P2P File Transfer
- Wi-Fi P2P ✓
- TCP ✓
- UDP ✓
- Wi-Fi Shell ✓

**Total Memory**: Flash ~120KB, RAM ~80KB, Heap ~100KB

---

## Checklist for Feature Selection

When selecting features, consider:

### Connectivity
- [ ] How does device connect to network? (STA/SoftAP/P2P)
- [ ] Internet required? (use Wi-Fi STA)
- [ ] Local-only? (SoftAP or P2P)
- [ ] Configuration method? (BLE Provisioning)

### Protocols
- [ ] Real-time data? (UDP)
- [ ] Reliable delivery? (TCP)
- [ ] Cloud telemetry? (MQTT)
- [ ] RESTful APIs? (HTTP Client)
- [ ] Web interface? (HTTPS Server)
- [ ] Constrained protocol? (CoAP)

### Management
- [ ] Remote debugging? (Memfault)
- [ ] OTA updates? (Memfault)
- [ ] Shell access? (Wi-Fi Shell)

### Resources
- [ ] Flash budget?
- [ ] RAM budget?
- [ ] Power constraints?
- [ ] Cost sensitivity?

### Security
- [ ] Credential storage? (Settings/NVS)
- [ ] Encrypted traffic? (TLS/DTLS)
- [ ] Secure provisioning? (BLE)
- [ ] Certificate management?

---

## Build Configuration

### Overlay File Architecture

Create modular overlay files that can be combined:

**Example Structure**:
```
project/
├── prj.conf                       # Base config
├── overlay-wifi-sta.conf          # Wi-Fi STA
├── overlay-wifi-shell.conf        # Shell commands
├── overlay-mqtt.conf              # MQTT protocol
├── overlay-memfault.conf          # Memfault monitoring
└── overlay-production.conf        # Production settings
```

**Build Command**:
```bash
west build -p -b nrf7002dk/nrf5340/cpuapp -- \
  -DEXTRA_CONF_FILE="overlay-wifi-sta.conf;overlay-mqtt.conf;overlay-memfault.conf"
```

### Memory Optimization

**Development Build** (all features enabled):
```conf
CONFIG_DEBUG=y
CONFIG_LOG_DEFAULT_LEVEL=4
CONFIG_HEAP_MEM_POOL_SIZE=131072  # 128KB
```

**Production Build** (optimized):
```conf
CONFIG_DEBUG=n
CONFIG_LOG_DEFAULT_LEVEL=2
CONFIG_SIZE_OPTIMIZATIONS=y
CONFIG_HEAP_MEM_POOL_SIZE=102400  # 100KB
CONFIG_NRF_WIFI_LOW_POWER=y       # Enable power saving
```

---

## References

- [NCS Documentation](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest/nrf/index.html)
- [Wi-Fi Guide](WIFI_GUIDE.md)
- [Configuration Guide](CONFIG_GUIDE.md)
- [Project Structure](PROJECT_STRUCTURE.md)

---

**Last Updated**: 2026-01-30
