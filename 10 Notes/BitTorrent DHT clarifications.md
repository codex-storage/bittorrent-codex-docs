---
tags:
  - bittorrent
  - dht
related-to:
  - "[[Learn BitTorrent]]"
---
#bittorrent 

| related-to | [[Learn BitTorrent]] |
| ---------- | -------------------- |

### Question 1

The BitTorrent Enhancement proposal for DHTs (BEP-5) includes the following paragraph:

“The return value for a query for peers includes an opaque value known as the "token." For a node to announce that its controlling peer is downloading a torrent, it must present the token received from the same queried node in a recent query for peers. When a node attempts to "announce" a torrent, the queried node checks the token against the querying node's IP address. This is to prevent malicious hosts from signing up other hosts for torrents. Since the token is merely returned by the querying node to the same node it received the token from, the implementation is not defined. Tokens must be accepted for a reasonable amount of time after they have been distributed. The BitTorrent implementation uses the SHA1 hash of the IP address concatenated onto a secret that changes every five minutes and tokens up to ten minutes old are accepted.”

Can you explain it better or make it clearer? I really do not see how those Tokens are working…

#### Answer

The use of tokens in the BitTorrent DHT (Distributed Hash Table) serves as a security measure to verify that a node genuinely intends to join a torrent swarm. Here’s a clearer explanation:

##### How Tokens Work

1. **Token Issuance:**
	- When a node (let’s call it Node A) queries another node (Node B) in the DHT for peers associated with a torrent, Node B responds with a list of peers. Along with the peer list, Node B includes an opaque “token” in its response.

	- This token is generated uniquely by Node B and is associated with the IP address of Node A at the time of the request.

2. **Using the Token:**

	- If Node A later wants to announce itself as a peer for the torrent (effectively telling Node B, “I’m now part of this swarm”), it must send this previously received token back to Node B.

	- Node B checks if the token matches what it previously issued for Node A’s IP address. This step ensures that the token is only valid for the original requester, which is Node A in this case.

3. **Security Check:**

	- This token mechanism prevents “malicious” nodes from misrepresenting other IP addresses as part of the torrent swarm. For example, without the token mechanism, an attacker could attempt to announce other IP addresses to the swarm, making it appear as though other nodes are participating when they’re not.

	- Since tokens are specific to the requesting IP and are time-sensitive, they are difficult to forge or reuse from a different IP address.

4. **Token Expiry:**

	- Tokens have a limited validity, usually around five to ten minutes. BitTorrent’s implementation, as mentioned, uses a SHA1 hash of the requester’s IP address combined with a secret that changes every five minutes. This time-bound validity ensures that if a token is misused, it won’t be valid for long, adding another layer of security.

In summary, the token is a temporary key used to confirm that a peer intending to announce itself actually performed a recent lookup from its current IP. This prevents unauthorized nodes from announcing other IPs, helping to keep the DHT swarm authentic and secure.

### Question 2

OK, later, I read:

“Upon inserting the first node into its routing table and when starting up thereafter, the node should attempt to find the closest nodes in the DHT to itself. It does this by issuing find_node messages to closer and closer nodes until it cannot find any closer. The routing table should be saved between invocations of the client software.”

When calling `find_node` to “closer and closer” nodes, where are we starting from?
#### Answer

In the DHT (Distributed Hash Table) protocol, the process of finding “closer and closer” nodes during a find_node search is centered on locating nodes that have identifiers (node IDs) numerically closer to the node initiating the search. Here’s how it works:

1. **Starting Point - Known Nodes in the Routing Table:**

	- When a node (let’s call it Node A) joins the DHT, it typically starts with an initial set of known nodes. These nodes may come from a pre-existing list, prior DHT connections, or even cached nodes from a previous session if the client saves its routing table.

	- Node A then starts by querying these known nodes to find nodes that are progressively closer (in terms of XOR distance) to its own ID.

2. **XOR Distance for Proximity:**

	- In the DHT, node IDs are compared using an XOR metric, where the “distance” between two nodes is the XOR result of their IDs. The closer the XOR result is to zero, the closer the node IDs are.

	- Node A issues find_node requests to its known nodes, asking for nodes that are closer to its own ID. Each find_node response provides a list of nodes that are closer to Node A’s ID than previous nodes.

3. **Iterative Process - Narrowing Down:**

	- Node A repeats this process iteratively: with each step, it queries nodes that are progressively closer to its ID until it can no longer find any nodes that are closer.

	- This process effectively narrows down the search space in the DHT and helps Node A locate its own neighborhood in the distributed network.

4. **Building the Routing Table:**

	- As Node A locates closer nodes, it adds them to its routing table, arranging them in buckets based on their XOR distance to its own ID. This routing table structure is then used for efficiently finding peers and performing lookups in the DHT.

