Source: [PerKeep](https://perkeep.org).

See also [[Using PerKeep]]

Some eraly notes as they may be relavent in longer term to shape the support for small content, files, and dictionaries, but also for short term improvement of the Codex protocol and data maodel.

There is a nice [video](https://youtu.be/PlAU_da_U4s?si=GdwoV2CpblfjKwCj) about PerKeep, and here comes some relevant notes.


## Layered Architecture

![image](https://hackmd-prod-images.s3-ap-northeast-1.amazonaws.com/uploads/upload_46f8f356aa6cf7de87c214fff0437ecb.png?AWSAccessKeyId=AKIA3XSAAW6AWSKNINWO&Expires=1744197377&Signature=rQX20PUY3cezYbVbOG0MYxalq%2FA%3D)

In the end it is a blob store. So how they look at it, is also interesting to us.

## Blobs:

- 0 - 16MB,
- no file names,
- no mime-types,
- no metadata,
- no versions,
- just **immutable** blocks
 
Blobs are represented by *blob-refs*, self-describing identifiers similar to Multihases:

![image](https://hackmd-prod-images.s3-ap-northeast-1.amazonaws.com/uploads/upload_78ced2cfd6de59d72131668165ef8773.png?AWSAccessKeyId=AKIA3XSAAW6AWSKNINWO&Expires=1744197402&Signature=azFlehR79Vu36PGnIy7NVwBR6UQ%3D)

They use SHA-224. There are no deletes.

Blobs are not just flat bytes. Blobs can also be JSON objects with certain known fields, e.g. files are represented by JSON objects:

![image](https://hackmd.io/_uploads/S1kdVh_pJg.png)

Thus, blobstore keeps both data and metadata.

5TB video file or VM image? Merkle tree of "bytes" schema blobs. Data at leaves. Rolling checksum cut points (similar to rsync - to be checked how does it really work). De-duplication within files & shifting files. Efficient seeks/pread.

Files and Dictionaries:

![image](https://hackmd.io/_uploads/Hkwkr3dpkg.png)

### Indexing

The role of the metadata layer is *indexing* - to speed up the search. Most importantyl, it can be fully reconstructed from the BlobStore. As we will on some slides later, on top of indexing we have something called *Corpus* - optimized in memory key-value store to make the search even faster.

Below some slides about indexing:

![image](https://hackmd.io/_uploads/HJQzL2_61x.png)

![image](https://hackmd.io/_uploads/BkGE83OT1l.png)

![image](https://hackmd.io/_uploads/ryjP83uaJl.png)

![image](https://hackmd.io/_uploads/rJ-KU3Oakg.png)

![image](https://hackmd.io/_uploads/HyeiLhd6ke.png)

![image](https://hackmd.io/_uploads/B1NT8h_Tkg.png)


## Handling mutations

This is handled by something called *mermanodes* and needa further investigation.

![image](https://hackmd.io/_uploads/BJ-6PhOTkx.png)

A permanode and is a singed unique object. Having a permanode, you can then add atrributes (or mutations) to it:

![image](https://hackmd.io/_uploads/Bk2v_3ua1x.png)

> There seems to be an error in the above slide - the text should be *Better title* I guess.

![image](https://hackmd.io/_uploads/BkpEYh_pJg.png)

Everytime, you *put* an attribute on a permanode, you create a *mutation* or a *claim* connected to the base permanode. A claim seems to be a blob on its own.

Thus, in short, in PerKeep we seem to be having *mutations by appending*.

To be further investigated.
