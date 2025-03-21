For the cleaner and more focused logging, it is adviced to compile the codex client using the following options in `build.nims`:

```nim
task codex, "build codex binary":
  buildBinary "codex",
    # default params
    # params = "-d:chronicles_runtime_filtering -d:chronicles_log_level=TRACE"
    params =
      "-d:chronicles_runtime_filtering -d:chronicles_log_level=TRACE -d:chronicles_enabled_topics:restapi:TRACE,node:TRACE"
```

In the example session, we will use two nodes:

- node-1 will be where we will upload (seed) the content
- node-2 will be from where we will be downloading the previously uploaded content.

Start node-1:

```bash
./build/codex --data-dir=./data-1 --listen-addrs=/ip4/127.0.0.1/tcp/8081 --api-port=8001 --nat=none --disc-port=8091 --log-level=TRACE
```


Generate fresh content:

```bash
dd if=/dev/urandom of=./data10M.bin bs=10M count=1
```

Upload content to node-1:

```bash
curl -X POST \
  http://localhost:8001/api/codex/v1/torrent \
  -H 'Content-Type: application/octet-stream' \
  -H 'Content-Disposition: filename="data10M.bin"' \
  -w '\n' \
  -T data10M.bin
11144249FFB943675890CF09342629CD3782D107B709
```

Set `info_hash` env var:

```bash
export INFO_HASH=11144249FFB943675890CF09342629CD3782D107B709
```

Start node-2:

```bash
./build/codex --data-dir=./data-2 --listen-addrs=/ip4/127.0.0.1/tcp/8082 --api-port=8002 --nat=none --disc-port=8092 --log-level=TRACE
```

Make sure that node-1 and node-2 are connect (should be automatic but sometimes we need to trigger it when running on localhost without nat).

Get the peerId from node-1

```bash
curl -H 'Accept: text/plain' 'http://localhost:8001/api/codex/v1/peerid' --write-out '\n'
16Uiu2HAmSfUAJfzaMadXynZCeVza4gFMR6krZM4mnZjYU6tQ2jog
```

Here it is also good to use env var:

```bash
export PEERID_NODE1=$(curl -H 'Accept: text/plain' 'http://localhost:8001/api/codex/v1/peerid')
```

Connect node-2 to node-1:

```bash
curl "http://localhost:8002/api/codex/v1/connect/${PEERID_NODE1}?addrs=/ip4/127.0.0.1/tcp/8081"
Successfully connected to peer
```

Make sure `INFO_HASH` env var is set:

```bash
export INFO_HASH=11144249FFB943675890CF09342629CD3782D107B709
```

Stream the content from node-2:

```bash
curl "http://localhost:8002/api/codex/v1/torrent/${INFO_HASH}/network/stream" -o "${INFO_HASH}.bin"
```

And to get just the torrent manifest:

```bash
curl "http://localhost:8002/api/codex/v1/torrent/${INFO_HASH}/network/manifest" | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2113  100  2113    0     0   853k      0 --:--:-- --:--:-- --:--:-- 1031k
{
  "infoHash": "111404010165844CF71179B42AC3C0462E31EB9E74B1",
  "torrentManifest": {
    "info": {
      "length": 10485760,
      "pieceLength": 262144,
      "pieces": [
        "111494BFE9E914669431DAF5446732E399EF3533820F",
        "1114DFB0389F65DBB285B3FB21B057408B5CAC27A188",
        "111417798F99E9CC9F6C0A96C00C667ED89A8187CEE9",
        "11146CFE9BFE2901BBA9DE573A0FE3613F77E26053FD",
        "111447E8129FEACB16613C77F687F7ABE88A7D3AF34F",
        "11140051FC4683B1E98CA24AD32ED05C0B6A02A9CE35",
        "11149C67E50A767DF3F910FB872C887B560B9E59BC64",
        "11141AC926FE56830A1DF069F979E4B2E6EE8E80E02B",
        "11146E372C5814D5EAEC958ACF15A09A25DDD254FD5F",
        "1114A494C6185B2C6697898F2B7519357DAAB52CCE4E",
        "11148FE075ED85EE50D056DB82A4C6802AC145AB14C6",
        "111479745CDC0A973D19A86EF8A7CA6BBF0A066183DE",
        "1114EB3C71D98ED8C51D8705A3562FFC816787B2BB63",
        "11146C4486F224B217BA6C7DDB41874631862AA987B3",
        "1114E32FFD30458E384E4B5C4311EC6CD60806B9E6E2",
        "1114698D13326C5DF59A95345D1FC5FEF4EE50A27EA1",
        "11147446B0137417EDBDE1EA4CAC5DB8F9A8DD30DDAE",
        "11149C1F5C49EB23134ECBB1007C3964F9D6DBC601A0",
        "1114656F38874EDD3E7D18949C1F716FECFE0DAE6D48",
        "1114AFDB1ED7DBA0DA42567F369B4D009D2917803711",
        "1114384092051DBBBE7BA04815925941C8674F2FD5A4",
        "111453026DA2899EB0AD8E7285B8CE2FD837513CEEBE",
        "1114F4A106192FE40C6ECF94D77DE62297AC3B4CFBC2",
        "1114E7D6C4633AFA8966819358179531352E35587807",
        "1114D16F9822A32589C3AFBD73A51819B3261AAD8E67",
        "1114985F1B5C2420F9984D285D2884B999B53031B462",
        "11145DE360A86D1105951F346F48350720A3E665A800",
        "111400E9988CACF7F0C637B2AE2CFD0A291FF7FD05C8",
        "1114D2EE427BA371FC157F30BC3467C87AC00581372B",
        "1114F00AE60ED70619C1829BF2E8261226D0467F175C",
        "111450650C77BACCE80C8833A1FCE3565D7252BDA8E2",
        "11141CE38B812A5A07AAEB99878C207146CA8181EC55",
        "11142503901ED89ED51E927873D39F5DCDD44F454DF2",
        "11143FBB54D2ECB2AB33953F090DF33279BBE20FC5B8",
        "1114048E4CE97C3DB74BD5F680F87E58FBCCA9A54EDD",
        "11149CF6AF5F6E71EAB600A4E156F27C189651D6C8D2",
        "1114467578C62EED05FDCF29BABD3CD6989709B3E419",
        "1114183914076A0BF45133D56FBFF9FC510AA76E4462",
        "11141012A7A054710B316DBA414EED131A17508CCC96",
        "1114D680DBBC01022878727D41FB86A1954474026139"
      ],
      "name": "data10M.bin"
    },
    "codexManifestCid": "zDvZRwzm5J2n6S12QWCHm3PdiavyQtLbC73txVr1DQoRUhyLKyUA"
  }
}
```

And here is the streaming log (node-2) for the reference:

```bash
TRC 2025-03-20 03:32:28.939+01:00 torrent requested:                         topics="codex restapi" tid=6977524 multihash=sha1/04010165844CF71179B42AC3C0462E31EB9E74B1
TRC 2025-03-20 03:32:28.940+01:00 Retrieving torrent manifest for infoHashCid topics="codex node" tid=6977524 infoHashCid=z8q*sJUUWC
TRC 2025-03-20 03:32:28.946+01:00 Successfully retrieved torrent manifest with given block cid topics="codex node" tid=6977524 cid=z8q*sJUUWC infoHashCid=z8q*sJUUWC
TRC 2025-03-20 03:32:28.946+01:00 Decoding torrent manifest                  topics="codex node" tid=6977524
TRC 2025-03-20 03:32:28.946+01:00 Decoded torrent manifest                   topics="codex node" tid=6977524 infoHashCid=z8q*sJUUWC torrentManifest="BitTorrentManifest(info: BitTorrentInfo(length: 10485760, pieceLength: 262144, pieces: @[sha1/94BFE9E914669431DAF5446732E399EF3533820F, sha1/DFB0389F65DBB285B3FB21B057408B5CAC27A188, sha1/17798F99E9CC9F6C0A96C00C667ED89A8187CEE9, sha1/6CFE9BFE2901BBA9DE573A0FE3613F77E26053FD, sha1/47E8129FEACB16613C77F687F7ABE88A7D3AF34F, sha1/0051FC4683B1E98CA24AD32ED05C0B6A02A9CE35, sha1/9C67E50A767DF3F910FB872C887B560B9E59BC64, sha1/1AC926FE56830A1DF069F979E4B2E6EE8E80E02B, sha1/6E372C5814D5EAEC958ACF15A09A25DDD254FD5F, sha1/A494C6185B2C6697898F2B7519357DAAB52CCE4E, sha1/8FE075ED85EE50D056DB82A4C6802AC145AB14C6, sha1/79745CDC0A973D19A86EF8A7CA6BBF0A066183DE, sha1/EB3C71D98ED8C51D8705A3562FFC816787B2BB63, sha1/6C4486F224B217BA6C7DDB41874631862AA987B3, sha1/E32FFD30458E384E4B5C4311EC6CD60806B9E6E2, sha1/698D13326C5DF59A95345D1FC5FEF4EE50A27EA1, sha1/7446B0137417EDBDE1EA4CAC5DB8F9A8DD30DDAE, sha1/9C1F5C49EB23134ECBB1007C3964F9D6DBC601A0, sha1/656F38874EDD3E7D18949C1F716FECFE0DAE6D48, sha1/AFDB1ED7DBA0DA42567F369B4D009D2917803711, sha1/384092051DBBBE7BA04815925941C8674F2FD5A4, sha1/53026DA2899EB0AD8E7285B8CE2FD837513CEEBE, sha1/F4A106192FE40C6ECF94D77DE62297AC3B4CFBC2, sha1/E7D6C4633AFA8966819358179531352E35587807, sha1/D16F9822A32589C3AFBD73A51819B3261AAD8E67, sha1/985F1B5C2420F9984D285D2884B999B53031B462, sha1/5DE360A86D1105951F346F48350720A3E665A800, sha1/00E9988CACF7F0C637B2AE2CFD0A291FF7FD05C8, sha1/D2EE427BA371FC157F30BC3467C87AC00581372B, sha1/F00AE60ED70619C1829BF2E8261226D0467F175C, sha1/50650C77BACCE80C8833A1FCE3565D7252BDA8E2, sha1/1CE38B812A5A07AAEB99878C207146CA8181EC55, sha1/2503901ED89ED51E927873D39F5DCDD44F454DF2, sha1/3FBB54D2ECB2AB33953F090DF33279BBE20FC5B8, sha1/048E4CE97C3DB74BD5F680F87E58FBCCA9A54EDD, sha1/9CF6AF5F6E71EAB600A4E156F27C189651D6C8D2, sha1/467578C62EED05FDCF29BABD3CD6989709B3E419, sha1/183914076A0BF45133D56FBFF9FC510AA76E4462, sha1/1012A7A054710B316DBA414EED131A17508CCC96, sha1/D680DBBC01022878727D41FB86A1954474026139], name: some(\"data10M.bin\")), codexManifestCid: zDvZRwzm5J2n6S12QWCHm3PdiavyQtLbC73txVr1DQoRUhyLKyUA)"
TRC 2025-03-20 03:32:28.946+01:00 Retrieving manifest for cid                topics="codex node" tid=6977524 cid=zDv*yLKyUA
TRC 2025-03-20 03:32:28.949+01:00 Decoding manifest for cid                  topics="codex node" tid=6977524 cid=zDv*yLKyUA
TRC 2025-03-20 03:32:28.949+01:00 Decoded manifest                           topics="codex node" tid=6977524 cid=zDv*yLKyUA
TRC 2025-03-20 03:32:28.949+01:00 Retrieving pieces from torrent             topics="codex node" tid=6977524
TRC 2025-03-20 03:32:28.950+01:00 Creating store stream for torrent manifest topics="codex node" tid=6977524
TRC 2025-03-20 03:32:28.950+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:28.973+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:28.974+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=0
TRC 2025-03-20 03:32:28.974+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=0
TRC 2025-03-20 03:32:28.982+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.007+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.008+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=1
TRC 2025-03-20 03:32:29.008+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=1
TRC 2025-03-20 03:32:29.014+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.026+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.026+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=2
TRC 2025-03-20 03:32:29.026+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=2
TRC 2025-03-20 03:32:29.033+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.056+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.056+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=3
TRC 2025-03-20 03:32:29.056+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=3
TRC 2025-03-20 03:32:29.063+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.074+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.074+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=4
TRC 2025-03-20 03:32:29.074+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=4
TRC 2025-03-20 03:32:29.081+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.091+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.092+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=5
TRC 2025-03-20 03:32:29.092+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=5
TRC 2025-03-20 03:32:29.097+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.119+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.119+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=6
TRC 2025-03-20 03:32:29.119+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=6
TRC 2025-03-20 03:32:29.126+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.138+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.139+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=7
TRC 2025-03-20 03:32:29.139+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=7
TRC 2025-03-20 03:32:29.145+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.169+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.169+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=8
TRC 2025-03-20 03:32:29.169+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=8
TRC 2025-03-20 03:32:29.176+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.195+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.196+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=9
TRC 2025-03-20 03:32:29.196+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=9
TRC 2025-03-20 03:32:29.201+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.222+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.223+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=10
TRC 2025-03-20 03:32:29.223+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=10
TRC 2025-03-20 03:32:29.230+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.241+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.241+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=11
TRC 2025-03-20 03:32:29.241+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=11
TRC 2025-03-20 03:32:29.247+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.270+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.270+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=12
TRC 2025-03-20 03:32:29.270+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=12
TRC 2025-03-20 03:32:29.277+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.288+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.288+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=13
TRC 2025-03-20 03:32:29.288+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=13
TRC 2025-03-20 03:32:29.295+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.318+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.318+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=14
TRC 2025-03-20 03:32:29.318+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=14
TRC 2025-03-20 03:32:29.324+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.336+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.336+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=15
TRC 2025-03-20 03:32:29.336+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=15
TRC 2025-03-20 03:32:29.343+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.371+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.371+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=16
TRC 2025-03-20 03:32:29.371+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=16
TRC 2025-03-20 03:32:29.378+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.388+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.389+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=17
TRC 2025-03-20 03:32:29.389+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=17
TRC 2025-03-20 03:32:29.395+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.418+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.418+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=18
TRC 2025-03-20 03:32:29.418+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=18
TRC 2025-03-20 03:32:29.425+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.436+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.436+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=19
TRC 2025-03-20 03:32:29.436+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=19
TRC 2025-03-20 03:32:29.443+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.465+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.465+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=20
TRC 2025-03-20 03:32:29.465+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=20
TRC 2025-03-20 03:32:29.472+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.484+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.484+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=21
TRC 2025-03-20 03:32:29.484+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=21
TRC 2025-03-20 03:32:29.491+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.513+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.514+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=22
TRC 2025-03-20 03:32:29.514+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=22
TRC 2025-03-20 03:32:29.518+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.535+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.536+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=23
TRC 2025-03-20 03:32:29.536+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=23
TRC 2025-03-20 03:32:29.542+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.564+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.564+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=24
TRC 2025-03-20 03:32:29.564+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=24
TRC 2025-03-20 03:32:29.569+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.595+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.595+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=25
TRC 2025-03-20 03:32:29.595+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=25
TRC 2025-03-20 03:32:29.602+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.633+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.633+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=26
TRC 2025-03-20 03:32:29.633+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=26
TRC 2025-03-20 03:32:29.640+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.663+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.663+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=27
TRC 2025-03-20 03:32:29.663+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=27
TRC 2025-03-20 03:32:29.668+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.688+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.688+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=28
TRC 2025-03-20 03:32:29.688+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=28
TRC 2025-03-20 03:32:29.695+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.716+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.717+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=29
TRC 2025-03-20 03:32:29.717+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=29
TRC 2025-03-20 03:32:29.722+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.737+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.738+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=30
TRC 2025-03-20 03:32:29.738+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=30
TRC 2025-03-20 03:32:29.745+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.767+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.768+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=31
TRC 2025-03-20 03:32:29.768+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=31
TRC 2025-03-20 03:32:29.773+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.789+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.790+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=32
TRC 2025-03-20 03:32:29.790+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=32
TRC 2025-03-20 03:32:29.796+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.818+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.818+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=33
TRC 2025-03-20 03:32:29.818+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=33
TRC 2025-03-20 03:32:29.823+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.838+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.838+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=34
TRC 2025-03-20 03:32:29.838+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=34
TRC 2025-03-20 03:32:29.845+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.866+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.867+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=35
TRC 2025-03-20 03:32:29.867+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=35
TRC 2025-03-20 03:32:29.872+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.890+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.891+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=36
TRC 2025-03-20 03:32:29.891+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=36
TRC 2025-03-20 03:32:29.897+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.919+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.920+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=37
TRC 2025-03-20 03:32:29.920+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=37
TRC 2025-03-20 03:32:29.926+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.943+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.943+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=38
TRC 2025-03-20 03:32:29.943+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=38
TRC 2025-03-20 03:32:29.950+01:00 Waiting for piece...                       topics="codex restapi" tid=6977524
TRC 2025-03-20 03:32:29.973+01:00 Fetched complete torrent piece - verifying... topics="codex node" tid=6977524
TRC 2025-03-20 03:32:29.973+01:00 Piece successfully verified                topics="codex node" tid=6977524 pieceIndex=39
TRC 2025-03-20 03:32:29.973+01:00 Got piece                                  topics="codex restapi" tid=6977524 pieceIndex=39
INF 2025-03-20 03:32:29.978+01:00 Sent bytes for torrent                     topics="codex restapi" tid=6977524 infoHash=sha1/04010165844CF71179B42AC3C0462E31EB9E74B1 bytes=10485760
```