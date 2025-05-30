For the convenience, you can also use a torrent file when downloading the content from the Codex network. After the content has been uploaded to the Codex network, the same torrent file can be used to refer to the content on the BitTorrent network as well as on the Codex network. In the end, both will result in the same `info` attribute and the same `info` hash.

> [!note]
> `.torrent` files are b-encoded, thus machine-readable. For even more convenience, we also support human-readable torrent files in JSON format.

Let's create an example torrent file. To keep it compact for the purpose of the clarity of this document, lets first upload a small file to our Codex network:

```bash
curl -X POST \
  http://localhost:8001/api/codex/v1/torrent \
  -H 'Content-Type: application/octet-stream' \
  -H 'Content-Disposition: filename="data1M.bin"' \
  -w '\n' \
  -T data1M.bin
1114F335440998515770ADF47E8A4626889E59D91DDE
```

Now let's retrieve the corresponding torrent manifest:

```bash
export INFO_HASH=1114F335440998515770ADF47E8A4626889E59D91DDE
curl "http://localhost:8002/api/codex/v1/torrent/${INFO_HASH}/network/manifest" | jq
{
  "infoHash": "1114F335440998515770ADF47E8A4626889E59D91DDE",
  "torrentManifest": {
    "info": {
      "length": 1048576,
      "pieceLength": 262144,
      "pieces": [
        "111421FEBA308CD51E9ACF88417193A9EA60F0F84646",
        "11143D4A8279853DA2DA355A574740217D446506E8EB",
        "11141AD686B48B9560B15B8843FD00E7EC1B59624B09",
        "11145015E7DA0C40350624C6B5A1FED1DB39720B726C"
      ],
      "name": "data1M.bin"
    },
    "codexManifestCid": "zDvZRwzmAzLRUfd3NGQ5JSP42ATgeVBeY9RE1CgYTWZDtPTDZcqz"
  }
}
```

The `info` part of the returned torrent manifest is already a valid torrent file in JSON encoding:

```json
{
  "info": {
    "length": 1048576,
    "name": "data1M.bin",
    "pieceLength": 262144,
    "pieces": [
	  "111421FEBA308CD51E9ACF88417193A9EA60F0F84646",
	  "11143D4A8279853DA2DA355A574740217D446506E8EB",
	  "11141AD686B48B9560B15B8843FD00E7EC1B59624B09",
	  "11145015E7DA0C40350624C6B5A1FED1DB39720B726C"
    ],    
  },
}
```

In the JSON encoding the order of the attributes does not matter: our client will sort the attributes accordingly in order to compute the correct corresponding `info` hash. 

Let's put the JSON representation of the torrent file into `data1M.bin.torrent.json`.

Because only `info` dictionary is used to compute the corresponding `info` hash, we do not have to include any other torrent attributes (e.g. `announce` attribute used by trackers).

> [!note]
> We currently do not support directories, thus we do not need to support the `files` attribute.

Also notice that the `name` attribute has purely advisory character - it is a suggested name to save a file or dictionary, but since we are returning the content via API, we do not have to observe this attribute. Yet, this attribute must be present and its value must be kept intact as changing it will affect the corresponding `info` hash value.

Our API allows us to use the JSON representation of the torrent file in order to retrieve the corresponding content:

```bash
curl -X POST \
  http://localhost:8001/api/codex/v1/torrent/torrent-file \
  -H 'Content-Type: application/json' \
  -w '\n' \
  -T data1M.bin.torrent.json -o ${INFO_HASH}-JSON.bin
```

We can check that the fetched file content is identical to `data1M.bin`:

```bash
openssl sha1 data1M.bin
SHA1(data1M.bin)= f0168f6ff31fd7511bdbcf7f70fecce959f321e7
openssl sha1 1114F335440998515770ADF47E8A4626889E59D91DDE-JSON.bin
SHA1(1114F335440998515770ADF47E8A4626889E59D91DDE-JSON.bin)= f0168f6ff31fd7511bdbcf7f70fecce959f321e7
```

Now let's create the corresponding binary version of the torrent file and use it to fetch the same content.

Having the torrent manifest, what we need to create the corresponding BitTorrent v1 torrent file is the content of the info dictionary. We need to change the name of the `pieceLength` attribute to `piece length` as required by the BitTorrent specs.

```json
{
  "info": {
    "length": 1048576,
    "name": "data1M.bin",
    "piece length": 262144,
    "pieces": [
	  "111421FEBA308CD51E9ACF88417193A9EA60F0F84646",
	  "11143D4A8279853DA2DA355A574740217D446506E8EB",
	  "11141AD686B48B9560B15B8843FD00E7EC1B59624B09",
	  "11145015E7DA0C40350624C6B5A1FED1DB39720B726C"
    ],    
  },
}
```



In order to convert this human readable, JSON representation of the `info` dictionary, we can use a Python script which we show in [[B-encoding using Python]]. Our encoder will enforce the correct order of the attributes while encoding.

The resulting b-encoded Torrent file content will be:

```bash
d4:infod6:lengthi1048576e4:name10:data1M.bin12:piece lengthi262144e6:pieces80:!��0���ψAq���`��FF=J�y�=��5ZWG@!}De��ֆ���`�[�C� ��YbK	P��@5$Ƶ����9rrlee
```

The encoder will also return the corresponding `info` hash value:

```bash
f335440998515770adf47e8a4626889e59d91dde
```

It should match the `info` value we got after uploading the content.

Putting the b-encoded torrent file in `data1M.bin.torrent`, we can use it directly the retrieve the corresponding content:

```bash
curl -X POST \
  http://localhost:8001/api/codex/v1/torrent/torrent-file \
  -H 'Content-Type: application/octet-stream' \
  -w '\n' \
  -T data1M.bin.torrent -o ${INFO_HASH}.bin
```

Also here we can see that the retrieved content matches the original:

```bash
openssl sha1 data1M.bin
SHA1(data1M.bin)= f0168f6ff31fd7511bdbcf7f70fecce959f321e7
openssl sha1 1114F335440998515770ADF47E8A4626889E59D91DDE.bin
SHA1(1114F335440998515770ADF47E8A4626889E59D91DDE.bin)= f0168f6ff31fd7511bdbcf7f70fecce959f321e7
```
