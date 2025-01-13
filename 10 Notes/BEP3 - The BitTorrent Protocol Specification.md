---
tags:
  - bittorrent
related-to:
  - "[[Learn BitTorrent]]"
---
#bittorrent 

| related-to | [[Protocol v1 clarifications]], [[BitTorrent DHT clarifications]] |
| ---------- | ----------------------------------------------------------------- |

[BEP3](https://www.bittorrent.org/beps/bep_0003.html) describes the original, or `v1` of the BitTorrent protocol.

### ChatGPT clarifications

### Question 1

Let’s look at the BitTorrent protocol spec BEP3. Here is a fragment:

”The peer wire protocol consists of a handshake followed by a never-ending stream of length-prefixed messages. The handshake starts with character ninteen (decimal) followed by the string 'BitTorrent protocol'. The leading character is a length prefix, put there in the hope that other new protocols may do the same and thus be trivially distinguishable from each other.“
#### Answer

The BitTorrent protocol handshake described in BEP3 is designed to establish a connection between peers and ensure that both ends are compatible BitTorrent clients. Here’s a breakdown of the process:

1. **Handshake Message Structure**:

	- The handshake message starts with a single byte, 19 in decimal (or 0x13 in hexadecimal), representing the length of the protocol identifier string.

	- This byte is followed by the string "BitTorrent protocol", which is 19 bytes long. Together, these identify the protocol to the receiving peer.

2. **Purpose of the Length Prefix**:

	- The initial byte (19) acts as a length prefix, signifying that the following 19 bytes contain the protocol identifier. This approach was intended to allow future protocols to adopt a similar structure, making it easy to distinguish between BitTorrent and other protocols that might operate in the same space.

	- This allows peers to recognize BitTorrent-specific connections and helps avoid potential conflicts with other protocols that might emerge.

3. **Follow-up After the Handshake**:

	- After this initial identifier, additional handshake data is exchanged, including reserved bytes for extensions, the info_hash (identifying the torrent), and the peer ID. Once the handshake completes, both peers start exchanging regular protocol messages for data transfer.

In essence, this structured handshake is BitTorrent’s way of both signaling its protocol and laying the foundation for a secure and compatible connection between peers.

### Question 1a

Which peer starts? I understand that the same message is returned by the other peer, right?
#### Answer

In the BitTorrent protocol, any peer can initiate a connection by sending the handshake message to another peer. There isn’t a designated “starting” peer since both peers are essentially equal. However, typically:

1. **Initiation**:

	- When a peer (let’s call it Peer A) discovers another peer (Peer B) via a tracker or peer exchange (PEX), it can initiate a TCP connection and send the handshake message to Peer B.

2. **Handshake Message Exchange**:

	- Peer B, upon receiving this handshake message, checks the protocol string and verifies the info_hash to ensure it matches the torrent both are sharing.

	- If the handshake is valid, Peer B responds with its own handshake message, mirroring Peer A’s structure. This confirms compatibility and readiness to exchange data blocks for the torrent.

3. **Symmetry of Handshake**:

	- The handshake message structure is identical from both sides, so after the initial connection, both peers have exchanged the same type of message. This mutual exchange verifies the connection and establishes both peers as legitimate participants in the torrent.

In practice, once this handshake is complete, either peer can begin sending subsequent messages as part of the data exchange, like interested, not interested, or request messages.

### Question 2

Can you further explain this part of the protocol, ideally with some examples?

“Next comes the 20 byte sha1 hash of the bencoded form of the info value from the metainfo file. (This is the same value which is announced as info_hash to the tracker, only here it's raw instead of quoted here). If both sides don't send the same value, they sever the connection. The one possible exception is if a downloader wants to do multiple downloads over a single port, they may wait for incoming connections to give a download hash first, and respond with the same one if it's in their list.”
#### Answer

The section you’re asking about in the BitTorrent protocol handshake describes the info_hash field, which is critical for confirming that both peers are participating in the same torrent. Here’s a step-by-step breakdown of what’s happening:

1. **What is `info_hash`?**

	- The info_hash is a 20-byte SHA1 hash that uniquely identifies the specific torrent. It’s derived from the “info” dictionary in the .torrent file, which contains metadata about the torrent’s content, such as file names, sizes, and piece information.

	- This hash is the same one that the peer announces to a tracker when joining the swarm, helping to ensure that all peers are part of the same torrent.

2. **Handshake Process Using `info_hash`**

	- During the handshake, each peer includes this 20-byte info_hash in the message. When Peer A sends the handshake to Peer B, it includes its info_hash, and Peer B responds with a handshake containing its own info_hash.

	- Each peer then checks whether the info_hash from the other peer matches its own. If the hashes match, it confirms that both peers are sharing the same torrent and can proceed to data exchange. If the hashes don’t match, the peers terminate the connection because they are not part of the same torrent.

3. **Why This Check is Necessary**

	- This verification prevents peers from mistakenly connecting to other torrents that may be using the same port or other shared resources. The info_hash acts as a unique identifier, so both peers can be sure they are trying to share the same files.

4. **Handling Multiple Torrents on a Single Port (Exception)**

	- Sometimes, a peer may download multiple torrents but only has one open port. In this case, when another peer connects and provides an info_hash, the receiving peer may check if it has that info_hash in its list of active torrents. If so, it proceeds with the handshake as normal. If not, it severs the connection.

	- This flexibility allows the peer to manage multiple torrents efficiently without needing multiple ports, accommodating multiple incoming handshake requests.

**Example Walkthrough**

Imagine Peer A is downloading a torrent with the `info_hash` `abc123...` (20-byte SHA1 hash). Peer B, which is also downloading the same torrent, wants to connect:

1. **Peer A** initiates the handshake to **Peer B**:

	- Peer A sends: `{19, "BitTorrent protocol", 8 reserved bytes, info_hash="abc123..."}`.

2. **Peer B** receives the handshake, verifies the info_hash, and sends a handshake back:

	- Peer B sends: {19, "BitTorrent protocol", 8 reserved bytes, info_hash="abc123..."}.

3. **Both Peers Compare `info_hash` Values**:

	- Peer A checks that the `info_hash` from Peer B matches `abc123...`, confirming they’re both in the same torrent swarm.

	- If the `info_hash` didn’t match, both peers would close the connection.

This `info_hash` exchange ensures that peers can only connect if they are sharing the same torrent, helping to maintain the integrity and organization of different torrent swarms.

### Question 2a

What is “download hash” in this context?
#### Answer

In this context, “download hash” refers to the `info_hash` of a specific torrent being downloaded. It’s another way of describing the unique identifier for a torrent. Here’s how it works:

- The `info_hash` (or “download hash” in this case) is a SHA1 hash generated from the “info” dictionary in the .torrent file.

- When multiple torrents are being downloaded on a single port, a peer waits to receive an incoming handshake from another peer with a specific `info_hash`. It can then check if that info_hash (download hash) matches any torrents it’s currently downloading.

- If the received `info_hash` matches one in the peer’s list, it can proceed with the handshake; if not, it severs the connection.

This approach allows peers to manage connections for multiple torrents simultaneously while ensuring each connection is specific to a single torrent.

### Question 3

And also, please explain with example the last part of the handshake:

“After the download hash comes the 20-byte peer id which is reported in tracker requests and contained in peer lists in tracker responses. If the receiving side's peer id doesn't match the one the initiating side expects, it severs the connection.”
#### Answer

The final part of the BitTorrent handshake includes a 20-byte peer ID immediately following the `info_hash`. This peer ID uniquely identifies each peer in the network, allowing them to recognize one another and track each peer’s activity within the swarm. Here’s how this part of the handshake works and why it’s important:

**Purpose of the Peer ID**

1. **Identification**: Each BitTorrent client generates a unique peer ID for itself when it starts a new torrent session. This ID helps peers keep track of each other and manage connections.

2. **Tracker Communication**: The peer ID is reported to the tracker when a client joins the torrent. The tracker then provides a list of other peers, each with their own peer IDs, to help them connect.

3. **Verification During Handshake**: When peers establish a connection, they exchange these peer IDs to confirm each other’s identities.

**Example Walkthrough**

Imagine two peers, Peer A and Peer B, trying to connect within the same torrent:

1. **Peer A Starts the Handshake**:

	- Peer A sends the handshake message: `{19, "BitTorrent protocol", 8 reserved bytes, info_hash="abc123...", peer_id="A12345..."}`.

• This message includes Peer A’s own peer ID ("A12345..."), which is unique to Peer A.

2. **Peer B Receives the Handshake:**

	- Peer B, which also has the same info_hash for this torrent, receives the handshake and extracts Peer A’s peer ID from it.

	- Peer B then sends back its own handshake with its peer ID: {19, "BitTorrent protocol", 8 reserved bytes, info_hash="abc123...", peer_id="B67890..."}.

3. **Checking the Peer ID**:

	- Peer A receives Peer B’s handshake and verifies the peer ID "B67890..." from Peer B’s message.

	- Both peers now have each other’s peer IDs and confirm that they’re connected to the correct nodes within the torrent swarm.

**Closing the Connection if IDs Don’t Match**

If Peer A expects a specific peer ID but receives one that doesn’t match (for example, if Peer B is impersonating a different peer), Peer A will sever the connection. This ensures that connections are only maintained between known and trusted peers, as verified by their unique peer IDs.

### Question 3a

Ok, so what happens after handshake. Following this:

“That's it for handshaking, next comes an alternating stream of length prefixes and messages. Messages of length zero are keepalives, and ignored. Keepalives are generally sent once every two minutes, but note that timeouts can be done much more quickly when data is expected.“

Peers start sending ”peer messages”. Can you give some example flows? And also, what are those “length prefixes” mentioned above?
#### Answer

After the handshake, BitTorrent peers begin exchanging a series of peer messages to coordinate downloading and uploading. These messages contain a length prefix to indicate the message’s size, allowing peers to read messages correctly from the data stream. Here’s how this works and what kind of messages typically flow between peers:

**Understanding Length Prefixes**

Each message in the peer protocol includes a length prefix at the beginning, which specifies the number of bytes that follow:

- The length prefix is typically a 4-byte integer.

- For example, if a message’s length prefix is 0009, it indicates that the next 9 bytes make up the message.

A special case is the keep-alive message: it has a length prefix of zero and no body, which simply tells the peer that the connection is still active without performing any specific action.

**Common Peer Message Types and Example Flows**

Here are some typical message types and how they might flow between peers after the handshake:

  

1. **Interested / Not Interested Messages**

	- **Purpose**: After connecting, a peer will send an interested message if it wants pieces from the other peer, or not interested if it doesn’t need anything at the moment.

	- **Flow Example**:

		- Peer A sends interested to Peer B if it’s looking for pieces.

		- Peer B responds with unchoke if it’s willing to upload to Peer A (choking controls upload permissions).

2. **Have Message**

- **Purpose**: A peer sends have messages to inform the other peer that it has successfully downloaded a specific piece.

- **Flow Example**:

	 - Peer A completes a piece and sends a have message with the piece index to Peer B.

	- Peer B can then decide to mark Peer A as a potential source for that piece.

3. **Request Message**

	- **Purpose**: To request specific pieces of data, the peer sends a request message specifying the index of the piece it wants.

	- **Flow Example:**

		- Peer A sends a request message to Peer B for a piece.

		- Peer B, if unchoked, responds with a piece message containing the data for the requested piece.

4. **Piece Message**

	- **Purpose**: The piece message contains the actual data being transferred, typically a specific piece or chunk of the file.

	- **Flow Example**:

		- Peer B sends a piece message to Peer A with the data after receiving a request from Peer A.

		- Peer A receives the piece, verifies it (using a hash check), and may send another request for the next piece.

5. **Choke / Unchoke Messages**

	- **Purpose**: Peers use choke and unchoke messages to control the upload permissions to other peers, helping manage bandwidth and prioritize uploads.

	- **Flow Example**:

		• If Peer B’s upload slots are full, it sends a choke message to Peer A, temporarily halting uploads.

		- When an upload slot frees up, Peer B sends an unchoke message to Peer A, allowing it to resume downloading.

6. **Keep-Alive Message**

	- **Purpose**: To keep the connection open, a peer will send a keep-alive message (length prefix of zero) if no other messages have been sent within a two-minute interval.

	- **Flow Example**:

		- Peer A sends a keep-alive message to Peer B after a period of inactivity to signal that the connection is still active.

**Example Peer Message Flow**

Imagine Peer A and Peer B connecting to share a file in the torrent:

1. **Handshake**: Both peers complete the handshake and verify they’re part of the same torrent.

2. **Interest and Permission**:

	- Peer A sends an interested message to Peer B, indicating it wants to download pieces.

	- Peer B responds with an unchoke message, granting permission to download.

3. **Data Requests**:

	- Peer A sends a request message asking for a piece.

	- Peer B responds with a piece message containing the data.

4. **Notification of Completion**:

	- After receiving and verifying a piece, Peer A sends a have message to inform Peer B that it has that piece.

	- Periodic Keep-Alive:

	- If there is a lull in communication, Peer A and Peer B may exchange keep-alive messages to ensure the connection stays active.

This cycle of requesting, receiving, and sharing updates continues until Peer A has downloaded all the necessary pieces of the file from Peer B and potentially other peers in the swarm.

### Question 4

How do the peers know which peer has which pieces? Initially, it may happen that there is only one “seeder”, right?
#### Answer

In the BitTorrent protocol, peers keep track of which pieces each peer in the swarm has through a system of bitfields and have messages:

1. **Bitfield Message (Initial Knowledge)**

	- When a peer first connects to another peer, it sends a bitfield message if it has any pieces of the file. This message is a binary string where each bit represents a piece, indicating whether the peer has that specific piece (1 for presence, 0 for absence).

	- **Example**: If the torrent file has 8 pieces, a bitfield of 10100011 would mean that the peer has pieces 0, 2, 6, and 7.

	- If the peer doesn’t have any pieces (for example, if it just started downloading), it may skip sending a bitfield or send an empty bitfield, depending on the client.

2. **Have Messages (Dynamic Updates)**

	- As peers download pieces, they send have messages to inform other connected peers of newly acquired pieces.

	- **Example**: If Peer A downloads piece 3, it sends a have message with the piece index (3) to each peer it’s connected to. This allows other peers to update their view of which pieces Peer A has.

3. **Initial Seeding and Peer Awareness**

	- In a new swarm, there might only be one seeder (a peer that has the entire file). The seeder’s bitfield will show all bits set to 1, indicating it has every piece.

	- Peers that connect to this seeder will quickly learn that it has all pieces and can request them in the order specified by the piece selection strategy (e.g., rarest first, random).

4. **Piece Selection Strategy**

	 - Once a peer knows which pieces each peer has, it uses a piece selection strategy (like rarest-first) to decide which pieces to request. This helps distribute pieces more evenly across the swarm and prevents all peers from downloading the same piece from the seeder.

Through the initial bitfield exchange and subsequent have messages, peers dynamically update their knowledge of which pieces each peer in the swarm has, allowing them to efficiently download and share pieces among themselves.

### Question 5

So, how does it all start? Let’s say Peer A found a tracker-less torrent file, and wants to download that file. As you see I mentioning tracker-less setup, which means that tracker file will not have `announces` field, but rather `nodes` filed. Thus, in this scenario we are using DHTs. I would like to learn a complete exchange, with all the messages for a file with 3 pieces, and a SWARM with 3 peers. Can you help?
#### Answer

In a tracker-less BitTorrent setup, peers use the Distributed Hash Table (DHT) to find each other without a central tracker. Here’s a step-by-step example of how Peer A could begin downloading a file with 3 pieces from a swarm of 3 peers using DHT.

##### Initial Setup

- **File Info**: The torrent file describes a file split into 3 pieces.

- **Swarm**: Three peers (A, B, and C) are in the swarm.

- **DHT**: The torrent has no announce field but contains a nodes field, listing known DHT nodes.

##### Steps in the Exchange

**Step 1: Peer A Joins the DHT Network**

1. **Bootstrapping**:

	- Peer A begins by connecting to DHT nodes listed in the nodes field. These bootstrap nodes help Peer A join the DHT network by introducing it to other nodes.

2. **Finding Peers for the Torrent**:

	- Peer A generates a unique DHT key for the torrent based on the info_hash (SHA1 hash of the torrent’s “info” section).

	- Peer A uses a find_node DHT query, asking nearby DHT nodes for peers with the same info_hash. Through several find_node and get_peers DHT requests, Peer A eventually locates Peers B and C, who are also sharing this torrent.

**Step 2: Peer A Connects to Peers B and C**  

1. **Peer-to-Peer Handshake**:

	- Peer A initiates a connection to Peer B by sending the BitTorrent handshake, including the protocol identifier, info_hash, and Peer A’s ID.

	- Peer B responds with its own handshake, verifying that both are part of the same torrent.

	- Peer A repeats this with Peer C.

**Step 3: Exchanging Bitfields**

1. **Bitfield Exchange**:

	- Peer B has all three pieces (is a seeder) and sends a bitfield 111 (indicating it has all pieces).

	- Peer C has only the first two pieces and sends a bitfield 110.

	- Peer A has no pieces yet and sends a bitfield 000.

2. **Updating Knowledge**:

	- After the bitfield exchange, Peer A knows which pieces each peer has:

	- Peer B can provide any piece (0, 1, or 2).

	- Peer C can only provide pieces 0 and 1.

**Step 4: Piece Requests and Data Transfer**

1. **Piece Selection Strategy**:

	- Peer A decides to request pieces from the rarest to the most common. It finds that piece 2 is rare (only available from Peer B).

2. **Requesting Pieces**:

	- **Requesting Piece 2**:

		- Peer A sends a request message to Peer B for piece 2.

		- Peer B responds with a piece message containing piece 2.

	- **Requesting Piece 0**:

		- Peer A then sends a request message to Peer C for piece 0.

		- Peer C responds with a piece message containing piece 0.

	- **Requesting Piece 1**:

		Finally, Peer A requests piece 1 from Peer C, who sends it over in a piece message.

3. **Announcing Newly Downloaded Pieces:**

	- Each time Peer A completes a piece, it sends a have message to both Peer B and Peer C to inform them of its new pieces.

	- After completing all three pieces, Peer A has the full file and can start serving as a seeder.

**Step 5: Ongoing Keep-Alives and Maintenance**

1. **Keep-Alive Messages**:

	- If there is a period of inactivity, the peers send keep-alive messages every two minutes to maintain the connection.

2. **Maintaining DHT Membership**:

	- Peer A periodically sends DHT ping requests to refresh its connection with other DHT nodes, ensuring it remains discoverable in the swarm.

This complete exchange shows how Peer A starts with a DHT lookup to find peers, establishes connections, exchanges bitfields to learn which pieces each peer has, and requests pieces in a way that optimizes download efficiency. Through this process, Peer A downloads the full file and becomes part of the swarm, ready to upload pieces to other peers.

### Question 5a

In **Step 3 - Exchanging Bitfields** - peers are sending their bitfields to other peers. But how does it happen? How peers know that they should send their bitfields to other peers? This is not a broadcast, right? What is the trigger?
#### Answer

In the BitTorrent protocol, the exchange of bitfields between peers is not automatic or broadcasted. Instead, bitfields are triggered by the completion of the handshake between two peers. Here’s how it happens:

1. **Handshake Completion**:

	- When two peers complete the handshake, they confirm they’re part of the same torrent by verifying the info_hash.

	- Once the handshake is successful, both peers now recognize each other as legitimate participants in the torrent.

2. **Bitfield Message Trigger**:

	- Immediately following the handshake, each peer sends a bitfield message to the other. This message provides a snapshot of which pieces the peer already possesses.

	- If a peer has no pieces (such as a newly joined downloader), it might send an empty or zeroed bitfield, depending on the client implementation.

3. **Purpose of Bitfield Exchange**:

	- This initial bitfield exchange is essential because it allows both peers to understand what pieces each other has, which guides future request and have messages.

	- After the initial exchange, any newly completed pieces are communicated with have messages, not a repeated bitfield.

So, the handshake acts as the trigger for the bitfield message. This is a direct, one-to-one exchange and not a broadcast. Each peer sends its bitfield only to those peers it connects with individually, ensuring that the swarm doesn’t experience unnecessary network load from broadcasting.
