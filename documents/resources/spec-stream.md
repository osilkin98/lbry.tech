# Spec: Files, Streams, and Blobs

A `file` is a single piece of content that was published to the LBRY network. When a file is published, it is converted into a stream of blobs.

A `blob` is a chunk of data. Every blob has a `blob hash`, which is the SHA384 hash of the blob. The maximum blob size is 2MB (2097152 bytes).

A `stream` is a collection of blobs that can be assembled into a file. The first blob in every stream is the `stream descriptor blob` or SD blob. The rest are `content blobs`. The SD blob contains the hashes of the content blobs, their order in the stream, and cryptographic information for decrypting them. 


## SD Blob Structure

```
{
   "stream_type":"lbryfile",
   "key":"94d89c0493c576057ac5f32eb0871180",
   "suggested_file_name":"6b706a7977755477704d632e6d7034",
   "stream_hash":"8cef6280f36f7e6590a6218da6b2eb8184ab1435c3f8d77f008088f5d2bc6bd2252a2beb9cfa3d9d40b9ce36d2d7b2ce"
   "stream_name":"6b706a7977755477704d632e6d7034",
   "blobs":[
      {
         "length":2097152,
         "blob_num":0,
         "blob_hash":"a6daea71be2bb89fab29a2a10face08143411a5245edcaa5efff48c2e459e7ec01ad20edfde6da43a932aca45b2cec61",
         "iv":"ef6caef207a207ca5b14c0282d25ce21"
      },
      {
         "length":2097152,
         "blob_num":1,
         "blob_hash":"bf2717e2c445052366d35bcd58edb108cbe947af122d8f76b4856db577aeeaa2def5b57dbb80f7b1531296bd3e0256fc",
         "iv":"a37b291a37337fc1ff90ae655c244c1d"
      },
      ...,
      {
         "length":0,
         "blob_num":45,
         "iv":"53677e463ddb3bf060a40b99f8236432"
      }
   ]
}
```

The `key` field is optional. The key may be stored externally on a keyserver. The keyserver would make the key available when presented with proof that the content was purchased.

TODO: protocol should include details about running a keyserver.

More details coming soon. For an example implementation, see the code here: https://github.com/lbryio/lbry.go/tree/master/stream
