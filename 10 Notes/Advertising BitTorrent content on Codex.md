---
tags:
  - bittorrent
related-to:
  - "[[How BitTorrent-Codex integration may look like?]]"
  - "[[Learn BitTorrent]]"
---
#bittorrent 

| related-to | [[How BitTorrent-Codex integration may look like?]], [[Learn BitTorrent]] |
| ---------- | ------------------------------------------------------------------------- |

This content builds upon [[BitTorrent metadata files]]

Let's start with BitTorrent protocol version 1 and let's start with relatively simple case: a single `40 kiB` single file:

![[data40k-v1.svg]]
We have a single, incomplete piece of size `64 kiB`, which will be transferred as 3 blocks (3rd block will be shorter, namely `8 kiB`). Peers in BitTorrent v1, operate at the piece level when requesting the content. The `request` message contains an `index`, `begin`, and `length`. Except when it is truncated at the end of the file, the length is generally power of 2 and all current implementations use $2^{14}$ = `16 kiB` and **close connections which request an amount greater than that**.

The corresponding torrent file will be the following:

```json
{
  "announce": "http://example.com/announce",
  "info": {
    "length": 40960,
    "name": "data40k.bin",
    "piece length": 65536,
    "pieces": [
      "1cc46da027e7ff6f1970a2e58880dbc6a08992a0"
    ]
  }
}
```

or in the b-encoded form:

```
d8:announce27:http://example.com/announce4:infod6:lengthi40960e4:name11:data40k.bin12:piece lengthi65536e6:pieces20:bbbbbbbbbbbbbbbbbbbb
```

where `bbbbbbbbbbbbbbbbbbbb` stands for `20` successive bytes being sha1 hash of the only piece. When computing the hash, no padding was used at the end of the file:

```bash
openssl sha1 experiment-3/data40k.bin 
SHA1(experiment-3/data40k.bin)= 1cc46da027e7ff6f1970a2e58880dbc6a08992a0
```

The info, sha1 hash of the `info` dictionary is: `1902d602db8c350f4f6d809ed01eff32f030da95`.

Now, let's assume our input is a magnet link corresponding to the above torrent file:

```
magnet:?xt=urn:btih:1902d602db8c350f4f6d809ed01eff32f030da95&dn=data40k.bin
```

The magnet link itself does not tell us details of the corresponding torrent file: what we have is only the hash of the bencoded `info` directory. How does the peer interested in downloading the content designated by the given magnet link finds the remaining details of the `info` dictionary? This is defined in [[BEP9 - Extension for Peers to Send Metadata Files]].

> [!note]
> BEP9 is also where the magnet links were introduced.

[[BEP9 - Extension for Peers to Send Metadata Files|BEP9]] uses [[BEP10 Extension Protocol]] as a “carrier”. The metadata is handled in blocks of 16KiB (16384 Bytes). The metadata blocks are indexed starting at 0. All blocks are 16KiB except the last block which may be smaller. The details of [[BEP9 - Extension for Peers to Send Metadata Files|BEP9]] and [[BEP10 Extension Protocol|BEP10]] are less important here, only when necessary for a better understanding of what has to be considered as an addition to the Codex protocol that we will mention some details where deemed necessary.

In version 1 of the BitTorrent protocol, we can only check the integrity of the data being downloaded at the piece level. In our example, there is only one piece, and the downloading peer will need three `request` messages in order to fetch the necessary data:

1. `{ index=0, being=0, length=16384 }`
2. `{ index=0, being=16384, length=16384 }`
3. `{ index=0, being=32768, length=8192 }`

Thus, two blocks of `16 kiB` and one shorter of `8 kiB`. Only after we fetch the whole piece, we will be able to compute and verify its hash.

When having multiple files, the situation does not change much at least form the perspective of Codex integration. Let’s looks at the example, which additionally will help us to clarify some interesting facts about piece alignment extensions and hybrid torrents.

Here we have an example with three files:

- `258 kiB` (named `data258k.bin`)
- `40 kiB` (named `data40k.bin`)
- `72 kiB` (named `data72k.bin`)

