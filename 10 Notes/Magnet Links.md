Magnet links has been introduced in [[BEP9 - Extension for Peers to Send Metadata Files]]. Here is a short summary with examples.

Magnet link format:

```
v1: magnet:?xt=urn:btih:<info-hash>&dn=<name>&tr=<tracker-url>&x.pe=<peer-address>
v2: magnet:?xt=urn:btmh:<tagged-info-hash>&dn=<name>&tr=<tracker-url>&x.pe=<peer-address>
```

BEP9 provides detailed description of all the fields. Here it is important to emphasize that only `xt` is mandatory. In particular for our purpose, in trackerless torrents, we may see in practice the magnet links having only the `xt` attribute. Other attributes, especially `x.pe` (peer address), may be used as well, but our discovery should not relay on any other attributes than `xt`.

Tow examples of such a simplest magnet-links:

**Version 1**

```
magnet:?xt=urn:btih:1902d602db8c350f4f6d809ed01eff32f030da95
```

**Version 2**

```
magnet:?xt=urn:btmh:122003ee5d27d20f46c2395fb00e407a85f87c885e2e072d01f4e4bc516474203fe1
```

**Hybrid**

```
magnet:?xt=urn:btmh:122003ee5d27d20f46c2395fb00e407a85f87c885e2e072d01f4e4bc516474203fe1&xt=urn:btih:1902d602db8c350f4f6d809ed01eff32f030da95
```

`122003ee5d27d20f46c2395fb00e407a85f87c885e2e072d01f4e4bc516474203fe1` is a SHA-256 multihash:

```
sha2-256/03ee5d27d20f46c2395fb00e407a85f87c885e2e072d01f4e4bc516474203fe1
```
