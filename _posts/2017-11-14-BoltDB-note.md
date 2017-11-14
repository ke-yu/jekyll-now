---
layout: post
title: BoltDB note - 1 
---

## 1 Overview

BoltDB is a key-value store. It supports namespace, or `Bucket`. Each `Bucket` contains a collection of key-value pairs. The memory representation of database is a B+tree. Each `node` of tree has a list of `inode` which stores a key-value pair or a key if the node is a branch node. For serialization, the database is written to a series of `page`s. Each `Bucket` is written to one or more pages. The size of `page` is the same as the OS’s memory page size. When the database is initialized, the database file is `mmap`ed into memory in read-only mode. 

BoltDB uses copy-on-write to achieve the isolation of transaction. A `node` is deserialized from the corresponding page during write, so if there is no modification to a Bucket, Bucket's root node will be `nil`. For deserialized nodes, in spill process (`Bucket.spill()`), new pages will be allocated from the database for them and original pages will not be returned to the OS immediately but instead will be added to BoltDB's freelist.
![copy-on-write]({{ site.baseurl }}/images/copy_on_write.png)

Note only concurrent read/write transactions (`db.View()` for read and `db.Update()` for write) are supported. It maintains a single write lock. 

## 2 Data Structure

### 2.1 page

```golang
const (
    branchPageFlag   = 0x01
    leafPageFlag     = 0x02
    metaPageFlag     = 0x04
    freelistPageFlag = 0x10
)

type page struct {
    id       pgid     
    flags    uint16   // Page flag 
    count    uint16   // If page is a branch or a leaf page, count is the number of its inode (element)
                      //
                      // If page is a freelist page, count is the number of pgid
    overflow uint32   // If the size of data is over one page (e.g 4K), overflow records the number of extra pages. E.g, 
                      // the size is 12K, there are two extra pages, so overflow = 2
    ptr      uintptr  // overlap with the first page element
}
```

* meta pgage (`page.flags=metaPageFlag`)

  Note meta page is immediately after page header, `meta()` function converts the address of `page.ptr` to `*meta`. So, `page.ptr` = `meta.magic`. 

```golang
// There are two meta pages in the database
type meta struct {
    magic    uint32
    version  uint32
    pageSize uint32
    flags    uint32
    root     bucket
    freelist pgid
    pgid     pgid
    txid     txid
    checksum uint64
}

func (p *page) meta() *meta {
    return (*meta)(unsafe.Pointer(&p.ptr))
}
```

```
|<----------page header-------->|<----------------meta-------------------->|
+----+-------+-------+----------+-------+---------+----------+-------+-----+
| id | flags | count | overflow | magic | version | pageSize | flags | ... |
+----+-------+-------+----------+-------+---------+----------+-------+-----+
                                ^
                                | &page.ptr
```

* leaf page (`page.flags=leafPageFlag`)

  A leaf page represents a leaf node in B+tree, therefore it contains key-value pairs. Each `leafPageElement` is deserialized to an `inode` which contains a key-value pair.

```golang
// leafPageElement represents a node on a leaf page.
type leafPageElement struct {
    flags uint32
    pos   uint32 // memory position 
    ksize uint32
    vsize uint32 // value only exists on the leaf node
}
```

```
|<----------page header-------->|<------leafPageElement------>|     |<------leafPageElement------>|
+----+-------+-------+----------+-------+-----+-------+-------+-----+-------+-----+-------+-------+---+---+-----+---+---+
| id | flags | count | overflow | flags | pos | ksize | vsize | ... | flags | pos | ksize | vsize | k | v | ... | k | v |
+----+-------+-------+----------+-------+--+--+-------+-------+-----+-------+-----+-------+-------+---+---+-----+---+---+
```

* branch page (`page.flags=branchPageFlag`)

  A branch page represent a branch node in B+tree, therefore it only contains keys. Each `branchPageElement` is deserialized to an `inode` which only contains a key.

```golang
// branchPageElement represents a node on a branch page.
type branchPageElement struct {
    pos   uint32 // memory position
    ksize uint32
    pgid  pgid
}

// branchPageElement retrieves the branch node by index
func (p *page) branchPageElement(index uint16) *branchPageElement {
  return &((*[0x7FFFFFF]branchPageElement)(unsafe.Pointer(&p.ptr)))[index]
}
```