Following the original [[BEP3 - The BitTorrent Protocol Specification|BEP3]], the files would be ordered in sequence one after and projected on the pieces spaces without any alignment. This would make, however, interoperability with the version 2 of the protocol harder. Normally, the same content will be represented by different `info` hashes for different versions of the protocol and thus the peers using `info` hash for version 1 of the protocol will form a separate swarm from those peers using `info` hash for version 2. In some circumstances, it may be beneficial for the peer to join both swarms in order to complete the download faster as some content may be available in the swarm for version 1 `info` hash while not being available in the swarm for version 2. To keep higher level of the protocols agnostic to which version of the protocol is used to acquire relevant pieces, it is desired that the content in both swarms has the same file names, order, and piece alignment so that the same `request` message can be used for both swarm resulting in the same bytes being downloaded. This is where [[BEP47 - Padding files and extended file attributes]] comes handy. [[BEP47 - Padding files and extended file attributes|BEP47]] allows the files in version 1 of the protocol to also be aligned on the piece boundary by introducing padding. This is what we see in the picture below.

![[multi-data-v1.svg]]

File `data258k.bin` is “sticking out” of the piece boundary by exactly `2 kiB`. In order to make sure that the next file is aligned to the following piece, a *padding* file of `62 kiB` is added. Similarly, file `40 kiB` file `data40k.bin` will be padded with `24576 Bytes` (`24 kiB`) and the last file will be padded with `56 kiB`.

> [!note]
> The last file is also padded to the piece boundary (*although I do not see any good reason for that*).


Below is the corresponding torrent file:

```json
{
  "announce": "http://example.com/announce",
  "info": {
    "files": [
      {
        "length": 264192,
        "path": [
          "data258k.bin"
        ]
      },
      {
        "attr": "p",
        "length": 63488,
        "path": [
          ".pad",
          "63488"
        ]
      },
      {
        "length": 40960,
        "path": [
          "data40k.bin"
        ]
      },
      {
        "attr": "p",
        "length": 24576,
        "path": [
          ".pad",
          "24576"
        ]
      },
      {
        "length": 73728,
        "path": [
          "data72k.bin"
        ]
      },
      {
        "attr": "p",
        "length": 57344,
        "path": [
          ".pad",
          "57344"
        ]
      }
    ],
    "name": "experiment-6",
    "piece length": 65536,
    "pieces": [
      "6e2f08ac8c18ac0e1a639474781882019bac40fd",
      "52422fcc223e90b84a3b60acbdee6104e2833234",
      "3c1a3b20f63ada631660c9416ec545ccc0995797",
      "8e59fe5e59b32fc58f6d0d44355f01cbc0e5a37a",
      "9772758f13faf635e02fecb47df4c25aed237981",
      "145355c75352ccc9a8500fcc6e8d21a65e4380b2",
      "19cc13b4954cc65b8de2578a6004d383947fa0dc",
      "92e3dbd456711254df6feddcb6f0b8b1aa7f53bf"
    ]
  }
}
```

Padding files have their own names, and are included in the `files` entry of the `info` dictionary. Clients aware of [[BEP47 - Padding files and extended file attributes|BEP47]] extension don't need to write the padding files to disk and should also avoid requesting byte-ranges covering their contents, e.g. via `request` messages. But for backwards-compatibility they must service such requests. For the calculation of piece hashes the content of padding file is all zeros.

Having this content alignment in place, a peer having a hybrid magnet link, i.e. a link with both version 1 and version 2 hashes, may try to join both version 1 and version 2 swarms for a faster content download.

Hybrid torrents seem a bit under-documented. I am trying to gather some thoughts about them in [[Hybrid Torrents]]. 

To summarize, for trackerless version 1 torrents, having only `info` hash as the input, in order to be able to successfully download and verify the integrity of the content, the peers need to support [[BEP9 - Extension for Peers to Send Metadata Files]].

### How the peer knows it gets the right metadata

