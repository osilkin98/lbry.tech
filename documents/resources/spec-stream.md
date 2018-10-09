# Spec: Files, Streams, and Blobs

A `file` is a single piece of content that was published to the LBRY network. When a file is published, it is converted into a stream of blobs.

A `blob` is a chunk of data. Every blob has a `blob hash`, which is the sha384 hash of the blob. The maximum blob size is 2MB (2097152 bytes).

A `stream` is a collection of blobs that can be assembled into a file. The first blob in every stream is the `stream descriptor blob`
or SD blob. This blob contains the hashes of the rest of the blobs, the order of the blobs, and cryptographic information for decoding the
blobs into a file. 


More details coming soon. For an example implementation, see the code here: https://github.com/lbryio/lbry.go/tree/master/stream
