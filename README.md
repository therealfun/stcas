# stcas - Simplest Content-Addressable Storage

stcas is more than a simple [content-addressable-storage](https://wikipedia.org/wiki/Content-addressable_storage)
set of tools to keep space-eficient backups using [data deduplication](https://en.wikipedia.org/wiki/Data_deduplication),
it is **the simplest** that I know.

The set has 3 tools:

- stcas - the main program
- stcas-backup - backup files/directories
- stcas-restore - restore backups
 
## Storage format

### Blob

The blob is just a piece of data and it is saved as
`blob/<SHA-TYPE>/<SHA:0-3>/<SHA:3-6>/<SHA>` files. You can change the
default SHA type and the directory schema by editing the `stcas` file.

To save a blob run:

```
echo "Simplest" | ./stcas write blob 
```

and you get the hash:

```
sha1:136e544257b24769118d92aca20209607a3da628
```

To read a blob run:

```
./stcas read blob sha1:136e544257b24769118d92aca20209607a3da628
```

### Objects

The objects are saved as `object/<SHA-TYPE>/<SHA:0-3>/<SHA:3-6>/<SHA>` files.

A file can be saved as a _data_ object:

```
type: data
part: sha
part: sha
...
```

Let's save the `./stcas-restore` program:

```
{
  echo "type: data"
  split -b 4096 --filter="./stcas write blob" < stcas-restore | sed -e 's/^/part: /'
} | ./stcas write object
```

The output will be something like:

```
sha1:4d3aea1117e6a20b23a2f614fe457dce380277bc
```

To read the object run:

```
./stcas read object sha1:4d3aea1117e6a20b23a2f614fe457dce380277bc
```

and you will get something like:

```
type: data
part: sha1:02d121957cb03991fe6960ea843ff93a33a9024c
part: sha1:6e1cab7e8e4abd51e6915045c704f3c21ca2616e
```

It is easier to use the `stcas-backup` program:

```
./stcas-backup -id test stcas-restore
```

The object will be saved as:

```
type: data
name: cmVzdG9yZQo=
size: 5047
mode: 755
mtime: 1526028222
owner: 100 user
group: 100 users
part: sha1:09e726f2a52af3eca80aa7325a8276ae00537766
part: sha1:7e158a0073723c69abd75dee49206a15692dc0f6
```

The name is saved in base64 format to avoid escaping special
characters. You can decode it with:

```
echo cmVzdG9yZQo= | base64 -d
```

### Tags

You can attach any number of tags to any object:

```
./stcas add-tag tag1 sha1:4d3aea1117e6a20b23a2f614fe457dce380277bc
./stcas add-tag tag2 sha1:4d3aea1117e6a20b23a2f614fe457dce380277bc
./stcas list-tags
```

### IDs

An ID is a name you give to an object (file/directory) to access it even
if it changes. For example, we save two copies of the `stcas-restore` program:


```
./stcas-backup -id x stcas-restore
echo >> stcas-restore
./stcas-backup -id x stcas-restore
```

We can list the current IDs:

```
./stcas list-ids
```

We can see the history of the `x` object, from the latest to the oldest version:

```
./stcas id-history x

1526027294 sha1:b93cdf79a245f001ccac4f069a3576ac6b0ca53b
1526021104 sha1:a8b1276239012784a1536b193b81d363dccb29cc
```

The first column is the time of _backup_ in UTC and you can _decode_
it with: `date -d @1526021104`.

You can have more than one ID pointing to an object:

```
./stcas set-id y $(TZ=UTC date +%s) sha1:a8b1276239012784a1536b193b81d363dccb29cc
./stcas id-history x

1526027294 sha1:b93cdf79a245f001ccac4f069a3576ac6b0ca53b
1526021104 sha1:a8b1276239012784a1536b193b81d363dccb29cc

./stcas id-history y

1526027491 sha1:a8b1276239012784a1536b193b81d363dccb29cc
```

## stcas-backup and stcas-restore

These two programs are examples of how to handle backups.

Example:

```
./stcas-backup -id test -tag this -tag that dir1 dir2 file1
./stcas-restore -id test /tmp/restore1
./stcas-restore -id test /tmp/restore2 "2 days ago"
./stcas-restore -sha sha1:a8b1276239012784a1536b193b81d363dccb29cc /tmp/restore3
```

The `./stcas-backup` program saves symlinks, directories and files. The files
are split in 4KB blobs. No file bigger than 12MB will be saved. You you
can change these limits by editing the `stcas-backup` file. You need to use
a different ID for any type of backup, otherwise it will hard finding
that backup. For example, you should use either:

```
./stcas-backup -id all ~/Documents ~/Mail
```

or

```
./stcas-backup -id doc  ~/Documents
./stcas-backup -id mail ~/Mail
```

You can restore objects (files/directories) by using the SHA of any
object or by using an ID. The later will allow you to restore content
from a specific date.

## Building stcas

You don't need to build stcas, at least not yet. Change it to suit your
needs and run it. You only need a couple of Unix programs which are
installed on almost all Linux distributions (ie. awk, bash, shasum ...),
but you don't need meson, proton, ninja or kung-fu build systems for
simple programs, let alone for the simplest ones.

## Improvement ideas

- variable sized blobs based on file size
- content-based slicing using a rolling hash (https://github.com/namelessjon/librabin)
- pick the SHA algoritm based on the blob size (eg. 0-4KB -> sha1, 4-64K sha224, >64KB sha384)
- compression for blobs (only after variable size blobs)
- search scripts
- mount the storage with libfuse
- indexers (add tags to objects based on content)
- clean-up scripts
- use rdup instead of stcas-backup/stcas-restore?