```
|<----------page header-------->|<-branchPageElement->|     |<-branchPageElement->|
+----+-------+-------+----------+-------+-----+-------+-----+-------+-----+-------+---+-----+---+
| id | flags | count | overflow | flags | pos | ksize | ... | flags | pos | ksize | k | ... | k |
+----+-------+-------+----------+-------+--+--+-------+-----+-------+-----+-------+---+-----+---+
```

* free page (`page.flags=freePageFlag`)

  There are two cases: if the number of free pages is over `64K` (note `page.count` is `uint16`), `page.count` will be set to a special value `0XFFFF` and the number of free pages is stored as the first element. 

```
|<----------page header-------->|
+----+-------+-------+----------+------+------+-----+------+
| id | flags | count | overflow | pgid | pgid | ... | pgid | 
+----+-------+-------+----------+------+------+-----+------+

|<----------page header--------->|
+----+-------+--------+----------+-------+------+-----+------+
| id | flags | 0xFFFF | overflow | count | pgid | ... | pgid | 
+----+-------+--------+----------+-------+------+-----+------+
```

### 2.2 freelist

```golang
// freelist represents a list of all pages that are available for allocation.
// It also tracks pages that have been freed but are still in use by open transactions.
type freelist struct {
    ids     []pgid          // all free and available free page ids.
    pending map[txid][]pgid // mapping of soon-to-be free page ids by tx.
    cache   map[pgid]bool   // lookup of all free and pending page ids.
}
```

The count of freelist is the count of all free pages + the count of pending free pages which are to-be-free pages from a write transaction.

The allocation of new pages will check `freelist.ids` and find out the contiguous `pgid`. If it fails, it will return `0` (note the first two pages are `meta` pages, so pgid will start from 2).

To free pages from a transaction, `free()` will free pages in `freelist.pending` map; when a page is freed, its overflow field will be expanded to contiguous pgids and add to `freelist.ids`. All freed pages’ `pgid` will be added to `freelist.cache`.
 
The initialization of freelist from a freelist page is done by `read()` which reads `page.count` (note if this value is `0xFFFF` then the value of the first ids is count) and then reads all ids to initialize `freelist.ids`. On the other way, `write()` writes the page ids onto a freelist page. All free and pending ids are saved to disk.

### 2.3 node

```golang
// node represents an in-memory, deserialized page.
type node struct {
	bucket     *Bucket
	isLeaf     bool
	unbalanced bool
	spilled    bool
	key        []byte
	pgid       pgid
	parent     *node
	children   nodes
	inodes     inodes
}

// inode represents an internal node inside of a node.
// It can be used to point to elements in a page or point
// to an element which hasn't been added to a page yet.
type inode struct {
	flags uint32
	pgid  pgid
	key   []byte
	value []byte
}
```

A page will be deserialized to a `node` if a Bucket is updated, and a tree will be built (so `node.children` and `node.parent` will be populated) by deserializing more pages to `node`s if there are writes to sub Buckets:

![build_tree]({{ site.baseurl }}/images/build_tree.png)

### 2.4 Bucket

```golang
// bucket represents the on-file representation of a bucket.
// This is stored as the "value" of a bucket key. If the bucket is small enough,
// then its root page can be stored inline in the "value", after the bucket
// header. In the case of inline buckets, the "root" will be 0.
type bucket struct {
	root     pgid   // page id of the bucket's root-level page
	sequence uint64 // monotonically incrementing, used by NextSequence()
}

// Bucket represents a collection of key/value pairs inside the database.
type Bucket struct {
	*bucket
	tx       *Tx                // the associated transaction
	buckets  map[string]*Bucket // subbucket cache
	page     *page              // inline page reference
	rootNode *node              // deserialized node for the root page.
	nodes    map[pgid]*node     // node cache

	// Sets the threshold for filling nodes when they split. By default,
	// the bucket will fill to 50% but it can be useful to increase this
	// amount if you know that your write workloads are mostly append-only.
	//
	// This is non-persisted across transactions so it must be set in every Tx.
	FillPercent float64
}
```

