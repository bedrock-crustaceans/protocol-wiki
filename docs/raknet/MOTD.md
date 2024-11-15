# Message of the Day (MOTD) Protocol Documentation

## Overview

A RakNet server, by default, listens on port `19132` for incoming [Unconnected Pings](#unconnected-ping). Once a valid ping is sent, the server sends back a [Unconnected Pong](#unconnected-pong). This contains the server's [MOTD](#motd-format), along with other server info. All this ensures that a client receives the details of the server before a connection is established.

## Packet Structure

### Unconnected Ping

When a client wants to display the server it sends an unconnected ping

- **Packet ID (1 byte):** `0x01`
- **Timestamp (8 bytes, Big-Endian):** The time since start of the client. Just setting it to current unix timestamp works fine
- **Magic Offline Message ID (16 bytes):**
  ```
  [
      0x00, 0xFF, 0xFF, 0x00, 
      0xFE, 0xFE, 0xFE, 0xFE, 
      0xFD, 0xFD, 0xFD, 0xFD, 
      0x12, 0x34, 0x56, 0x78
  ]
  ```
- **GUID (8 bytes, Big-Endian):** Unique identifier for the client.

#### Validation Criteria

1. **Packet Size:**  
   The total size must be exactly 33 bytes.
2. **Packet ID:**  
   Must strictly be `0x01`.
3. **Timestamp:**  
   It is just a timestamp of uptime, but we can check it is not infront of unix timestamp
4. **Magic Offline Message ID:**  
   Must match the predefined 16-byte sequence.
5. **GUID:**  
   It doesn't really need to be checked, but it is there

### Unconnected Pong

In response to a valid unconnected ping, the server dispatches an unconnected pong packet containing the MOTD and additional server details.

- **Packet ID (1 byte):** `0x1C` (Denotes Unconnected Pong)
- **Timestamp (8 bytes, Big-Endian):** The time since start of the server. Just setting it to current unix timestamp works fine
- **Server GUID (8 bytes, Big-Endian):** Unique server identifier.
- **Magic Offline Message ID (16 bytes):** Identical to the one used in the ping.
- **MOTD Length (2 bytes, Big-Endian):** Specifies the length of the MOTD string.
- **MOTD (Variable Length):** The actual Message of the Day in byte format.

## MOTD Generation

The MOTD is what shows up in the server list, it is what people see before joining the server.

### MOTD Format

The MOTD follows a semicolon-separated (`;`) format comprising various fields, semi colons can be escaped with a backslash (`\`)
```
<Edition>;<MOTD>;<Protocol Version>;<Game Version>;<Player Count>;<Max Player Count>;<Server GUID>;<World Name>;<Game Mode>;<Nintendo Limited>;<IpV4Port>;<IpV6Port>;<Extra Data>;
```

#### Field Breakdown & Data Types

- **Edition:** string  
  Specifies the game edition (e.g., `MCPE` for Minecraft Pocket Edition or `MCEE` for Minecraft Education Edition). Both seem to work with windows minecraft.
- **MOTD:** string  
  The actual message that is displayed, it can have Minecraft colors.  
- **Protocol Version:** uint16  
  Numeric identifier indicating the game protocol version (e.g., `748`).  
- **Game Version:** string  
  Denotes the server's game version (e.g., `1.20.40`, or on CubeCraft: `1`).  
- **players online:** uint32  
  Current number of active players (e.g., `10`).  
- **players max:** uint32  
  Maximum number of players allowed on the server (e.g., `100`).  
- **server id:** uint64  
  Unique identifier for the server (the server GUID but as a number).  
- **level name:** string  
  Name of the game world (e.g., `world`, some docs call this submotd).  
- **gamemode:** string  
  Current game mode (e.g., `Survival`, `Creative`, `Adventure`).  
- **NintendoLimited:** boolean  
  Returns the status of Nintendo's limitation to join (0 or 1).  
- **port v4:** uint16  
  The port of the server for IPv4, useful for servers that have both IPv4 and IPv6.  
- **port v6:** uint16  
  The port of the server for IPv6, useful for servers that have both IPv4 and IPv6.  
- **Extra Data:**  
  Ends with `0` to signify the conclusion of the MOTD string.  

## Code Examples
::: details Code Examples
::: code-group

```rust [src/main.rs]
use std::io;
use std::net::UdpSocket;
use std::thread;
use std::time::{SystemTime, UNIX_EPOCH};

const MAGIC_OFFLINE_MESSAGE_ID: [u8; 16] = [
    0x00, 0xFF, 0xFF, 0x00, 
    0xFE, 0xFE, 0xFE, 0xFE, 
    0xFD, 0xFD, 0xFD, 0xFD, 
    0x12, 0x34, 0x56, 0x78,
];

fn gen_motd(server_guid: u64, listen_port: u16) -> String {
    let edition = "MCPE";
    let motd = "Welcome to my Minecraft server!";
    let protocol_version = "748";
    let version = "1.20.40";
    let player_count = "10";
    let max_player_count = "100";
    let world_name = "world";
    let gamemode = "Survival";
    let nintendo_limited = "1";
    let port = listen_port.to_string();
    let server_guid = server_guid.to_string();

    return format!(
        "{};{};{};{};{};{};{};{};{};{};{};{};0;",
        edition,
        motd,
        protocol_version,
        version,
        player_count,
        max_player_count,
        server_guid,
        world_name,
        gamemode,
        nintendo_limited,
        port,
        port
    );
}

fn check_valid_ping(buf: &[u8], size: usize) -> bool {
    // Check length of buffer. If we just do .len() it will be 1024 not 33, so we have to pass the size
    if size != 33 {
        return false;
    }

    if buf[0] != 0x01 {
        return false;
    }

    let time: [u8; 8] = buf[1..9].try_into().unwrap();
    let current_time = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .expect("Time went backwards")
        .as_secs();
    let time = u64::from_be_bytes(time);
    if time > current_time {
        return false;
    }

    let magic_offline_message_id: [u8; 16] = buf[9..25].try_into().unwrap();
    if magic_offline_message_id != MAGIC_OFFLINE_MESSAGE_ID {
        return false;
    }

    let guid: [u8; 8] = buf[25..33].try_into().unwrap();
    let guid = u64::from_be_bytes(guid);

    println!("Received packet with guid: {}, time: {}", guid, time);

    return true;
}

fn write_pong(server_guid: u64, motd: String) -> Vec<u8> {
    let current_time = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .expect("Time went backwards")
        .as_secs();

    let mut response = Vec::new();
    response.push(0x1C); // Packet ID for Unconnected Pong
    response.extend(&current_time.to_be_bytes());
    response.extend(&server_guid.to_be_bytes());
    response.extend(&MAGIC_OFFLINE_MESSAGE_ID);

    let motd_bytes = motd.as_bytes();
    let motd_length = motd_bytes.len() as u16;
    response.extend(&motd_length.to_be_bytes());
    response.extend(motd_bytes);

    return response;
}

fn listen(listen_port: u16) {
    let socket: UdpSocket = match UdpSocket::bind(format!("127.0.0.1:{}", listen_port)) {
        Ok(s) => s,
        Err(e) => {
            eprintln!("Failed to bind to port 19132: {}", e);
            return;
        }
    };
    socket
        .set_nonblocking(true)
        .expect("Cannot set non-blocking");

    let server_guid: u64 = 1234567890123456789; // Replace with your server's GUID

    let mut buf = [0u8; 1024];

    loop {
        match socket.recv_from(&mut buf) {
            Ok((size, src)) => {
                if !check_valid_ping(&buf, size) {
                    println!("Received invalid ping");
                    continue;
                }

                // Unconnected ping received
                let motd = gen_motd(server_guid, listen_port);
                let response = write_pong(server_guid, motd);

                if let Err(e) = socket.send_to(&response, src) {
                    eprintln!("Failed to send unconnected pong: {}", e);
                } else {
                    println!(
                        "Sent unconnected pong to {}, from {}",
                        src, listen_port
                    );
                }
            }
            Err(ref e) if e.kind() == io::ErrorKind::WouldBlock => {
                // No data received, continue listening
                thread::sleep(std::time::Duration::from_millis(100));
            }
            Err(e) => {
                eprintln!("Error receiving data: {}", e);
            }
        }
    }
}

fn main() {
    listen(19132);
}
```

```ts [src/main.ts BUN]
let MAGIC_OFFLINE_MESSAGE_ID = new Buffer([0x00, 0xFF, 0xFF, 0x00, 0xFE, 0xFE, 0xFE, 0xFE, 0xFD, 0xFD, 0xFD, 0xFD, 0x12, 0x34, 0x56, 0x78]);

function genMotd(serverGuid: number, listenPort: number) {
  let edition = "MCPE";
  let motd = "Welcome to my Minecraft server!";
  let protocolVersion = "748";
  let version = "1.20.40";
  let playerCount = "10";
  let maxPlayerCount = "100";
  let worldName = "world";
  let gamemode = "Survival";
  let nintendoLimited = "1";
  let port = listenPort.toString();
  let guid = serverGuid.toString();

  let order = [
    edition,
    motd,
    protocolVersion,
    version,
    playerCount,
    maxPlayerCount,
    guid,
    worldName,
    gamemode,
    nintendoLimited,
    port,
    port,
    "0;",
  ];

  return order.join(";");
}

function checkValidPing(buf: Buffer, size: number) {
  if (size !== 33) {
    return false;
  }

  if (buf[0] !== 0x01) {
    return false;
  }

  let time = buf.subarray(1, 9);
  time = time.readBigUInt64BE();


  let magicOfflineMessageId = buf.subarray(9, 25);
  if (!magicOfflineMessageId.equals(MAGIC_OFFLINE_MESSAGE_ID)) {
    return false;
  }

  let guid = buf.subarray(25, 33);
  guid = guid.readBigUInt64BE();


  return true;
}

function writePong(serverGuid: number, motd: string) {
  const currentTime = BigInt(Math.floor(Date.now() / 1000)); // Current time in seconds

    const response = Buffer.alloc(1 + 8 + 8 + MAGIC_OFFLINE_MESSAGE_ID.length + 2 + motd.length);
    
    let offset = 0;
    response.writeUInt8(0x1C, offset); // Packet ID for Unconnected Pong
    offset += 1;

    // Write current time
    response.writeBigUInt64BE(currentTime, offset);
    offset += 8;

    // Write server GUID
    response.writeBigUInt64BE(BigInt(serverGuid), offset);
    offset += 8;

    // Write MAGIC_OFFLINE_MESSAGE_ID
    MAGIC_OFFLINE_MESSAGE_ID.copy(response, offset);
    offset += MAGIC_OFFLINE_MESSAGE_ID.length;

    // Write MOTD length and bytes
    response.writeUInt16BE(motd.length, offset);
    offset += 2;

    response.write(motd, offset);

    return response;
}

function listen(listenPort: number) {
  let socket = Bun.udpSocket({
    port: listenPort,
    socket: {
      data: (socket, buf, port, addr) => {
        let serverGuid = 1234567890123456789;

        if (!checkValidPing(buf, buf.length)) {
          console.log("Received invalid ping");
          return;
        }

        let motd = genMotd(serverGuid, listenPort);
        let response = writePong(serverGuid, motd);

        socket.send(response, port, addr);
      },
    },
  });
}

listen(19132);
```
:::
## Sources
- [Go Raknet](https://github.com/Sandertv/go-raknet/blob/master/internal/message/unconnected_ping.go#L8)<br>
- [PHP MC Query](https://github.com/xPaw/PHP-Minecraft-Query/blob/master/src/MinecraftQuery.php#L212)<br>
- [Raknet](https://github.com/facebookarchive/RakNet/blob/1a169895a900c9fc4841c556e16514182b75faf8/Source/RakPeer.cpp#L135)<br>
- [Phantom](https://github.com/jhead/phantom/blob/44056d83afbfe60b253faff01308d171ac21e8d6/internal/proto/proto.go#L23)<br>
- [Geyser](https://github.com/GeyserMC/Protocol/blob/46b4ad37b159b1fc59e45871f7572101f5ed43ab/bedrock-connection/src/main/java/org/cloudburstmc/protocol/bedrock/BedrockPong.java#L23)<br>
- [Wiki.vg](https://wiki.vg/Raknet_Protocol#Unconnected_Ping)<br>
- Wireshark 
