---
tags:
  - codex/block-exchange
  - codex/libp2p
  - libp2p
related:
  - "[[Codex Blocks]]"
  - "[[Uploading and downloading content in Codex]]"
  - "[[Protocol of data exchange between Codex nodes]]"
---
| related | [[Codex Blocks]], [[Uploading and downloading content in Codex]], [[Protocol of data exchange between Codex nodes]] |
| ------- | ------------------------------------------------------------------------------------------------------------------- |

Codex block exchange protocol is built on top of [[libp2p]].

To understand how Codex Block Exchange protocol is built on top of libp2p (Codex protocol for short), is it good to grasp some basics of how protocols are generally implemented on top of libp2p. [Simple ping tutorial](https://vacp2p.github.io/nim-libp2p/docs/tutorial_1_connect/) and [Custom protocol in libp2p](https://vacp2p.github.io/nim-libp2p/docs/tutorial_2_customproto/), and [Protobuf usage](https://vacp2p.github.io/nim-libp2p/docs/tutorial_3_protobuf/) together with introduction to the [Switch](https://docs.libp2p.io/concepts/multiplex/switch/) component are good introductory reads, without which it may be kind of hard to understand the high-level structure of the Codex client.

To quickly summarize, starting a P2P node with a custom protocol using libp2p can be described as follows.

We derive our protocol type (e.g. `TestProto`) from `LPProtocol`:

```nim
const TestCodec = "/test/proto/1.0.0"

type TestProto = ref object of LPProtocol
```

We initialize our `TestProto` providing our `codecs` and a `handler`, which will be called for each incoming peer asking for this protocol:

```nim
proc new(T: typedesc[TestProto]): T =
  # every incoming connections will in be handled in this closure
  proc handle(conn: Connection, proto: string) {.async.} =
    # Read up to 1024 bytes from this connection, and transform them into
    # a string
    echo "Got from remote - ", string.fromBytes(await conn.readLp(1024))
    # We must close the connections ourselves when we're done with it
    await conn.close()

  return T.new(codecs = @[TestCodec], handler = handle)
```

Then, we create a *switch*, e.g:

```nim
proc createSwitch(ma: MultiAddress, rng: ref HmacDrbgContext): Switch =
  var switch = SwitchBuilder
    .new()
    .withRng(rng)
    # Give the application RNG
    .withAddress(ma)
    # Our local address(es)
    .withTcpTransport()
    # Use TCP as transport
    .withMplex()
    # Use Mplex as muxer
    .withNoise()
    # Use Noise as secure manager
    .build()

  return switch
```

And finally, we tie everything up and trigger a simple communication between two peers:

```nim
proc main() {.async.} =
  let
    rng = newRng()
    localAddress = MultiAddress.init("/ip4/0.0.0.0/tcp/0").tryGet()
    testProto = TestProto.new()
    switch1 = createSwitch(localAddress, rng)
    switch2 = createSwitch(localAddress, rng)

  switch1.mount(testProto)

  await switch1.start()
  await switch2.start()

  let conn =
    await switch2.dial(switch1.peerInfo.peerId, switch1.peerInfo.addrs, TestCodec)

  await testProto.hello(conn)

  # We must close the connection ourselves when we're done with it
  await conn.close()

  await allFutures(switch1.stop(), switch2.stop())
    # close connections and shutdown all transports
```

Now, let's find the analogous steps in our Codex client.

### Codex

In `codex.nim`, we have:

```nim
let
  keyPath =
    if isAbsolute(config.netPrivKeyFile):
      config.netPrivKeyFile
    else:
     config.dataDir / config.netPrivKeyFile

  privateKey = setupKey(keyPath).expect("Should setup private key!")
  server = try:
    CodexServer.new(config, privateKey)
  except Exception as exc:
    error "Failed to start Codex", msg = exc.msg
    quit QuitFailure
```

and later:

```nim
waitFor server.start()
```

It is in `CodexServer.new` (`codex/codex.nim`) where we create our *switch*:

```nim
proc new*(
    T: type CodexServer, config: CodexConf, privateKey: CodexPrivateKey
): CodexServer =
  ## create CodexServer including setting up datastore, repostore, etc
  let switch = SwitchBuilder
    .new()
    .withPrivateKey(privateKey)
    .withAddresses(config.listenAddrs)
    .withRng(Rng.instance())
    .withNoise()
    .withMplex(5.minutes, 5.minutes)
    .withMaxConnections(config.maxPeers)
    .withAgentVersion(config.agentString)
    .withSignedPeerRecord(true)
    .withTcpTransport({ServerFlags.ReuseAddr})
    .build()
```

A moment later, in the same proc, we have:

```nim
let network = BlockExcNetwork.new(switch)
```

and then close to the end:

```nim
switch.mount(network)
```

Finally, in `CodexServer.start`

```nim
proc start*(s: CodexServer) {.async.} =
  trace "Starting codex node", config = $s.config

  await s.repoStore.start()
  s.maintenance.start()

  await s.codexNode.switch.start()
```

Thus, `BlockExcNetwork` is our protocol type (`codex/blockexchange/network/network.nim`):

```nim
type BlockExcNetwork* = ref object of LPProtocol
  peers*: Table[PeerId, NetworkPeer]
  switch*: Switch
  handlers*: BlockExcHandlers
  request*: BlockExcRequest
  getConn: ConnProvider
  inflightSema: AsyncSemaphore
```

In the constructor, `BlockExcNetwork.new`, a couple of request functions are defined and attached to `self.request` (`BlockExcRequest` type). Then, `self.init()` is called:

```nim
method init*(b: BlockExcNetwork) =
  ## Perform protocol initialization
  ##

  proc peerEventHandler(peerId: PeerId, event: PeerEvent) {.async.} =
    if event.kind == PeerEventKind.Joined:
      b.setupPeer(peerId)
    else:
      b.dropPeer(peerId)

  b.switch.addPeerEventHandler(peerEventHandler, PeerEventKind.Joined)
  b.switch.addPeerEventHandler(peerEventHandler, PeerEventKind.Left)

  proc handle(conn: Connection, proto: string) {.async, gcsafe, closure.} =
    let peerId = conn.peerId
    let blockexcPeer = b.getOrCreatePeer(peerId)
    await blockexcPeer.readLoop(conn) # attach read loop

  b.handler = handle
  b.codec = Codec
```

Here we see the familiar `handler` and `codec` being set. Let's have closer look at `getOrCreatePeer`:

```nim
proc getOrCreatePeer(b: BlockExcNetwork, peer: PeerId): NetworkPeer =
  ## Creates or retrieves a BlockExcNetwork Peer
  ##

  if peer in b.peers:
    return b.peers.getOrDefault(peer, nil)

  var getConn: ConnProvider = proc(): Future[Connection] {.async, gcsafe, closure.} =
    try:
      return await b.switch.dial(peer, Codec)
    except CancelledError as error:
      raise error
    except CatchableError as exc:
      trace "Unable to connect to blockexc peer", exc = exc.msg

  if not isNil(b.getConn):
    getConn = b.getConn

  let rpcHandler = proc(p: NetworkPeer, msg: Message) {.async.} =
    b.rpcHandler(p, msg)

  # create new pubsub peer
  let blockExcPeer = NetworkPeer.new(peer, getConn, rpcHandler)
  debug "Created new blockexc peer", peer

  b.peers[peer] = blockExcPeer

  return blockExcPeer
```

Here we recognize the familiar `dial` operation, and we see a new abstraction - `NetworkPeer` - representing the peer. In the `NetworkPeer` we find the `readLoop` defined above in the protocol handler:

```nim
proc readLoop*(b: NetworkPeer, conn: Connection) {.async.} =
  if isNil(conn):
    return

  try:
    while not conn.atEof or not conn.closed:
      let
        data = await conn.readLp(MaxMessageSize.int)
        msg = Message.protobufDecode(data).mapFailure().tryGet()
      await b.handler(b, msg)
  except CancelledError:
    trace "Read loop cancelled"
  except CatchableError as err:
    warn "Exception in blockexc read loop", msg = err.msg
  finally:
    await conn.close()
```

We read from the connection, decode the message (see [[Protocol of data exchange between Codex nodes]]), forward it down to the `handler`, which is the `rpcHandler` we see above in `getOrCreatePeer`. `rpcHandler` is then defined at the protocol level (`BlockExcNetwork`) as:

```nim
proc rpcHandler(b: BlockExcNetwork, peer: NetworkPeer, msg: Message) {.raises: [].} =
  ## handle rpc messages
  ##
  if msg.wantList.entries.len > 0:
    asyncSpawn b.handleWantList(peer, msg.wantList)

  if msg.payload.len > 0:
    asyncSpawn b.handleBlocksDelivery(peer, msg.payload)

  if msg.blockPresences.len > 0:
    asyncSpawn b.handleBlockPresence(peer, msg.blockPresences)

  if account =? Account.init(msg.account):
    asyncSpawn b.handleAccount(peer, account)

  if payment =? SignedState.init(msg.payment):
    asyncSpawn b.handlePayment(peer, payment)
```

Let's focus for a moment on `handleBlocksDelivery`:

```nim
proc handleBlocksDelivery(
    b: BlockExcNetwork, peer: NetworkPeer, blocksDelivery: seq[BlockDelivery]
) {.async.} =
  ## Handle incoming blocks
  ##

  if not b.handlers.onBlocksDelivery.isNil:
    await b.handlers.onBlocksDelivery(peer.id, blocksDelivery)
```

What are `handlers`? It is an instance of `BlockExcHandlers` set in `BlockExcEngine.new` (`codex/blockexchange/engine/engine.nim`). There, `onBlockDelivery` member is set to:

```nim
proc blocksDeliveryHandler(
      peer: PeerId, blocksDelivery: seq[BlockDelivery]
  ): Future[void] {.gcsafe.} =
    engine.blocksDeliveryHandler(peer, blocksDelivery)
```

which forwards the `seq[BlockDelivery]` payload to:

```nim
proc blocksDeliveryHandler*(
    b: BlockExcEngine, peer: PeerId, blocksDelivery: seq[BlockDelivery]
) {.async.} =
  trace "Received blocks from peer", peer, blocks = (blocksDelivery.mapIt(it.address))

  var validatedBlocksDelivery: seq[BlockDelivery]
  for bd in blocksDelivery:
    logScope:
      peer = peer
      address = bd.address

    if err =? b.validateBlockDelivery(bd).errorOption:
      warn "Block validation failed", msg = err.msg
      continue

    if err =? (await b.localStore.putBlock(bd.blk)).errorOption:
      error "Unable to store block", err = err.msg
      continue

    if bd.address.leaf:
      without proof =? bd.proof:
        error "Proof expected for a leaf block delivery"
        continue
      if err =? (
        await b.localStore.putCidAndProof(
          bd.address.treeCid, bd.address.index, bd.blk.cid, proof
        )
      ).errorOption:
        error "Unable to store proof and cid for a block"
        continue

    validatedBlocksDelivery.add(bd)

  await b.resolveBlocks(validatedBlocksDelivery)
  codex_block_exchange_blocks_received.inc(validatedBlocksDelivery.len.int64)

  let peerCtx = b.peers.get(peer)

  if peerCtx != nil:
    await b.payForBlocks(peerCtx, blocksDelivery)
    ## shouldn't we remove them from the want-list instead of this:
    peerCtx.cleanPresence(blocksDelivery.mapIt(it.address))
```

Here we see that each received block is [[Codex Block Validation|validated]] and then `resolveBlocks` is called:

```nim
proc resolveBlocks*(b: BlockExcEngine, blocksDelivery: seq[BlockDelivery]) {.async.} =
  b.pendingBlocks.resolve(blocksDelivery)
  await b.scheduleTasks(blocksDelivery)
  await b.cancelBlocks(blocksDelivery.mapIt(it.address))
```

