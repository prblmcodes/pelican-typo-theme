Title: The second article
Date: 2012-12-01 10:02
Slug: article-2
Author: Mike
Category: Tech
tags: Miky

I studied LSM trees at university and after encountering them twice in Designing Data-Intensive Applications and Database Internals I decided to implement something in Java.

The idea behind this project is not to provide the most efficient implementation ever, but to experiment with storing data on disk, any suggestions are welcome!

Here’s the Github repo if you want to have a look, this article is also published on Medium.

Introduction 
An LSM tree is a structure used by NoSQL databases, such as Cassandra, RocksDB, LevelDB, Dynamo, and so on. It’s suitable for write-intensive applications.

We can distinguish two key components of the tree, the in-memory buffer, also called Memtable, and the disk-resident tables. The main idea is to accept writes to the in-memory part of the tree, and to flush them periodically, or when a certain size is met.

A key aspect of this structure is ordering, indeed, keys are sorted both in RAM and on disk, enabling logarithmic searches.

For the sake of this project elements in the tree are simple key-value pairs.

Memtable 
Having sorted elements in memory is not a new problem we can exploit any efficient order-preserving data structure for this part, such as Red-Black or AVL trees.

In this particular implementation, I decided to build a Skip List, which provides the same theoretical complexity in the average case of balanced trees, but is straightforward to implement. 1

A Skip List is a multi-leveled linked list. The idea is to have fast lanes between nodes, and, by carefully constructing them, we can reduce the number of links we need to traverse while searching.

skiplist
The list properties are:

elements at level zero are sorted;
the number of levels are 
log
⁡
(
n
)
log(n), where 
n
n is the size of the list;
if a node is at level 
i
i, then is must also be at level 
i
−
1
i−1.
Searching 
Given the above properties, searching is done as follows:

start at the highest level and traverse until the node key is less than the wanted key;
if the successor surpasses the wanted key, go down a level and repeat, else we found the element. Eventually, we’ll reach level zero, and determine if the element is found or not.
Inserting 
Insertion proceeds as follows:

locate the insert position with the same logic as before;
determine a level for the new element;
insert as in a linked list, but at each required level. Note that for this to work we need to keep track of a predecessor buffer while descending levels. This way we can correctly replace successors pointers at each level.
public void add(ByteArrayPair item) {

    // Locate the element keeping track of predecessors at each level
    Node current = sentinel;
    for (int i = levels - 1; i >= 0; i--) {
        while (current.next[i] != null && current.next[i].val.compareTo(item) < 0)
            current = current.next[i];
        buffer[i] = current; 
    }

    // Replace current value if possible
    if (current.next[0] != null && current.next[0].val.compareTo(item) == 0) {
        current.next[0].val = item;
        return;
    }

    // Insert new node at a random level, updating predecessors
    Node newNode = new Node(item, levels);
    for (int i = 0; i < randomLevel(); i++) {
        newNode.next[i] = buffer[i].next[i];
        buffer[i].next[i] = newNode;
    }
}
Choosing a level 
To determine a level, we can toss a coin and keep going until we get heads. This would require a lot of random generations, a faster way is to generate a single number and use its binary representation as boolean values.

private int randomLevel() {
    int level = 1;
    long n = rn.nextLong();
    while (level < levels && (n & (1L << level)) != 0)
        level++;
    return level;
}
SSTable 
A Sorted String Table is a disk-based structure for sorted immutable data. They consist of two main files, one with actual data and another with an index to speed up look-ups.

Indexing and Look-Ups 
Given the data file, searching for a key can be implemented with a full scan. This is tremendously slow on big files, hence we rely on indexing to skip portions of data.

Given a sampling factor 
k
k, we build a sparse index with keys at position 
0
,
k
,
2
k
0,k,2k, and so on. By storing the index in an array we can rely on binary search to find a given offset in the data file, where we can start a linear scan. This permits us to skip a lot of unnecessary comparisons and locate a file portion that likely stores our value.

sstable
Note that we can stop the search as soon as the current element surpasses the wanted one. Below is the code for searching, this implementation is as lazy as possible, meaning that we only read what’s strictly necessary while iterating on the input stream.