In summary, the search starts with known nodes and iteratively progresses toward nodes with IDs numerically closer to the querying node’s own ID, refining the list of nodes to build a complete, locally optimized view of the network. This method ensures that the node can quickly locate nearby nodes in the DHT structure.

### Question 3

But here we are talking about starting a fresh node, and adding the first node (itself), how does it know which ID to use as a starting point of this ”closer and closer” search to start subsequent nodes? Is it random from the whole ID space?
#### Answer

When a fresh DHT node (with no pre-existing routing table) starts, it generally performs the following steps to join the network:

1. **Choosing an Initial ID:**

	- Each DHT node generates its own unique ID when it first joins the network. This ID is typically selected randomly from the entire ID space. The ID does not directly influence which nodes it will connect to at first but serves as a unique identifier for locating its “closest” nodes in the network.

2. **Starting with Bootstrap Nodes:**

	- Since the new node has no information about the existing network, it must begin by contacting bootstrap nodes. These are predefined nodes whose IP addresses are embedded in the client software or are provided by a trusted external source.

	- The bootstrap nodes help the fresh node “see” the network by responding with a list of other nodes that are relatively close to the new node in the ID space.

3. **Executing the find_node Search:**

	- Using these bootstrap nodes as a starting point, the new node begins issuing find_node requests. The goal is to locate nodes that are progressively closer to its own ID by comparing XOR distances.

	- Each response provides a list of nodes with IDs closer to the new node’s own ID, allowing it to iteratively narrow down and populate its routing table.

4. **Building a Routing Table:**

	 - As the new node receives responses, it builds a routing table by adding nodes in “buckets” based on their proximity (XOR distance) to its own ID. This process enables the new node to gradually become part of the DHT by efficiently locating nearby peers in the network.

By using bootstrap nodes and performing iterative find_node queries, the new node can establish its place in the DHT without needing prior knowledge of any specific starting ID. The randomly generated ID ensures that each node’s location within the DHT is unique and well-distributed across the ID space, which supports the overall balance and resilience of the network.

### Question 4

It says in the context of KRPC protocol that:

“The transaction ID should be encoded as a short string of binary numbers, typically 2 characters are enough as they cover 2^16 outstanding queries.”

What is a “string of binary numbers”? Can you give an example?

> This is an excellent example of how **not** to write specs. The term “string of binary numbers” is nonsense in best case, or an oxymoron in the worst. Binary is already a representation of a number. A number can `33` - here using *base-10* or *decimal* encoding. The same number written in *base-16*, or *hexadecimal* encoding is `11` and in *base-2* or binary encoding it is `00010001`. But what is a binary number? It is a shortcut to say *a number in binary encoding*. So what would be a string of binary numbers. Would “1110011, 11011111, 000111” qualify? 
> This is one of many examples, where the language in the spec is unclear and ambiguous.

> It is amusing to see how ChatGPT tries hard to make sense out of it. Admittedly doing quite a good job here…
#### Answer

In the context of the KRPC (Kademlia RPC) protocol used by BitTorrent DHT, a “string of binary numbers” refers to a sequence of bytes (not just 0s and 1s) that represent a transaction ID. This ID is used to match requests with responses and is typically short, only 2 bytes long. Each byte can represent values from 0 to 255, so with two bytes (16 bits), you can represent 65,536 unique transaction IDs, which is sufficient for managing active requests in the network.

Example

A “string of binary numbers” could look like this in hexadecimal (which is often used to represent binary data more compactly):

	•	0x01 0xAB or in binary: 00000001 10101011

Here, 0x01AB represents a 2-byte transaction ID. Since the KRPC protocol expects a transaction ID to be short and efficient, these IDs are kept at 2 bytes, allowing enough room to track multiple requests at once without using excessive bandwidth.

In many programming languages, these IDs would simply be handled as byte arrays or short strings and treated as opaque binary data, meaning there’s no need for it to be human-readable.

#### Followup question

So, we need 4 characters, right? What do they say that 2 characters are sufficient?

#### Answer

When the KRPC protocol documentation refers to using “2 characters” to encode the transaction ID, it typically means 2 **bytes**, not literal ASCII characters. Since each byte can represent 256 values (0-255), two bytes provide a range of 65,536 possible transaction IDs, which is often enough to manage outstanding queries.

Here’s how this works in binary and hexadecimal formats:  

- Binary: `00000001 10101011` (16 bits total, where each “character” is actually 8 bits or 1 byte).

- Hexadecimal: `0x01AB` (which is still 2 bytes or 16 bits, representing the same value as above).

In hexadecimal, this range covers values from 0x0000 to 0xFFFF, making it possible to manage a high number of concurrent requests without needing more than 2 bytes.