#### 2.4.1 Bucket creation

A `Bucket` could be created through `TX.CreateBucket()` or `Bucket.CreateBucket()` for nested bucket. `TX` has an anonymous root `Bucket`. 

```golang
func (b *Bucket) CreateBucket(key []byte) (*Bucket, error) {
	...

	// Move cursor to correct position.
	c := b.Cursor()
	k, _, flags := c.seek(key)

	...

	// Create empty, inline bucket.
	var bucket = Bucket{
		bucket:      &bucket{},
		rootNode:    &node{isLeaf: true},
		FillPercent: DefaultFillPercent,
	}
	var value = bucket.write()

	// Insert into node.
	key = cloneBytes(key)
	c.node().put(key, key, value, 0, bucketLeafFlag)

	...
	return b.Bucket(key), nil
}
```

`CreateBucket()` simply creates a `Bucket` and writes it to binary buffer (see below `Bucket.write()` and `node.write()`), then creates an `inode` under root Bucket's `rootNode` and saves `key` (which is new Bucket name) and `value` (which is the binary buffer of new Bucket) to this `inode`. Note the flag is `bucketLeafFlag` which means this `inode` is a sub-Bucket. For newly created `Bucket`, its `pgid` is always `0`. A Bucket whose `pgid` is not `0` and has `rootNode` is a dirty Bucket whose page(s) will be added to freelist during spill process.

```golang
	...
	c.node().put(key, key, value, 0, bucketLeafFlag)
	...
	return b.Bucket(key), nil
```

`b.Bucket(key)` gets sub bucket and put its root Bucket's `buckets` map. The following diagram shows creating a Bucket "MyBucket" under root Bucket. Note `pgid` of "MyBucket" Bucket is `0`, whilst `pgid` of `rootNode` of root Bucket is `3`.

![mybucket]({{ site.baseurl }}/images/first_bucket.png)

When a Bucket is created, it will be written to binary buffer and save this binary buffer as the `value` of `inode` in its parent Bucket's node by calling `node.put()`. `Bucket.write()` and `node.write()` show how to save a Bucket to a binary buffer:
  * Write bucket header (`root` and `sequence`).
  * Write its `rootNode` to a fake page.

```golang
// write allocates and writes a bucket to a byte slice.
func (b *Bucket) write() []byte {
    var n = b.rootNode
    var value = make([]byte, bucketHeaderSize+n.size())

    // Write a bucket header.
    var bucket = (*bucket)(unsafe.Pointer(&value[0]))
    *bucket = *b.bucket

    // Convert byte slice to a fake page and write the root node.
    var p = (*page)(unsafe.Pointer(&value[bucketHeaderSize]))
    n.write(p)  // see node.write() below

    return value
}

// write writes the items onto one or more pages.
func (n *node) write(p *page) {
    ...
    p.count = uint16(len(n.inodes))

    b := (*[maxAllocSize]byte)(unsafe.Pointer(&p.ptr))[n.pageElementSize()*len(n.inodes):]
    for i, item := range n.inodes {
        if n.isLeaf {
            elem := p.leafPageElement(uint16(i))
            elem.pos = ...
            elem.flags = item.flags
            elem.ksize = uint32(len(item.key))
            elem.vsize = uint32(len(item.value))
        } else {
            elem := p.branchPageElement(uint16(i))
            elem.pos = ...
            elem.ksize = uint32(len(item.key))
            elem.pgid = item.pgid
        }

        klen, vlen := len(item.key), len(item.value)
        if len(b) < klen+vlen {
            b = (*[maxAllocSize]byte)(unsafe.Pointer(&b[0]))[:]
        }

        // Write data for the element to the end of the page.
        ...
    }
}
```

#### 2.4.2 Open Bucket and get key-value

Open an existing Bucket will set Bucket's `page` field but won't deserialize page to node.

```golang
func (b *Bucket) openBucket(value []byte) *Bucket {
	var child = newBucket(b.tx)
	...
	if child.root == 0 {
		child.page = (*page)(unsafe.Pointer(&value[bucketHeaderSize]))
	}

	return &child
}
``` 