public byte[] get(byte[] key) {

    if (compare(key, minKey) == -1 || compare(key, maxKey) == 1)
        return null;

    // binary search an offset to start search
    long offset = getCandidateOffsetIndex(key);
    int remaining = size - sparseSizeCount.getInt(offsetIndex);
    
    // move input stream to the offset given by the index
    is.seek(offset);

    int cmp = 1;
    int searchKeyLen = key.length, readKeyLen, readValueLen;

    byte[] readKey;
    while (cmp > 0 && remaining > 0) {

        remaining--;
        readKeyLen = is.readVByteInt();

        // gone too far
        if (readKeyLen > searchKeyLen) {
            return null;
        }

        // gone too short
        if (readKeyLen < searchKeyLen) {
            readValueLen = is.readVByteInt();
            is.skip(readKeyLen + readValueLen);
            continue;
        }

        // read full key, compare, if equal read value
        readValueLen = is.readVByteInt();
        readKey = is.readNBytes(readKeyLen);
        cmp = compare(key, readKey);

        if (cmp == 0) {
            return is.readNBytes(readValueLen);
        } else {
            is.skip(readValueLen);
        }
    }

    return null;
}
Bloom Filters 
What happens when we search for a key that’s not on disk? We waste a lot of precious CPU cycles on binary searching and seeking on an offset, and iterating until we surpass the wanted key.

To avoid unnecessary operations we can rely on a compact and probabilistic structure such as Bloom Filters. The idea is to have a structure that answers membership queries, having some false positive answers, but no false negatives. We can tune the structure for our particular needs, by specifying a false-positive rate.

So, while looking for a key, we first test for probabilistic membership, and if the answer is negative, we can early return null from the search.

public byte[] get(byte[] key) {
    if (!bloomFilter.mightContain(key))
        return null;
    ...
}
Data layout 
Data is disk-resident, hence we need to define a binary format to follow, with minimal overhead. An SSTable is made of 
n
n elements, where each one of them has a variable length key and value.

Key and value pairs are byte arrays, and to lay them out on disk we encode their length 
l
l, followed by 
l
l bytes. Each integer is written in variable byte encoding, to not waste 32 bits on small numbers. 2

This encoding uses a byte to store a continuation bit and a 7-bit payload containing part of the represented number. For instance, consider the number 
456
456 and its binary representation 
111001000
111001000, the variable byte encoded version is:

∣
1
∣
0000011
∣
0
∣
1001000
∣
∣1∣0000011∣0∣1001000∣

The first byte begins with one, hence that the number is not finished, while the second block starts with zero, indicating no more bytes are needed to decode the current integer.

The index file is a list of keys, so we can use the same length plus payload encoding for it. Each index entry has an offset related to it, specifying the number of bytes to skip in the file to reach it. Those offsets are increasing, so we can use something like delta-encoding to store them. Below is an example of such encoding:
0
,
25
,
76
,
…
0,25,76,…
↓
↓
0
,
(
25
−
0
)
,
(
76
−
25
)
,
…
0,(25−0),(76−25),…
Resulting integers are smaller, thus encoded in fewer bytes.

Finally, Bloom Filters are represented by some hyperparameters and a bit vector, that we can encode as is.

Putting it all together 
We saw how to construct an SStable and how a Skip List works in memory, It is time to combine the two to obtain the final engine.

The main components of the tree are:

in-memory mutable buffer or Memtable: a skip list as presented in the previous paragraphs, with a max size;
in-memory immutable buffers: a list of skip lists containing memtables that need to be flushed to disk;
disk-resident tables: a collection of SSTables obtained from memtables flushing, they are divided into levels, level zero containing the most recent data.
tree
We are going to first see how primitives are defined, and then give an overview of how the tree is maintained, with buffer flushing and table compaction.

Insertion and Search 
To insert a new element we simply add it to the in-memory buffer. If the list does not exceed the maximum size we are done, otherwise the current list is scheduled for disk flushing, and the mutable buffer is re-initialized.

Searching is a bit trickier, and has at most three steps:

query the mutable buffer;
query all the immutable buffers scheduled for flushing;
query all the disk tables starting from level zero and on.
If, at any point, the wanted key is found, we can stop the search.