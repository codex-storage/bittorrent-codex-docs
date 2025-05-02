## Uploading

The following setup has been used for testing:

```bash
» tree ./dir   
./dir
├── dir1
│   ├── file11.txt # File 11
│   └── file12.txt # File 12
├── file1.txt      # File 1
└── file2.txt.     # File 2

2 directories, 4 files
```

It is included in `tests/fixtures/tarballs/dir`.

The content has been archived using `tar`:

```bash
» tar -cf tar -cf testtarball.tar dir
» tar -tf testtarbar.tar 
dir/
dir/file2.txt
dir/file1.txt
dir/dir1/
dir/dir1/file11.txt
dir/dir1/file12.txt
```

A codex node has been started:

```bash
./build/codex --data-dir=./data-1 --listen-addrs=/ip4/127.0.0.1/tcp/8081 --api-port=8001 --nat=none --disc-port=8091 --log-level=TRACE
```

The tarball has been uploaded to codex using new `tar` API (the output shows reponse):

```bash
» curl -X POST http://localhost:8001/api/codex/v1/tar \
  -H 'Content-Type: application/octet-stream' \
  -H 'Content-Disposition: filename="testtarbar.tar"' \
  -w '\n' \
  -T testtarbar.tar | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  6654  100   510  100  6144  53297   627k --:--:-- --:--:-- --:--:--  722k
[
  {
    "name": "dir",
    "cid": "zDvZRwzm597U7Bq9rDZ29KpqodiGHfAuLqMTXixujUFT3QYNDjgj",
    "children": [
      {
        "name": "file2.txt",
        "cid": "zDvZRwzm5oLGkg8kW7g6fRZSfmCqqc44JAUivqJ5TwrwZneCh6V5"
      },
      {
        "name": "file1.txt",
        "cid": "zDvZRwzmDJSePX8bskWzKUrQPDutLiFhCuDtenAZDnpMZ52LUCWh"
      },
      {
        "name": "dir1",
        "cid": "zDvZRwzkzNF8iJKdSXGJuoiC2pJ1sfrA9brunjhNTts3PrEQ92fs",
        "children": [
          {
            "name": "file11.txt",
            "cid": "zDvZRwzm8ZuLQB7kG3fcZqqNRPXbhe2d4du6pDC9MyoRRw12GKfS"
          },
          {
            "name": "file12.txt",
            "cid": "zDvZRwzmDd6Mg6E98nrXVkZ7THfK5zyQ6Y3j8fSSavSsAYReteLT"
          }
        ]
      }
    ]
  }
]
```

We see that each directory and file gets their separate cids. The cids of the files are regular Codex Manifest cids. The cids for the directories, are cids of *Codex Directory Manifest*. You can retrieve the directory manifest using `dirmanifest` API:

```bash
» curl "http://localhost:8001/api/codex/v1/data/zDvZRwzm597U7Bq9rDZ29KpqodiGHfAuLqMTXixujUFT3QYNDjgj/network/dirmanifest" | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   262  100   262    0     0   154k      0 --:--:-- --:--:-- --:--:--  255k
{
  "cid": "zDvZRwzm597U7Bq9rDZ29KpqodiGHfAuLqMTXixujUFT3QYNDjgj",
  "manifest": {
    "name": "dir",
    "cids": [
      "zDvZRwzm5oLGkg8kW7g6fRZSfmCqqc44JAUivqJ5TwrwZneCh6V5",
      "zDvZRwzmDJSePX8bskWzKUrQPDutLiFhCuDtenAZDnpMZ52LUCWh",
      "zDvZRwzkzNF8iJKdSXGJuoiC2pJ1sfrA9brunjhNTts3PrEQ92fs"
    ]
  }
}
```

The same for the subdirectory `dir1`:

```bash
» curl "http://localhost:8001/api/codex/v1/data/zDvZRwzkzNF8iJKdSXGJuoiC2pJ1sfrA9brunjhNTts3PrEQ92fs/network/dirmanifest" | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   208  100   208    0     0   155k      0 --:--:-- --:--:-- --:--:--  203k
{
  "cid": "zDvZRwzkzNF8iJKdSXGJuoiC2pJ1sfrA9brunjhNTts3PrEQ92fs",
  "manifest": {
    "name": "dir1",
    "cids": [
      "zDvZRwzm8ZuLQB7kG3fcZqqNRPXbhe2d4du6pDC9MyoRRw12GKfS",
      "zDvZRwzmDd6Mg6E98nrXVkZ7THfK5zyQ6Y3j8fSSavSsAYReteLT"
    ]
  }
}
```

## Downloading

You can download separate files in the regular way, e.g. for `file11.txt`:

```bash
» curl "http://localhost:8001/api/codex/v1/data/zDvZRwzm8ZuLQB7kG3fcZqqNRPXbhe2d4du6pDC9MyoRRw12GKfS/network/stream" -o "file11.txt"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100     8  100     8    0     0   3188      0 --:--:-- --:--:-- --:--:--  4000
» cat file11.txt 
File 11
```

Now, let's download only `dir1`:

```bash
» curl "http://localhost:8001/api/codex/v1/dir/zDvZRwzkzNF8iJKdSXGJuoiC2pJ1sfrA9brunjhNTts3PrEQ92fs/network/stream" -o "dir1.tar"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  3711    0  3711    0     0  1401k      0 --:--:-- --:--:-- --:--:-- 1812k
```

We can check the contents of `dir1.tar`:

```bash
» tar -tf dir1.tar 
dir1
dir1/file11.txt
dir1/file12.txt
```

And we can successfully extract it to see that it has the same content:

```bash
» tar -xf dir1.tar
» tree ./dir1                                            
./dir1
├── file11.txt
└── file12.txt

1 directory, 2 files
» cat dir1/file11.txt 
File 11
» cat dir1/file12.txt
File 12
```

Finally, we can fetch the the whole previously uploaded `dir` directory:

```bash
» curl "http://localhost:8001/api/codex/v1/dir/zDvZRwzm597U7Bq9rDZ29KpqodiGHfAuLqMTXixujUFT3QYNDjgj/network/stream" -o "dir.tar" 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  6271    0  6271    0     0  1955k      0 --:--:-- --:--:-- --:--:-- 2041k
```

And also here, we can check its contents:

```bash
» tar -tf dir.tar    
dir
dir/file2.txt
dir/file1.txt
dir/dir1
dir/dir1/file11.txt
dir/dir1/file12.txt

» tar -xf dir.tar
» tree ./dir 
./dir
├── dir1
│   ├── file11.txt
│   └── file12.txt
├── file1.txt
└── file2.txt

» cat dir/file1.txt 
File 1
» cat dir/file2.txt
File 2
» cat dir/dir1/file11.txt 
File 11
» cat dir/dir1/file12.txt 
File 12
```

## Next steps

This is rough implementation. The most important, missing things are:

- while uploading tarballs, actually do streaming - now we are cheating a bit, so the current implementation will not work well on big file. On the other hand, while downloading directories, we are already doing some sort of a streaming while building the resulting tarballs by using unconstrained async queue.
- make sure we correctly record the permissions and modification timestamps. We have some basic stuff in place, but for the moment we hard-code it.

