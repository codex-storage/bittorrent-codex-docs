See also: [[Using Python on macOS]].

We will use a very simple Python script to validate encodings. You can find the script here: https://gist.github.com/marcinczenko/a87baf506bb0fd4bdff752cda4c685a1

Our input will be the following `data1M.bin.torrent.json` file:

```nim
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
    ]
  }
}
```

To *b-encode* we run:

```bash
python bencoder.py --info-only data1M.bin.torrent.json
Using values from file: data1M.bin.torrent.json
Info only: True
{
  "length": 1048576,
  "name": "data1M.bin",
  "piece length": 262144,
  "pieces": [
    "21FEBA308CD51E9ACF88417193A9EA60F0F84646",
    "3D4A8279853DA2DA355A574740217D446506E8EB",
    "1AD686B48B9560B15B8843FD00E7EC1B59624B09",
    "5015E7DA0C40350624C6B5A1FED1DB39720B726C"
  ]
}
infohash f335440998515770adf47e8a4626889e59d91dde
```

The `--info-only` option instructs the script to only b-encode the info dictionary and do not include other attributes like `announce`.

We get as an output the `data1M.bin.torrent` file with the following content:

```bash
d4:infod6:lengthi1048576e4:name10:data1M.bin12:piece lengthi262144e6:pieces80:!��0���ψAq���`��FF=J�y�=��5ZWG@!}De��ֆ���`�[�C� ��YbK	P��@5$Ƶ����9rrlee
```

It is not very readable, because pieces has been converted to hex strings to a flattened byte array - in the end this is supposed to be machine readable code.