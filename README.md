# Complete WebRTC Video Streaming Server - Step-by-Step Explanation

## Table of Contents
1. [Overview](#overview)
2. [Program Flow](#program-flow)
3. [Data Structures](#data-structures)
4. [Function Reference](#function-reference)
5. [Detailed Execution Flow](#detailed-execution-flow)

---

## Overview

This is a **GStreamer-based WebRTC video streaming server** that:
- Captures video from a V4L2 camera device
- Encodes video using hardware encoder (H.264 or H.265)
- Streams to multiple browser clients via WebRTC
- Simultaneously sends stream to a UDP destination
- Uses WebSocket for WebRTC signaling

**Architecture Pattern:** One video source → Multiple consumers (tee pattern)

---

## Program Flow

```
┌─────────────────────────────────────────────────────────────┐
│                         MAIN PROGRAM                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  1. Parse Command Line Arguments                             │
│     - bitrate, fps, resolution, codec, device, etc.          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  2. Build Main GStreamer Pipeline                            │
│     v4l2src → encoder → RTP payloader → tee                  │
│                                          ├→ udpsink          │
│                                          └→ (clients added   │
│                                              dynamically)    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  3. Start HTTP/WebSocket Server (port 8080)                  │
│     - Serves index.html on HTTP                              │
│     - Handles WebSocket connections on /ws                   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  4. Start GLib Main Loop                                     │
│     - Processes events                                       │
│     - Waits for client connections                           │
└─────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
                    ▼                   ▼
        ┌──────────────────┐   ┌──────────────────┐
        │ Client Connects  │   │ Client Connects  │
        └──────────────────┘   └──────────────────┘
                    │                   │
                    └─────────┬─────────┘
                              ▼
        ┌────────────────────────────────────────┐
        │ For Each Client:                       │
        │  - Create WebRTC sub-pipeline          │
        │  - Link to tee element                 │
        │  - Perform WebRTC negotiation          │
        │  - Exchange ICE candidates             │
        │  - Stream video                        │
        └────────────────────────────────────────┘
```

---

## Data Structures

### ReceiverEntry
The core data structure representing each connected client:

```cpp
struct _ReceiverEntry {
    SoupWebsocketConnection *connection;  // WebSocket connection to client
    GHashTable *r_table;                  // Reference to receiver table
    
    GstElement *pipeline;                 // Client's sub-pipeline (bin)
    GstElement *webrtcbin;                // WebRTC element
    GstElement *queue;                    // Buffer queue
    
    gchar *client_ip;                     // Client's IP address
    GstPad *tee_src_pad;                  // Source pad from main tee
    GstPad *sink_pad;                     // Sink pad to client queue
};
```

**Purpose:** Each client gets their own `ReceiverEntry` containing all resources needed for streaming.

---

## Function Reference

### 1. Main Entry Point

#### `main(int argc, char *argv[])`
**Lines: 700-851**

**Purpose:** Program initialization and main event loop

**Step-by-Step:**
```
1. Parse command-line arguments (bitrate, codec, resolution, etc.)
2. Build encoder pipeline string based on codec choice
3. Create main GStreamer pipeline:
   v4l2src → queue → encoder → tee → udpsink
4. Get reference to tee element (for dynamic branching)
5. Set pipeline to PLAYING state
6. Create receiver hash table (stores all clients)
7. Create HTTP/WebSocket server
8. Register signal handlers (Ctrl+C cleanup)
9. Start availability update thread
10. Run GLib main loop (blocks here until Ctrl+C)
11. Cleanup on exit
```

**Key Variables:**
- `webrtc_pipeline` - Global main pipeline
- `video_tee` - Global tee element reference
- `receiver_entry_table` - Hash table of all connected clients

---

### 2. HTTP/WebSocket Handlers

#### `soup_http_handler()`
**Lines: 552-583**

**Purpose:** Serve HTML page to browsers

**Flow:**
```
1. Check if path is "/" or "/index.html"
   - If not → Return 404
2. Read index.html file from disk
3. Send file content to browser
4. Return 200 OK
```

#### `soup_websocket_handler()`
**Lines: 585-632**

**Purpose:** Handle new WebSocket connections

**Flow:**
```
1. New WebSocket connection arrives
2. Check if server is available (rate limiting)
   - If not available → Reject connection
3. Extract client's IP address
4. Call create_receiver_entry() to set up streaming
5. Add ReceiverEntry to hash table
6. Register 'closed' callback for cleanup
```

**Rate Limiting:**
- `available` flag prevents simultaneous connections
- `waiting_period` (5 seconds) between connections
- Prevents resource exhaustion

---

### 3. Client Pipeline Management

#### `create_receiver_entry()`
**Lines: 208-308**

**Purpose:** Create streaming pipeline for new client

**Detailed Steps:**

```
Step 1: Allocate ReceiverEntry structure
├─ Allocate memory for ReceiverEntry
├─ Store WebSocket connection reference
└─ Increment connection refcount

Step 2: Create client sub-pipeline
├─ Create bin (container for elements)
├─ Create queue element
│  ├─ max-size-buffers: 100
│  ├─ leaky: 2 (drop old frames if full)
│  └─ flush-on-eos: TRUE (clean shutdown)
├─ Create webrtcbin element
│  ├─ bundle-policy: MAX_BUNDLE
│  ├─ stun-server: stun.l.google.com:19302
│  └─ turn-server: (if configured)
└─ Link: queue → webrtcbin

Step 3: Add to main pipeline
├─ Add client bin to webrtc_pipeline
└─ Create ghost pad on bin (exposes queue's sink pad)

Step 4: Connect to tee element
├─ Request new source pad from tee
├─ Get client bin's sink pad
├─ Link: tee_src_pad → bin_sink_pad
└─ Store pad references in ReceiverEntry

Step 5: Connect signal handlers
├─ on-negotiation-needed → on_negotiation_needed_cb
│  (Triggered when WebRTC needs to create offer)
└─ on-ice-candidate → on_ice_candidate_cb
   (Triggered when ICE candidate is found)

Step 6: Start client pipeline
├─ Set pipeline to PLAYING state
├─ Wait for state change (5 second timeout)
└─ Return ReceiverEntry if successful
```

**Pipeline Structure Created:**
```
Main Pipeline:
  v4l2src → encoder → tee ─┬→ udpsink
                           │
                           ├→ Client1: queue → webrtcbin
                           ├→ Client2: queue → webrtcbin
                           └→ Client3: queue → webrtcbin
```

---

### 4. WebRTC Negotiation

#### `on_negotiation_needed_cb()`
**Lines: 362-378**

**Purpose:** Start WebRTC offer/answer exchange

**Flow:**
```
1. Triggered automatically by webrtcbin when client connects
2. Create GstPromise for async operation
3. Call webrtcbin's "create-offer" signal
4. Promise resolves → calls on_offer_created_cb()
```

**WebRTC Negotiation Sequence:**
```
Client                          Server
  │                               │
  │  1. WebSocket connect         │
  │ ────────────────────────────> │
  │                               │ create_receiver_entry()
  │                               │ webrtcbin triggers negotiation
  │                               │
  │  2. SDP Offer (JSON)          │
  │ <──────────────────────────── │ on_offer_created_cb()
  │                               │
  │  3. SDP Answer (JSON)         │
  │ ────────────────────────────> │ soup_websocket_message_cb()
  │                               │ set-remote-description
  │                               │
  │  4. ICE Candidate             │
  │ ────────────────────────────> │ soup_websocket_message_cb()
  │ <──────────────────────────── │ on_ice_candidate_cb()
  │  5. ICE Candidate             │
  │                               │
  │  (Multiple ICE exchanges)     │
  │                               │
  │  6. DTLS Handshake            │
  │ <══════════════════════════=> │
  │                               │
  │  7. Video streaming starts    │
  │ <════════════════════════════ │
  │        (RTP/SRTP)             │
```

#### `on_offer_created_cb()`
**Lines: 322-360**

**Purpose:** Handle created SDP offer

**Flow:**
```
1. Extract SDP offer from promise
2. Set as local description on webrtcbin
3. Convert SDP to text string
4. Create JSON object:
   {
     "type": "offer",
     "sdp": "<sdp_string>"
   }
5. Send JSON via WebSocket to client
6. Cleanup (free strings, unref promise)
```

**SDP (Session Description Protocol):**
Contains media information:
- Video codec (H.264/H.265)
- RTP payload type (96)
- Network transport details
- Encryption parameters
- Media capabilities

---

### 5. ICE Candidate Exchange

#### `on_ice_candidate_cb()`
**Lines: 380-401**

**Purpose:** Send discovered ICE candidates to client

**Flow:**
```
1. Triggered by webrtcbin when ICE candidate found
2. Receive: mline_index, candidate string
3. Create JSON:
   {
     "type": "ice",
     "candidate": {
       "sdpMLineIndex": 0,
       "candidate": "candidate:..."
     }
   }
4. Send to client via WebSocket
```

**ICE (Interactive Connectivity Establishment):**
- Finds best network path between server and client
- Tests multiple connection possibilities:
  - Direct connection (host candidate)
  - Through NAT (server reflexive)
  - Through TURN relay (relay candidate)

**Example ICE Candidate:**
```
candidate:2914868183 1 udp 2113937151 192.168.25.68 59507 typ host generation 0
           │         │  │        │           │         │     │
           │         │  │        │           │         │     └─ Candidate type
           │         │  │        │           │         └─────── Port
           │         │  │        │           └───────────────── IP address
           │         │  │        └───────────────────────────── Priority
           │         │  └────────────────────────────────────── Protocol
           │         └───────────────────────────────────────── Component (1=RTP)
           └─────────────────────────────────────────────────── Foundation
```

---

### 6. Message Processing

#### `soup_websocket_message_cb()`
**Lines: 403-540**

**Purpose:** Process incoming WebSocket messages from client

**Message Types Handled:**

##### Type 1: "answer" (SDP Answer)
```
1. Client sends their SDP answer
2. Parse JSON to extract SDP string
3. Parse SDP string into GstSDPMessage
4. Create GstWebRTCSessionDescription
5. Set as remote description on webrtcbin
6. WebRTC connection established
```

##### Type 2: "ice-candidate" (ICE Candidate)
```
1. Client sends ICE candidate
2. Parse JSON to extract:
   - sdpMLineIndex (media line index)
   - candidate (ICE candidate string)
3. Handle mDNS workaround:
   - Replace ".local" addresses with actual client IP
   - Example: "candidate:...a1b2c3.local..." → "192.168.25.68"
4. Add ICE candidate to webrtcbin
5. WebRTC tests this connection path
```

**mDNS Workaround (Lines 514-516):**
```cpp
std::regex local_pattern(R"(\S+\.local)");
std::string modified = std::regex_replace(
    std::string(candidate_string), 
    local_pattern, 
    std::string(receiver_entry->client_ip)
);
```

**Why Needed:**
- Browsers use mDNS for privacy (hides local IP)
- Server needs actual IP to connect
- Regex replaces ".local" with real IP

---

### 7. Client Disconnection

#### `soup_websocket_closed_cb()`
**Lines: 542-550**

**Purpose:** Handle client disconnect

**Flow:**
```
1. WebSocket connection closed
2. Look up ReceiverEntry in hash table
3. Add pad probe to tee_src_pad
4. Probe will block data flow → pad_probe_cb()
```

#### `pad_probe_cb()`
**Lines: 147-161**

**Purpose:** Block data flow before cleanup

**Flow:**
```
1. Probe blocks downstream data
2. Remove this probe
3. Add EOS event probe to sink_pad
4. Send EOS event
5. EOS flows through → event_probe_cb()
```

#### `event_probe_cb()`
**Lines: 85-145**

**Purpose:** Clean teardown of client pipeline

**Detailed Cleanup Steps:**
```
Step 1: Wait for EOS event
├─ Only proceed when EOS arrives
└─ Ensures all data is flushed

Step 2: Remove probe
└─ Cleanup probe itself

Step 3: Unlink pads
├─ Unlink tee_src_pad from sink_pad
├─ Release request pad from tee
├─ Unref tee_src_pad
└─ Unref sink_pad

Step 4: Stop client pipeline
├─ Set pipeline to NULL state
├─ Wait for state change
└─ Verify pipeline stopped cleanly

Step 5: Remove from main pipeline
├─ Remove client bin from webrtc_pipeline
├─ Unref webrtcbin element
├─ Unref queue element
└─ Unref pipeline bin

Step 6: Cleanup ReceiverEntry
├─ Remove from hash table
└─ This triggers destroy_receiver_entry()
```

**Why This Complex Process?**
- Can't just disconnect immediately
- Need to flush all pending data
- Must avoid deadlocks in GStreamer
- Ensures clean shutdown without memory leaks

**Visual Flow:**
```
Client disconnects
       │
       ▼
soup_websocket_closed_cb()
       │
       ▼
Add probe to tee_src_pad (blocks data)
       │
       ▼
pad_probe_cb() triggers
       │
       ▼
Send EOS event
       │
       ▼
EOS flows through pipeline
       │
       ▼
event_probe_cb() triggers
       │
       ▼
Unlink pads → Stop pipeline → Remove from main → Free memory
       │
       ▼
Client fully disconnected
```

---

### 8. Helper Functions

#### `destroy_receiver_entry()`
**Lines: 310-320**

**Purpose:** Free ReceiverEntry memory

**Flow:**
```
1. Called when entry removed from hash table
2. Unref WebSocket connection
3. Free ReceiverEntry structure
```

#### `get_string_from_json_object()`
**Lines: 634-649**

**Purpose:** Convert JSON object to string

**Flow:**
```
1. Create JsonNode from JsonObject
2. Create JsonGenerator
3. Generate JSON string
4. Cleanup generator and node
5. Return string (caller must free)
```

#### `bus_watch_cb()`
**Lines: 174-206**

**Purpose:** Monitor GStreamer bus for errors/warnings

**Flow:**
```
1. Called when message appears on bus
2. Handle GST_MESSAGE_ERROR:
   - Print error message
   - Print debug info
3. Handle GST_MESSAGE_WARNING:
   - Print warning message
   - Print debug info
4. Return G_SOURCE_CONTINUE (keep watching)
```

#### `update_availability()`
**Lines: 165-172**

**Purpose:** Rate limiting for connections

**Flow:**
```
1. Run in separate thread
2. Sleep for waiting_period (5 seconds)
3. Set available = true
4. Repeat forever
```

**Purpose:**
- Prevents connection flooding
- Gives time for previous client setup
- Avoids resource exhaustion

#### `exit_sighandler()`
**Lines: 651-667**

**Purpose:** Handle Ctrl+C gracefully

**Flow:**
```
1. Catch SIGINT or SIGTERM
2. Stop main pipeline
3. Unref pipeline
4. Quit main loop
5. Return TRUE (signal handled)
```

---

## Detailed Execution Flow

### Startup Sequence

```
┌────────────────────────────────────────────────────────────┐
│ 1. PROGRAM START                                           │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ 2. Parse Arguments                                         │
│    ./Vadd --device=/dev/video0 --bitrate=6000 ...         │
│    ├─ device = /dev/video0                                │
│    ├─ bitrate = 6000 kbps                                 │
│    ├─ fps = 60                                            │
│    ├─ resolution = 1920x1080                              │
│    ├─ codec = h264                                        │
│    └─ udp_ip:port = 192.168.25.90:5001                   │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ 3. Build Encoder String                                   │
│    "omxh264enc target-bitrate=6000 ... ! h264parse        │
│     ! rtph264pay mtu=1400 ! application/x-rtp,            │
│     media=video,encoding-name=H264,payload=96"            │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ 4. Build Pipeline String                                  │
│    "v4l2src device=/dev/video0 do-timestamp=false         │
│     io-mode=4 ! video/x-raw,width=1920,height=1080,       │
│     framerate=60/1,format=NV12 ! queue !                  │
│     <encoder> ! tee name=t t. ! queue !                   │
│     udpsink clients=192.168.25.90:5001"                   │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ 5. Create Pipeline                                         │
│    webrtc_pipeline = gst_parse_launch(pipeline_string)    │
│                                                            │
│    Pipeline Structure:                                     │
│    ┌──────────────────────────────────────────────────┐  │
│    │ v4l2src                                          │  │
│    │   ↓                                              │  │
│    │ video/x-raw,1920x1080@60fps,NV12                │  │
│    │   ↓                                              │  │
│    │ queue                                            │  │
│    │   ↓                                              │  │
│    │ omxh264enc (hardware encoder)                   │  │
│    │   ↓                                              │  │
│    │ h264parse                                        │  │
│    │   ↓                                              │  │
│    │ rtph264pay (packetize into RTP)                 │  │
│    │   ↓                                              │  │
│    │ application/x-rtp (RTP stream)                  │  │
│    │   ↓                                              │  │
│    │ tee (split stream)                              │  │
│    │   ├→ queue → udpsink (192.168.25.90:5001)      │  │
│    │   └→ (dynamic branches added for clients)       │  │
│    └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ 6. Get Tee Element Reference                              │
│    video_tee = gst_bin_get_by_name(pipeline, "t")        │
│    (Needed for dynamic branching)                         │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ 7. Add Bus Watch                                          │
│    bus = gst_pipeline_get_bus(webrtc_pipeline)           │
│    gst_bus_add_watch(bus, bus_watch_cb, NULL)            │
│    (Monitor for errors/warnings)                          │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ 8. Start Pipeline                                         │
│    gst_element_set_state(webrtc_pipeline, PLAYING)       │
│                                                            │
│    Camera starts capturing                                │
│    Encoder starts encoding                                │
│    UDP stream starts sending                              │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ 9. Create Receiver Hash Table                             │
│    receiver_entry_table = g_hash_table_new_full(...)     │
│    (Stores all connected clients)                         │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ 10. Create HTTP/WebSocket Server                          │
│     soup_server = soup_server_new(...)                   │
│     soup_server_add_handler("/", soup_http_handler)      │
│     soup_server_add_websocket_handler("/ws",             │
│         soup_websocket_handler)                           │
│     soup_server_listen_all(8080)                         │
│                                                            │
│     Server listening on:                                  │
│     - http://192.168.25.90:8080/ (HTML page)             │
│     - ws://192.168.25.90:8080/ws (WebSocket)             │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ 11. Register Signal Handlers                              │
│     g_unix_signal_add(SIGINT, exit_sighandler)           │
│     g_unix_signal_add(SIGTERM, exit_sighandler)          │
│     (Handle Ctrl+C gracefully)                            │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ 12. Start Availability Thread                             │
│     std::thread async_thread(update_availability)         │
│     (Rate limiting for connections)                       │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ 13. Run Main Loop                                         │
│     g_main_loop_run(mainloop)                            │
│                                                            │
│     BLOCKS HERE - Processing events:                      │
│     - WebSocket connections                               │
│     - GStreamer messages                                  │
│     - Signal handlers                                     │
│     - Timer callbacks                                     │
│                                                            │
│     Continues until Ctrl+C                                │
└────────────────────────────────────────────────────────────┘
```

### Client Connection Flow

```
┌────────────────────────────────────────────────────────────┐
│ CLIENT OPENS BROWSER                                       │
│ http://192.168.25.90:8080/                                │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ soup_http_handler()                                        │
│ - Serve index.html                                        │
│ - Browser displays page with video element                │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ BROWSER JAVASCRIPT EXECUTES                                │
│ - Creates RTCPeerConnection                               │
│ - Opens WebSocket: ws://192.168.25.90:8080/ws            │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ soup_websocket_handler()                                   │
│ 1. Check available flag                                   │
│ 2. Set available = false (block other connections)        │
│ 3. Get client IP: 192.168.25.68                          │
│ 4. Call create_receiver_entry(connection, "192.168.25.68")│
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ create_receiver_entry()                                    │
│                                                            │
│ 1. Create client bin:                                     │
│    ┌─────────────────────────────────────────┐           │
│    │ queue → webrtcbin                       │           │
│    └─────────────────────────────────────────┘           │
│                                                            │
│ 2. Add bin to main pipeline                               │
│                                                            │
│ 3. Request source pad from tee:                           │
│    Main Pipeline:                                         │
│    tee ─┬→ udpsink                                       │
│         ├→ Client1 bin                                    │
│         └→ (new) Client2 bin ← JUST ADDED               │
│                                                            │
│ 4. Link tee → client bin                                 │
│                                                            │
│ 5. Register callbacks:                                    │
│    - on-negotiation-needed                                │
│    - on-ice-candidate                                     │
│                                                            │
│ 6. Start client pipeline (PLAYING state)                 │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ WEBRTCBIN TRIGGERS: on-negotiation-needed                 │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ on_negotiation_needed_cb()                                 │
│ - Call webrtcbin's create-offer signal                    │
│ - Promise created for async operation                     │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ on_offer_created_cb()                                      │
│                                                            │
│ 1. Extract SDP offer from promise                         │
│    v=0                                                    │
│    o=- 1118777725134491962 0 IN IP4 0.0.0.0              │
│    s=-                                                    │
│    m=video 9 UDP/TLS/RTP/SAVPF 96                        │
│    a=rtpmap:96 H264/90000                                │
│    a=ice-ufrag:a4auOfmH3y+VLl2LYSH5MN35io+hUJh0         │
│    ...                                                    │
│                                                            │
│ 2. Set as local description on webrtcbin                  │
│                                                            │
│ 3. Create JSON message:                                   │
│    {                                                      │
│      "type": "offer",                                     │
│      "sdp": "<full_sdp_string>"                          │
│    }                                                      │
│                                                            │
│ 4. Send via WebSocket to browser                         │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ BROWSER RECEIVES OFFER                                     │
│ - JavaScript: pc.setRemoteDescription(offer)              │
│ - Create answer: pc.createAnswer()                        │
│ - Set local description: pc.setLocalDescription(answer)   │
│ - Send answer via WebSocket                               │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ soup_websocket_message_cb()                                │
│ Message type: "answer"                                    │
│                                                            │
│ 1. Parse JSON to extract SDP answer                       │
│ 2. Parse SDP string into GstSDPMessage                    │
│ 3. Create GstWebRTCSessionDescription                     │
│ 4. Set as remote description:                             │
│    g_signal_emit_by_name(webrtcbin,                      │
│        "set-remote-description", answer)                  │
│                                                            │
│ Now both sides know media parameters!                     │
└────────────────────────────────────────────────────────────┘
                          │
                ┌─────────┴─────────┐
                │                   │
                ▼                   ▼
    ┌────────────────────┐ ┌────────────────────┐
    │ SERVER discovers   │ │ BROWSER discovers  │
    │ ICE candidates     │ │ ICE candidates     │
    └────────────────────┘ └────────────────────┘
                │                   │
                ▼                   ▼
    ┌────────────────────┐ ┌────────────────────┐
    │on_ice_candidate_cb │ │ soup_websocket_    │
    │                    │ │ message_cb()       │
    │ Send to browser ───┼─┤ Receive from       │
    │                    │ │ browser            │
    └────────────────────┘ └────────────────────┘
                │                   │
                └─────────┬─────────┘
                          ▼
┌────────────────────────────────────────────────────────────┐
│ ICE NEGOTIATION COMPLETE                                   │
│ - Best connection path found                              │
│ - DTLS handshake performed                                │
│ - SRTP keys exchanged                                     │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ VIDEO STREAMING ACTIVE                                     │
│                                                            │
│ Main Pipeline:                                             │
│   v4l2src → encoder → tee ─┬→ udpsink (192.168.25.90)    │
│                            └→ queue → webrtcbin           │
│                                          ↓                 │
│                                    [SRTP packets]          │
│                                          ↓                 │
│                                      INTERNET              │
│                                          ↓                 │
│                                       BROWSER              │
│                                          ↓                 │
│                                    Video plays!            │
└────────────────────────────────────────────────────────────┘
```

### Data Flow During Streaming

```
┌─────────────────────────────────────────────────────────────┐
│                    VIDEO FRAME CAPTURE                       │
└─────────────────────────────────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            │               │               │
            ▼               ▼               ▼
      [Frame 1]       [Frame 2]       [Frame 3]
            │               │               │
            └───────────────┴───────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ v4l2src (camera driver)                                     │
│ - Captures raw frames from /dev/video0                      │
│ - Format: NV12 (YUV 4:2:0)                                 │
│ - Size: 1920x1080                                          │
│ - Rate: 60 fps                                             │
│ - Output: Raw video buffer                                  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ queue                                                        │
│ - Buffers frames                                            │
│ - Decouples capture from encoding                           │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ omxh264enc (hardware encoder)                               │
│ - Compresses raw frames                                     │
│ - Target bitrate: 6000 kbps                                │
│ - CBR (Constant Bit Rate)                                   │
│ - GOP length: 10 frames                                     │
│ - Output: H.264 NAL units                                   │
│                                                              │
│ Example frame sizes:                                        │
│ - I-frame: ~50 KB                                          │
│ - P-frame: ~10 KB                                          │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ h264parse                                                    │
│ - Parses H.264 bitstream                                    │
│ - Identifies frame boundaries                               │
│ - Extracts SPS/PPS headers                                  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ rtph264pay                                                   │
│ - Packetizes H.264 into RTP packets                         │
│ - MTU: 1400 bytes                                           │
│ - Adds RTP headers                                          │
│ - Sequence numbers, timestamps                              │
│                                                              │
│ RTP Packet Structure:                                       │
│ [RTP Header][H.264 Payload]                                │
│  12 bytes    ~1388 bytes                                    │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ tee (duplicates stream)                                     │
│ - Creates multiple output branches                          │
│ - Each branch gets same data                                │
└─────────────────────────────────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            │               │               │
            ▼               ▼               ▼
   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
   │   udpsink    │ │  Client 1    │ │  Client 2    │
   │              │ │  queue       │ │  queue       │
   │ 192.168.     │ │      ↓       │ │      ↓       │
   │ 25.90:5001   │ │  webrtcbin   │ │  webrtcbin   │
   └──────────────┘ └──────────────┘ └──────────────┘
            │               │               │
            ▼               ▼               ▼
    [UDP packets]   [WebRTC/SRTP]   [WebRTC/SRTP]
            │               │               │
            ▼               ▼               ▼
    Recording/      Browser 1        Browser 2
    Processing
```

**RTP Packet Flow to Client:**
```
RTP Packet → queue (buffering) → webrtcbin
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
              DTLS encrypt    Add SRTP auth    Bundle streams
                    │                 │                 │
                    └─────────────────┴─────────────────┘
                                      │
                                      ▼
                            Network (UDP/TCP/TURN)
                                      │
                                      ▼
                                   Browser
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
            DTLS decrypt    Verify SRTP    De-bundle
                    │                 │                 │
                    └─────────────────┴─────────────────┘
                                      │
                                      ▼
                            Jitter buffer (smooth playback)
                                      │
                                      ▼
                            H.264 decoder
                                      │
                                      ▼
                            Video rendering
```

### Client Disconnection Flow

```
┌────────────────────────────────────────────────────────────┐
│ USER CLOSES BROWSER TAB / NAVIGATES AWAY                   │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ WebSocket connection closed                                │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ soup_websocket_closed_cb()                                 │
│ 1. Look up ReceiverEntry from connection                  │
│ 2. Store reference to receiver_entry_table                │
│ 3. Add probe to tee_src_pad:                             │
│    gst_pad_add_probe(                                     │
│        tee_src_pad,                                       │
│        GST_PAD_PROBE_TYPE_BLOCK_DOWNSTREAM,              │
│        pad_probe_cb)                                      │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ PROBE BLOCKS DATA FLOW                                     │
│                                                            │
│ Main pipeline continues:                                   │
│ v4l2src → encoder → tee ─┬→ udpsink (still flowing)      │
│                          ├→ Client1 (still flowing)       │
│                          └→ Client2 (BLOCKED) ←           │
│                                     ↑                      │
│                              Probe active here            │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ pad_probe_cb() TRIGGERS                                    │
│ 1. Remove blocking probe                                  │
│ 2. Add EOS event probe to sink_pad                        │
│ 3. Send EOS event downstream:                             │
│    gst_pad_send_event(sink_pad, gst_event_new_eos())    │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ EOS EVENT FLOWS THROUGH PIPELINE                          │
│                                                            │
│ Client2 sub-pipeline:                                     │
│ tee → queue → webrtcbin                                   │
│         │         │                                       │
│         ▼         ▼                                       │
│    (flush)   (flush)                                      │
│         │         │                                       │
│         └────┬────┘                                       │
│              ▼                                            │
│          EOS event                                        │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ event_probe_cb() TRIGGERS                                  │
│                                                            │
│ Step 1: Verify EOS event arrived                          │
│ Step 2: Remove EOS probe                                  │
│ Step 3: Unlink pads                                       │
│    gst_pad_unlink(tee_src_pad, sink_pad)                 │
│    gst_element_release_request_pad(tee, tee_src_pad)     │
│                                                            │
│ Pipeline now:                                              │
│ v4l2src → encoder → tee ─┬→ udpsink                       │
│                          ├→ Client1                       │
│                          └→ (disconnected)                │
│                                                            │
│ Client2 bin:                                               │
│ queue → webrtcbin (orphaned, not connected to tee)       │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ Step 4: Stop Client Pipeline                              │
│    gst_element_set_state(pipeline, GST_STATE_NULL)       │
│    gst_element_get_state(pipeline, &state, NULL, 0)      │
│                                                            │
│ State changes:                                             │
│ PLAYING → PAUSED → READY → NULL                          │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ Step 5: Remove from Main Pipeline                         │
│    gst_bin_remove(webrtc_pipeline, client_bin)           │
│                                                            │
│ Main pipeline structure cleaned                           │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ Step 6: Free GStreamer Objects                            │
│    if (GST_IS_OBJECT(webrtcbin))                         │
│        gst_object_unref(webrtcbin)                        │
│    if (GST_IS_OBJECT(queue))                             │
│        gst_object_unref(queue)                            │
│    if (GST_IS_OBJECT(pipeline))                          │
│        gst_object_unref(pipeline)                         │
│                                                            │
│ Reference counts:                                          │
│ webrtcbin: 1 → 0 (freed)                                 │
│ queue: 1 → 0 (freed)                                      │
│ pipeline: 1 → 0 (freed)                                   │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ Step 7: Remove from Hash Table                            │
│    g_hash_table_remove(r_table, connection)               │
│    └→ Triggers destroy_receiver_entry()                   │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ destroy_receiver_entry()                                   │
│    g_object_unref(connection)  // WebSocket connection    │
│    g_slice_free(ReceiverEntry) // Free structure          │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ CLEANUP COMPLETE                                           │
│                                                            │
│ - All memory freed                                         │
│ - All references released                                  │
│ - Main pipeline still running for other clients           │
│ - Server ready for new connections                         │
└────────────────────────────────────────────────────────────┘
```

---

## Key Concepts Explained

### 1. Tee Element (Dynamic Branching)

**What it does:** Duplicates stream to multiple outputs

**Why needed:** 
- One camera → Multiple clients
- Each client gets same video
- Clients independent (one disconnect doesn't affect others)

**How it works:**
```
Input: Single RTP stream
       │
       ▼
    ┌─────┐
    │ tee │
    └─────┘
       │
   ┌───┼───┬───┐
   │   │   │   │
   ▼   ▼   ▼   ▼
  UDP C1  C2  C3

- Each output is a "request pad" (created dynamically)
- Data copied to all outputs
- Zero-copy when possible (shared buffers)
```

### 2. GStreamer Bins

**What it is:** Container for multiple elements

**Why useful:**
- Group related elements
- Treat group as single element
- Easier management

**Example:**
```cpp
// Create bin
GstElement *client_bin = gst_bin_new("client");

// Add elements to bin
gst_bin_add_many(GST_BIN(client_bin), 
    queue, webrtcbin, NULL);

// Link elements inside bin
gst_element_link(queue, webrtcbin);

// Add entire bin to pipeline
gst_bin_add(GST_BIN(main_pipeline), client_bin);
```

### 3. Ghost Pads

**What it is:** Exposes internal pad externally

**Why needed:**
- Link to elements inside bin from outside
- Bin acts as single element with visible pads

**Example:**
```cpp
// Get internal pad
GstPad *sink_pad = gst_element_get_static_pad(queue, "sink");

// Create ghost pad on bin
gst_element_add_pad(client_bin, 
    gst_ghost_pad_new("sink", sink_pad));

// Now can link to bin from outside
gst_pad_link(tee_src_pad, bin_sink_pad);
```

### 4. Pad Probes

**What it is:** Callback on data flow

**Types:**
- **BLOCK:** Stop data flow
- **EVENT_DOWNSTREAM:** Monitor events
- **DATA:** Inspect/modify data

**Why needed:**
- Safe cleanup (flush data first)
- Avoid deadlocks
- Synchronization

### 5. WebRTC Signaling

**What it is:** Exchange of connection info before streaming

**Required information:**
1. **SDP (Session Description Protocol)**
   - Media capabilities (codecs)
   - Network transport
   - Encryption params

2. **ICE Candidates**
   - Possible connection paths
   - IP addresses and ports
   - Connection priorities

**Flow:**
```
Server                          Client
  │                               │
  │  1. SDP Offer                │
  │ ────────────────────────────> │
  │                               │
  │  2. SDP Answer               │
  │ <──────────────────────────── │
  │                               │
  │  3. ICE Candidates           │
  │ <═══════════════════════════> │
  │  (multiple exchanges)         │
  │                               │
  │  4. Connection established   │
  │ <═══════════════════════════> │
  │                               │
  │  5. Media flows              │
  │ ═════════════════════════════>│
```

### 6. Reference Counting

**GLib uses reference counting for memory management**

**Rules:**
- `g_object_new()` → refcount = 1
- `g_object_ref()` → refcount++
- `g_object_unref()` → refcount--
- When refcount = 0 → object freed

**Common pattern:**
```cpp
// Create (refcount = 1)
GstElement *element = gst_element_factory_make("queue", NULL);

// Add to bin (bin takes reference, refcount = 2)
gst_bin_add(bin, element);

// You still own your reference
// Must unref when done
g_object_unref(element);  // refcount = 1

// When bin is destroyed, it unrefs → refcount = 0 → freed
```

---

## Performance Considerations

### 1. Hardware Encoding
- Uses `omxh264enc` (hardware encoder)
- Much faster than software encoding
- Lower CPU usage
- Critical for real-time streaming

### 2. Queue Configuration
```cpp
g_object_set(queue, 
    "max-size-buffers", 100,    // Max 100 frames
    "leaky", 2,                 // Drop old frames (not new)
    "flush-on-eos", TRUE);      // Clean shutdown
```

**Tuning:**
- Smaller buffer → Lower latency, more drops
- Larger buffer → Higher latency, fewer drops

### 3. Zero-Copy Optimization
```cpp
v4l2src io-mode=4  // DMABUF mode
```
- Avoids copying video frames
- Direct memory sharing between driver and encoder
- Significant performance improvement

### 4. Rate Limiting
- 5-second delay between connections
- Prevents resource exhaustion
- Allows time for setup

---

## Common Issues and Solutions

### Issue 1: "Level is too low"
**Cause:** Encoder level doesn't match resolution/fps
**Solution:** Specify level parameter in encoder

### Issue 2: "No real random source"
**Cause:** Can't access `/dev/random` for DTLS keys
**Solution:** Install `haveged` or `rng-tools`

### Issue 3: GLib-CRITICAL refcount errors
**Cause:** Double-free or incorrect reference counting
**Solution:** Ensure proper ref/unref balance

### Issue 4: Client gets black screen
**Causes:**
- ICE negotiation failed
- Firewall blocking
- Codec mismatch
**Debug:** Check browser console, enable GST_DEBUG

### Issue 5: High CPU usage
**Causes:**
- Software encoding instead of hardware
- Too many clients
- High resolution/fps
**Solution:** Verify hardware encoder, limit clients, reduce quality

---

## Summary

This program is a **production-ready WebRTC video streaming server** that:

1. **Captures** video from hardware camera
2. **Encodes** using hardware acceleration
3. **Duplicates** stream to multiple destinations
4. **Serves** web page with video player
5. **Negotiates** WebRTC connections dynamically
6. **Streams** encrypted video to browsers
7. **Cleans up** gracefully when clients disconnect

**Key strengths:**
- Efficient (hardware encoding, zero-copy)
- Scalable (multiple clients supported)
- Robust (proper error handling, clean shutdown)
- Standard-compliant (WebRTC, RTP, SRTP)

**Use cases:**
- Live streaming
- Remote monitoring
- Video conferencing
- Surveillance systems
- Remote control/telepresence
