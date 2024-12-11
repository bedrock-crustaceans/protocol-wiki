---
mentions:
    - akashic-records-of-the-abyss
    - theaddonn
    - Misledwater79
    - Tom-Teclador
prev:
    text: 'Learning MCBE Protocol'
    link: '/info/learn'
next:
    text: 'RakNet Data Types'
    link: '/raknet/data-types'
---

# RakNet Protocol

Latest documentation on the RakNet protocol mainly for Minecraft Bedrock Edition(s). Explains packets, their data types, and their formatation.

## What is RakNet?

RakNet is a UDP transfer layer. It is mainly used in Minecraft Bedrock, but is used and can be used in other games. This document will cover RakNet as a whole, but the main focus is on the MCBE implementation.

## How does RakNet work?

As mentioned before, RakNet protocol uses UDP. This means that the [Packets]() that are sent, might not always make it. This doesn't matter for things like [Pings]() and [Pongs](), but when dealing with more important information, we want to make sure the clients(players) get that data. To combat this, RakNet uses [Frame Set]() packets. These split data up if it's too big and also keeps track of what data did or didn't make it to the recipient. If it didn't recieve it, the sender will resend it. This uses [ACK]() & [NACK]() packets which the receiver of a [Frame Set]() packet sends back to either say it got this part of the data or is missing this part of the data.

## Handshake sequence

This is the sequence that an client(s) and the server go through to verify and connect said client(s). (The following will be in the perspective of one client and one sevrer)

---

> [!NOTE]
> Unconnected client(s) will be sending [Unconnected Pings]() and in responce, recieve [Unconnected Pongs]() from the server. This is how the client recieves the [MOTD]() of the server.

* The client sends an [Open Connection Request 1]()
* Server responds by sending an [Open Connection Reply 1]()
* Client sends an [Open Connection Request 2]()
* Server replys with an [Open Connection Reply 2]()

> [!NOTE]
> From here, the connection is made and we start sending packets in [Frame Sets]()
>
> Along with that, the client now sends [Connected Pings]() and in responce, recieve [Connected Pong]() from the server. No longer sending the unconnected versions of said packets. (If stopped, we have lost connection)

* Client sends a [Connection Request]()
* Server responds with a [Connection Request Accepted]()
* Lastly, client sends a [New Incoming Connection]()

---

Now, all data recieved from frame sets are [Game Packets](). The data recieved from the game packet is now delt with by the [Bedrock Protocol]() (We will go over it after RakNet).

This is the barebones of RakNet, so do not worry if you do not understand yet. Our next part is going to go over the [Data Types](/raknet/data-types.md) used by RakNet.

---

#### Resources

[[Jenkin Software RakNet Docs](http://www.jenkinssoftware.com/raknet/manual/systemoverview.html)]: Information on how RakNet works & communicates

[[wiki.vg/Minecraft Wiki](https://minecraft.wiki/w/Minecraft_Wiki:Projects/wiki.vg_merge/Raknet_Protocol)]: Information on handshake sequence
