# Chapter 5. Hadoop I/O

## Data Integrity

The usual way of detecting corrupted data is by computing a `checksum` for the data when it first enters the system, and
again whenever it is transmitted across a channel that is unreliable and hence capable of corrupting the data. (checksum
just for error detection)

A commonly used error-detecting code is CRC-32. CRC-32 is used for checksumming in Hadoop’s `ChecksumFileSystem`, while
HDFS uses a more efficient variant called CRC-32C.

### Data Integrity in HDFS

HDFS transparently checksums all data written to it and by default verifies checksums when reading data.

Datanodes are responsible for verifying the data they receive before storing the data and its checksum. This applies to
data that they receive from clients and from other datanodes during replication. A client writing data sends it to a
pipeline of datanodes (as explained in **Chapter 3**), and the last datanode in the pipeline verifies the checksum. If
the datanode detects an error, the client receives a subclass of `IOException`.

When clients read data from datanodes, they verify checksums as well, comparing them with the ones stored at the
datanodes. Each datanode keeps a persistent log of checksum verifications, so it knows the last time each of its blocks
was verified. When a client successfully verifies a block, it tells the datanode, which updates its log.

> In addition to block verification on client reads, each datanode runs a DataBlockScan ner in a background thread that
> periodically verifies all the blocks stored on the datanode.

Because HDFS stores replicas of blocks, it can “heal” corrupted blocks by copying one of the good replicas to produce a
new, uncorrupted replica. The way this works is that if a client detects an error when reading a block, it reports the
bad block and the datanode it was trying to read from to the namenode before throwing a `ChecksumException`. The
namenode marks the block replica as corrupt so it doesn’t direct any more clients to it or try to copy this replica to
another datanode. It then schedules a copy of the block to be replicated on another datanode, so its replication factor
is back at the expected level. Once this has happened, the corrupt replica is deleted.

It is possible to disable verification of checksums by passing `false` to the setVerify `Checksum()` method on
`FileSystem` before using the `open()` method to read a file. The same effect is possible from the shell by using the
`-ignoreCrc` option with the `-get` or the equivalent `-copyToLocal` command.

> This feature is useful if you have a corrupt file that you want to inspect so you can decide what to do with it.

### LocalFileSystem

The Hadoop `LocalFileSystem` performs client-side checksumming. When you write a file called _filename_, the filesystem
client transparently creates a hidden file, _filename.crc_, in the same directory containing the checksums for each
chunk of the file. Checksums are verified when the file is read, and if an error is detected, `LocalFileSystem` throws
a `ChecksumException`.

Disable checksum

    Configuration conf = ...
    FileSystem fs = new RawLocalFileSystem();
    fs.initialize(null, conf);

### ChecksumFileSystem

`LocalFileSystem` uses `ChecksumFileSystem` to do its work, and this class makes it easy to add checksumming to other (
nonchecksummed) filesystems, as `ChecksumFileSystem` is just a wrapper around `FileSystem`.

    FileSystem rawFs = ...
    FileSystem checksummedFs = new ChecksumFileSystem(rawFs);

## Compression

File compression benefits:

- Reduce the space needed to store files
- Speed up data transfer across the network or to or from disk

A summary of compression formats

![compression format.png](compression%20format.png)

> The gzip is normally used, gzip file format is DEFLATE with extra headers and a footer

All compression algorithms exhibit a space/time trade-off: faster compression and decompression speeds usually come at
the expense of smaller space savings.

Create a compressed file _file.gz_ using the fastest compression method:

    gzip -1 file

> -1 mean optimize for speed, and -9 means optimize for space

The “Splittable” column indicates whether the compression format supports splitting (that is, whether you can seek to
any point in the stream and start reading from some point further on). Splittable compression formats are especially
suitable for MapReduce.

### Codecs

A _codec_ is the implementation of a compression-decompression algorithm. In Hadoop, a codec is represented by an
implementation of the `CompressionCodec` interface.

![hadoop compression codec.png](hadoop%20compression%20codec.png)

#### Compressing and decompressing streams with CompressionCodec

`CompressionCodec` has two methods that allow you to easily compress or decompress data.

To compress data being written to an output stream, use the `createOutputStream` (`OutputStream out`) method to create a
`CompressionOutputStream` to which you write your uncompressed data to have it written in compressed form to the
underlying stream.

Conversely, to decompress data being read from an input stream, call `createInputStream`(`InputStream in`) to obtain a
`CompressionInputStream`, which allows you to read uncompressed data from the underlying stream