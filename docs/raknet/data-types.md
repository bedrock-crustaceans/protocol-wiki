---
mentions:
    - akashic-records-of-the-abyss
    - theaddonn
    - MisledWater79
prev:
    text: 'Learning RakNet'
    link: '/raknet/start'
next:
    text: 'Frames'
    link: '/raknet/frames'
---

# Data Types

Data types used for Minecraft Bedrock's RakNet implimentation.
Explains what types are used, their sizes, ranges, and any other info on the type.

## Basic Data Types

The following shows ranges of unsigned types. All types used are encoded as little-endian.

|               | Byte Size | Range              | Notes                                       |
| ------------- | --------- | ------------------ | ------------------------------------------- |
| Boolean (u8)  | 1         | 0 to 1             | Written as a Byte but only uses top BIT.    |
| Byte (u8)     | 1         | 0 to 255           | 8 BITS.                                     |
| Short (u16)   | 2         | 0 to 65,635        |                                             |
| Int24 (u24)   | 4         | 0 to 16,777,215    | Written as a Int but only uses top 3 Bytes. |
| Int (u32)     | 4         | 0 to 4,294,967,295 |                                             |
| Long (u64)    | 8         | 0 to (2^64)−1      |                                             |

## Structure Data Types

These are structure types this implementation uses.

|                            | Byte Size | Notes                                                                                                        |
| -------------------------- | --------- | ------------------------------------------------------------------------------------------------------------ |
| [Magic](#magic)            | 16        | This is also known as RakNet's `OFFLINE_MESSAGE_DATA_ID`.                                                    |
| [Address](#address)        | 7-29      | Size is dependent on IP version. IPv4 is 7 bytes, IPv6 is 29 bytes.                                          |
| String                     | 2-65,535  | Has one unsigned short defining the size, followed by that many bytes.                                       |
| [Frame](/raknet/frames.md) | 3-8,214   | Has one byte defining flags, along with an unsigned short defining size in BITS, followed by that many bits. |

## Magic

Magic(aka offline message data) is used only and always in offline packets. The reason this is used is for a couple reason, security, and to tell if it's just an offline packet. Here are a couple comments on this from the [offical repo](https://github.com/facebookarchive/RakNet/).

```c++
// Make sure highest bit is 0, so isValid in DatagramHeaderFormat is false
static const unsigned char OFFLINE_MESSAGE_DATA_ID[16]={0x00,0xFF,0xFF,0x00,0xFE,0xFE,0xFE,0xFE,0xFD,0xFD,0xFD,0xFD,0x12,0x34,0x56,0x78};
...
// The reason for all this is that the reliability layer has no way to tell between offline messages that arrived late for a player that is now connected,
// and a regular encoding. So I insert OFFLINE_MESSAGE_DATA_ID into the stream, the encoding of which is essentially impossible to hit by chance
```

Magic is always these bytes `00ffff00fefefefefdfdfdfd12345678`. It seems like it was just for some format and is just hard to obtain by just pure chance.

## Address

An address type is split into two. Both start with a Byte, this is the protocol and is either a `4` for IPv4 or a `6` for IPv6.

IPv4 has 4 Bytes for the IP followed by a unsigned short for the port.

IPv6 has a unsigned short for the address family, an unsigned short for the port, 4 Bytes for the flow info, 16 Bytes for the IP, and 4 bytes for the scope ID.

### Random Info

The original RakNet implementation is honestly a mess. But it is also not just for minecraft, it's a multiplayer game network engine. Hence why it has a lot of seemingly unessacary code. Which is why plenty of people make their own implementations that are made just for Minecraft.

Here is an sample of seemingly unessacary code. This is used to check if a packet is an offline packet.
```c++
// The reason for all this is that the reliability layer has no way to tell between offline messages that arrived late for a player that is now connected,
// and a regular encoding. So I insert OFFLINE_MESSAGE_DATA_ID into the stream, the encoding of which is essentially impossible to hit by chance
if (length <=2)
{
    *isOfflineMessage=true;
}
else if (
    ((unsigned char)data[0] == ID_UNCONNECTED_PING ||
    (unsigned char)data[0] == ID_UNCONNECTED_PING_OPEN_CONNECTIONS) &&
    length >= sizeof(unsigned char) + sizeof(RakNet::Time) + sizeof(OFFLINE_MESSAGE_DATA_ID))
{
    *isOfflineMessage=memcmp(data+sizeof(unsigned char) + sizeof(RakNet::Time), OFFLINE_MESSAGE_DATA_ID, sizeof(OFFLINE_MESSAGE_DATA_ID))==0;
}
else if ((unsigned char)data[0] == ID_UNCONNECTED_PONG && (size_t) length >= sizeof(unsigned char) + sizeof(RakNet::TimeMS) + RakNetGUID::size() + sizeof(OFFLINE_MESSAGE_DATA_ID))
{
    *isOfflineMessage=memcmp(data+sizeof(unsigned char) + sizeof(RakNet::Time) + RakNetGUID::size(), OFFLINE_MESSAGE_DATA_ID, sizeof(OFFLINE_MESSAGE_DATA_ID))==0;
}
else if (
    (unsigned char)data[0] == ID_OUT_OF_BAND_INTERNAL	&&
    (size_t) length >= sizeof(MessageID) + RakNetGUID::size() + sizeof(OFFLINE_MESSAGE_DATA_ID))
{
    *isOfflineMessage=memcmp(data+sizeof(MessageID) + RakNetGUID::size(), OFFLINE_MESSAGE_DATA_ID, sizeof(OFFLINE_MESSAGE_DATA_ID))==0;
}
else if (
    (
    (unsigned char)data[0] == ID_OPEN_CONNECTION_REPLY_1 ||
    (unsigned char)data[0] == ID_OPEN_CONNECTION_REPLY_2 ||
    (unsigned char)data[0] == ID_OPEN_CONNECTION_REQUEST_1 ||
    (unsigned char)data[0] == ID_OPEN_CONNECTION_REQUEST_2 ||
    (unsigned char)data[0] == ID_CONNECTION_ATTEMPT_FAILED ||
    (unsigned char)data[0] == ID_NO_FREE_INCOMING_CONNECTIONS ||
    (unsigned char)data[0] == ID_CONNECTION_BANNED ||
    (unsigned char)data[0] == ID_ALREADY_CONNECTED ||
    (unsigned char)data[0] == ID_IP_RECENTLY_CONNECTED) &&
    (size_t) length >= sizeof(MessageID) + RakNetGUID::size() + sizeof(OFFLINE_MESSAGE_DATA_ID))
{
    *isOfflineMessage=memcmp(data+sizeof(MessageID), OFFLINE_MESSAGE_DATA_ID, sizeof(OFFLINE_MESSAGE_DATA_ID))==0;
}
else if (((unsigned char)data[0] == ID_INCOMPATIBLE_PROTOCOL_VERSION&&
    (size_t) length == sizeof(MessageID)*2 + RakNetGUID::size() + sizeof(OFFLINE_MESSAGE_DATA_ID)))
{
    *isOfflineMessage=memcmp(data+sizeof(MessageID)*2, OFFLINE_MESSAGE_DATA_ID, sizeof(OFFLINE_MESSAGE_DATA_ID))==0;
}
else
{
    *isOfflineMessage=false;
}
```

#### Resources

[[RakNet Repo](https://github.com/facebookarchive/RakNet/)]: Information on Magic along with code snippets

[[wiki.vg/Minecraft Wiki](https://minecraft.wiki/w/Minecraft_Wiki:Projects/wiki.vg_merge/Raknet_Protocol)]: Information on data types
