Defined in `codex/blockexchange/protobuf/message.proto`:

```
// Protocol of data exchange between Codex nodes.
// Extended version of https://github.com/ipfs/specs/blob/main/BITSWAP.md

syntax = "proto3";

package blockexc.message.pb;

message Message {

  message Wantlist {
    enum WantType {
      wantBlock = 0;
      wantHave = 1;
    }

    message Entry {
      bytes block = 1;       // the block cid
      int32 priority = 2;    // the priority (normalized). default to 1
      bool cancel = 3;       // whether this revokes an entry
      WantType wantType = 4; // Note: defaults to enum 0, ie Block
      bool sendDontHave = 5; // Note: defaults to false
    }

    repeated Entry entries = 1;  // a list of wantlist entries
    bool full = 2;               // whether this is the full wantlist. default to false
  }

  message Block {
    bytes prefix = 1; // CID prefix (cid version, multicodec and multihash prefix (type + length)
    bytes data = 2;
  }

  enum BlockPresenceType {
    presenceHave = 0;
    presenceDontHave = 1;
  }

  message BlockPresence {
    bytes cid = 1;
    BlockPresenceType type = 2;
    bytes price = 3;    // Amount of assets to pay for the block (UInt256)
  }

  message AccountMessage {
    bytes address = 1; // Ethereum address to which payments should be made
  }

  message StateChannelUpdate {
    bytes update = 1; // Signed Nitro state, serialized as JSON
  }

  Wantlist wantlist = 1;
  repeated Block payload = 3;
  repeated BlockPresence blockPresences = 4;
  int32 pendingBytes = 5;
  AccountMessage account = 6;
  StateChannelUpdate payment = 7;
}
```