The extension only sends the content of the `info` dictionary. For a good reason. Anything outside of the `info` dictionary cannot be validated, as the corresponding `info` hash covers only the `info` entry of the torrent file. So, for example, for our single file torrent:

```json
{
  "announce": "http://example.com/announce",
  "info": {
    "length": 40960,
    "name": "data40k.bin",
    "piece length": 65536,
    "pieces": [
      "1cc46da027e7ff6f1970a2e58880dbc6a08992a0"
    ]
  }
}
```

The `info` dictionary is:

```json
{
  "length": 40960,
  "name": "data40k.bin",
  "piece length": 65536,
  "pieces": [
    "1cc46da027e7ff6f1970a2e58880dbc6a08992a0"
  ]
}
```

or bencoded (for convenience I am using Python notation, where `b"some-data"` is a byte object or a sequence of bytes):

```Python
b"d6:lengthi40960e4:name11:data40k.bin12:piece lengthi65536e6:pieces20:\x1c\xc4m\xa0'\xe7\xffo\x19p\xa2\xe5\x88\x80\xdb\xc6\xa0\x89\x92\xa0e"
```

This is a short `info` dictionary, thus the client would only request a single piece (incomplete `16 kiB` block - `90` bytes in this case). A peer, after indicating it support the [[BEP10 Extension Protocol|BEP10]] extension protocol (by setting bit `4` of `reserved_byte[5]` to `1`) during the general handshake, it will send the following BEP9 extension handshake message:

```Python
{b'm': {b'ut_metadata', 3}, b'metadata_size': 90}
```

It will be bencoded and the complete extension handshake message (including length prefix (`uint32_t`, big endian), message id (`uint8_t`), extended message id (`uint8_t`) and payload) will be represented by the following byte object:

```Python
from bep_0052_torrent_creator import encode
handshakePayload = encode({b'm': {b'ut_metadata': 3}, b'metadata_size': 90})
# b'd1:md11:ut_metadatai3ee13:metadata_sizei90ee'
assert len(handshakePayload) == 44
extensionHandshake = b'\x00\x00\x00\x32\x16\x00d1:md11:ut_metadatai3ee13:metadata_sizei90ee'
assert len(extensionHandshake) == 50 # 0x32 in extensionHandshake[3]
```

Then we will have metadata `request`:

```Python
requestPayload = {b'msg_type': 0, b'piece': 0}
requestBenc = encode(requestPayload)
assert requestBenc == b'd8:msg_typei0e5:piecei0ee'
assert len(requestBenc) == 25
metadataRequest = b'\x00\x00\x00\x1F\x16\x03d8:msg_typei0e5:piecei0ee'
assert len(metadataRequest) == 31 # 0x1F in extensionHandshake[3]
```

Finally the response will be:

```Python
responsePayload = {b'msg_type': 1, b'piece': 0, b'total_size': 90}
responseBenc = encode(responsePayload)
assert responseBenc == b'd8:msg_typei1e5:piecei0e10:total_sizei90ee'
assert len(responseBenc) == 42
metadataResponse = metadataResponse = b"\x00\x00\x00\x8A\x16\x03d8:msg_typei1e5:piecei0e10:total_sizei90eed6:lengthi40960e4:name11:data40k.bin12:piece lengthi65536e6:pieces20:\x1c\xc4m\xa0'\xe7\xffo\x19p\xa2\xe5\x88\x80\xdb\xc6\xa0\x89\x92\xa0e"
assert len(metadataResponse) == 138 # 0x8A in extensionHandshake[3]
```

After receiving metadata, the peer can compute hash over the bencoded response payload and compare it with the `info` hash from the magnet link:

```Python
assert sha1(bencoded).hexdigest() == '1902d602db8c350f4f6d809ed01eff32f030da95'
```

Having metadata, we can now verify each retrieved piece.

### BitTorrent Version 2