Therefore, fetching key and value of current element will be directly from page:

```golang
func (c *Cursor) keyValue() ([]byte, []byte, uint32) {
	...

	// Retrieve value from node.
	if ref.node != nil {
		inode := &ref.node.inodes[ref.index]
		return inode.key, inode.value, inode.flags
	}

	// Or retrieve value from page.
	elem := ref.page.leafPageElement(uint16(ref.index))
	return elem.key(), elem.value(), elem.flags
}
```

#### 2.4.3 Put key-value to Bucket

When some key-value pairs are added to "MyBucket" `Bucket`, a new `node` will be created as this Bucket's `rootNode` and `inode`s will be created to store key-value pairs, so all write operations are applied to memory model. Note the value of `inode` is not a binary buffer now, instead it stores string value. As its value is no longer is sub Bucket, `inode.flag` is `0` (instead of being `bucketLeafFlag`)

![key_value]({{ site.baseurl }}/images/insert_key_value.png)

#### 2.4.4 Serialize Bucket to page

The serialization of Bucket (store to pages) is done in `Bucket.spill()` when a writable transaction is committed. A Bucket could be inlineable if it doesn't contain sub-buckets (by checking `inode.flag`) and its total size is less than `pageSize / 4` (see `Bucket.inlineable()`). Inlineable Bucket will be directly stored to its parent's inode's value; otherwise parent's inode's value is a pointer. 
So if a sub Bucket is not inlineable, `value` is the pointer to sub Bucket's `bucket` field. 

```golang
// spill writes all the nodes for this bucket to dirty pages.
func (b *Bucket) spill() error {
	for name, child := range b.buckets {
		var value []byte
		if child.inlineable() {
			child.free()
			value = child.write()
		} else {
			child.spill()
			value = make([]byte, unsafe.Sizeof(bucket{}))
			var bucket = (*bucket)(unsafe.Pointer(&value[0]))
			*bucket = *child.bucket
		}
		...
		var c = b.Cursor()
		...
		// see below node.put(oldKey, newKey, value, pgid, flags)
		c.node().put([]byte(name), []byte(name), value, 0, bucketLeafFlag)
	}
	...
	// Spill nodes.
	if err := b.rootNode.spill(); err != nil {
		return err
	}
	...
}

func (n *node) put(oldKey, newKey, value []byte, pgid pgid, flags uint32) {
	...
	n.inodes = append(n.inodes, inode{})
	copy(n.inodes[index+1:], n.inodes[index:])

	inode := &n.inodes[index]
	inode.flags = flags
	inode.key = newKey
	inode.value = value
	inode.pgid = pgid
}
```

`spill()` will recursively call `spill()` on each sub Bucket, so each sub Bucket either write to dirty pages or write to a binary buffer which will be written to its parent's page if the Bucket is inlineable. After all its sub Buckets have finished `spill()`, its root node will be written to new pages in `node.spill()`.

```golang
func (n *node) spill() error {
	// sort and for each n.children[i].spill()
    ...
	
	// We no longer need the child list because it's only used for spill tracking.
	n.children = nil

	// Split nodes into appropriate sizes. The first node will always be n.
	var nodes = n.split(tx.db.pageSize)
	for _, node := range nodes {
		// Add node's page to the freelist if it's not new.
		if node.pgid > 0 {
			tx.db.freelist.free(tx.meta.txid, tx.page(node.pgid))
			node.pgid = 0
		}

		// Allocate contiguous space for the node.
		p, err := tx.allocate((node.size() / tx.db.pageSize) + 1)
		...

		// Write the node.
		node.pgid = p.id
		node.write(p)
		node.spilled = true
		...
	}
	...
}
```

Note in `spill()` process, a node might be splited. If a node's `pgid` is larger than `0`, it is a deserialized node from a page, then the original pages will be added to freelist and the transaction will allocate (`TX.allocate()`) new pages for this node. The serialized page layout is shown below:

![inline_bucket]({{ site.baseurl }}/images/spill.png) 

(To be continue)