In BitTorrent version 2, our input will be version 2 magnet link contained tagged `tagged-info-hash`, which is the [multihash](https://github.com/multiformats/multihash) formatted, hex encoded full `info` hash for torrents in the new metadata format. For example, for our three files contents from the previous example the `sha256` of the bencoded `info` dictionary will be:

```
970603312f21c543826c3bad8e289de8d68678298701b8579ce448895ce6dcd6
```

and as multihash it will get `1220` prefix:

```
1220970603312f21c543826c3bad8e289de8d68678298701b8579ce448895ce6dcd6
```

The v2 torrent file can be the following:

```json
{
  "announce": "http://example.com/announce",
  "info": {
    "file tree": {
      "data258k.bin": {
        "": {
          "length": 264192,
          "pieces root": "d62f5c8510048ba73a1950a6d27750c6262f88be2016f8f9d4b2b9ffe51477e1"
        }
      },
      "data40k.bin": {
        "": {
          "length": 40960,
          "pieces root": "703ef11e93d8ef1f052abace41e3e40fa99842697b405e4c744abad11017a11a"
        }
      },
      "data72k.bin": {
        "": {
          "length": 73728,
          "pieces root": "857663dce7d614983b289c0939513130c0bfd451447268655413b5a6b72a593c"
        }
      }
    },
    "meta version": 2,
    "name": "experiment-6",
    "piece length": 65536
  },
  "piece layers": {
    "857663dce7d614983b289c0939513130c0bfd451447268655413b5a6b72a593c": "bfff3a7f25089296b73b2cb80c48cf799c858e043e52f4a6326e938e900bb0e9b4b42d885b50f3d447ffd4a17905d5d03a0495270a71177d41155e86a03d9bdb",
    "d62f5c8510048ba73a1950a6d27750c6262f88be2016f8f9d4b2b9ffe51477e1": "52a914929e9bde10dfc1f80141d15c82c8c092693cc3eab03139e24d69e9e305ffc938c3889eb4a7168894673d525f983b3749b1ae19bdd645fdffe5dbcf043b8186be160cf2774d92786db55a0d7e3a960a9f02f52bdd801573bc81b7c2152b5f22fe59816961da2f451afa606ece30ce9ec7036423bfd14adbb6fb2a17559a9770f868e8c6a8cc042eb694bb9dd2ecee97160f6b5f63715217180e3943a840"
  }
}
```

Now, important thing to notice, is that the `pieces layers` attribute is not part of the `info` dictionary. Using the [[BEP9 - Extension for Peers to Send Metadata Files]] alone we will still be able to verify the integrity of each file after downloading its all pieces: in the `file tree` entry of the `info` dictionary, we have the `pieces_root` property which is the root of the Merkle Tree corresponding to that file. Unfortunately, we cannot verify the pieces as the `info` dictionary does not provide us with the hashes of the corresponding pieces. Even having access to the `piece layer` attribute from the torrent file, we would not be able to early verify integrity of the blocks we download.

BitTorrent version 2, solves that issue by introducing three new peer messages:

- `hash request`
- `hashes`
- `hash reject`

Later I will provide some examples of those messages, but here it is enough to say that with great flexibility, they allow us to retrieve the Merkle inclusion proofs for each file. Together with [[BEP9 - Extension for Peers to Send Metadata Files|BEP9]] the peer can efficiently verify the contents being downloaded down to the `16 kiB` level.

### Extending Codex

From the above analysis, we see that for Codex protocol to be able to directly host BitTorrent content, we need the following extensions:

1. Peers need to advertise their corresponding SPRs under both `info` hash versions 1 and 2, so that they can be discovered.
2. To handle BitTorrent version 1 traffic, it is sufficient to support [[BEP9 - Extension for Peers to Send Metadata Files]].
3. We already use Merkle inclusion proofs, so we have great deal of flexibility of how we want to support torrents version 2. The `info` dictionary already provides us with the original file roots via `pieces root`, so we basically have to make sure we can provide the relevant intermediate layers aligned to the piece length (the `pieces root` will be built on top of that). Moreover, having inclusion proofs in place we should be able to improve version 1 torrents as well. With original pieces hashes coming from the `info` dictionary, we can secure authenticity of the content, and with Codex inclusion proofs we can enhance torrents version 1 with early validation.

More detailed discussion will follow after learning more low level details of the Codex client.