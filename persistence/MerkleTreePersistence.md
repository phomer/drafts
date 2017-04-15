# Tendermint Persistence

## Overview

Merkleeyes is a **Tendermint** ABCi app that provides *Merkle* proofs for keyed values. It supports both in-memory and persistent storage. Apps such as Basecoin can use it to hold their application state. 

This post will explore the underlying data-structures that drive the code and our extensions to it that now support historic queries.

## Keys and Values

An app can send pairs of keys and values as a series of Sets and Removes. These are embedded within the underlying ABCi protocol (specifically CheckTx and DeliverTx). Once a series of these changes are done, the app can choose to permanently Save these changes by calling Commit.

These key-values are initially held in memory, where we build a binary+ tree on top of them. That may seem simple, but it needs a bit of explanation. 

## Binary+ Tree

A standard binary tree is a data-structure where we create a 'node' that has pointers to two or less, children. A bunch of these nodes forms a tree-like structure. Traditionally we visualized these  binary trees as upside down with the root on top and the branches  stretching out downwards.

If a node has children, we say that it is an internal node. If it doesn't, it sits at the very bottom of the tree, so we call it a leaf node.

So a binary+ tree is just a binary tree, but with all of the data stored in the leaf nodes. The internal nodes are only used to hold copies of the keys, and are there to effectively support doing a binary search on the underlying set of leaf nodes.

As we build up the tree, we add in new leaf nodes so that they are ordered lexically, which is a fancier way of saying mostly alphabetical. To insure that the existing leaf nodes always remain at the bottom of the tree, each new insert requires us to add an internal node as well. This means that if there are, say, 10 leaf nodes, then there should be 20 nodes in the tree. Opps, 19, because the very first node doesn't actually need a corresponding internal node.

## Performance Considerations

One problem with an ordered binary+ trees is that if all the nodes come into the tree in ascending order, we'll keep adding them only to the right side, which will cause the tree to degenerate into a linked list with no left-hand children. If we were to search for the last node we would have to visit all N nodes.

To avoid this and for better performance, we automatically balance the tree, the same as is done for an AVL Tree. This keeps the tree from becoming too lop-sided.  This is less expensive than a fully weight balanced tree, but the trade-off is faster insert and deletion times. AVL
trees insure that inserts and deletes are no worse than Log N, or to put a little more causally, these operations need only follow a specific 'path' down the tree to accomplish their goal; they don't have to visit every node that exists. That makes a big difference once the tree has become huge.

The app can do any number of Sets and Removes on these keys. Technically a Set can add a new value or it can replace an existing one, but for that second case we choose to ignore it and add in a new node anyways. So essentially this makes the binary+ tree immutable. The data in the node that composes the binary tree gets written once and is locked into place. However, this data is only a part of the what makes up a node, as we shall soon see.

## Merkle Trees

Once a Save is issued, the binary tree is turned into a Merkle tree. That is, each internal node hashes its underlying children and the leaf nodes hash the key and the contents.

Hashing against a large enough space really acts like a 'summary' of any underlying data. Some information is removed from that data, so it is not reversible, but what remains is manipulated and reordered in a way that mostly insures that the summary is unique enough. A Merkle proof is a way of leveraging this hashing to prove membership of an individual item within a larger set. This proof consists of all of the hashes at each level in the tree that can be combined and recalculated, then matched against the root hash for the overall set. 

In this way, if we get an underlying piece and the hashes on the outside of the path (aunts or uncles), we should be able to recompute the root hash. If we can't, there is something wrong with the data.

This assures people that there has been no tampering with any of these parts, even if their origins are unknown.

## Similarities

It happens to be really convenient that a Merkle tree and a binary+ tree are also exactly the same size and have very similar structures. One key difference is that the order of the leaf nodes in a Merkle tree is left unspecified. So having the binary+ leaf nodes lexically ordered is fine.

This Merkle tree can then be persisted into a key/value data store. We can't however, just take the app's keys and values and write them out, since we have a whole lot of internal nodes that need to be saved as well. Another convenience is that the hashes we calculated for the Merkle tree just also happen to be unique relative to the set of all nodes. That is, each hash is a summary of the underlying children for internal nodes, or the key and the value for leaf ones. This guarantees that we are not going to collide. So we can use these hashes as the keys for persistence, and encode the rest of the node data into the value.

As an almost free optimization, since we might need to access the same persistent nodes many times within a short span of time, it would be best to cache them in memory if they have been used recently. This LRU cache is again keyed on the hashes, even through it contains a pointer to the actual node in memory.

## References and Immutability

Now this then gives us a whole bunch of different ways to refer to any node. We identify the leaf nodes by their unique keys, but they also can be accessed by their hash. The internal nodes can be identified by their hashes, but we still have the original pointers of the binary+ tree in their as well. The keys for the internal nodes aren't unique, but they do act as guide posts for finding out where the lexically ordered leaf nodes ended up.

Since we almost got to immutability when the data was in binary+ form, it seems like we can get there with the Merkle tree. We just make sure we compute the hashes. We nearly achieve this, but again there is a wrinkle. The nodes in memory have pointers to their children, but we
can just as easily refer to them by their children's hashes. It becomes necessary for us to delete these pointers sometimes in order to deal with cleaning up the tree in between changes. If for example, we cache one node because it is in the current Set path, and it still has pointers to both underlying children, then at least one of those children is not in the cache and should have been garbage collected, but can't be.

## A Funny Thing Happened on the Way to the Forum

When we go to add in a new leaf node to an existing tree something interesting happens. If we take a simple example of adding in a new leaf node on the far right, we will have to create a node for the leaf and an new internal node to hold its neighbor. As well, every parent going upwards through the internal nodes will have changed now, all the way up to the root, but we really wanted them to be immutable. To accomplish this, we end up recreating all Log N nodes in the path, pointing them to the new right side children, but also to the existing left children too.

This modification has a curious side-effect. If you start at the new root, you have a fully complete binary tree that includes the new node. However if you start at the old root, you still have the original, immutable, binary tree that is unchanged. Both trees exist, happily intertwined with each other.

## Orphans

For convenience we refer to the nodes of the older version of the tree as being 'orphaned', even though they are still valid in that tree.

During a series of Sets and Removes, there are many such paths created, and many nodes that are getting orphaned. If we remove every link to the roots, all of the nodes they will eventually be garbage collected. In that sense, if there are no persistence nodes then simply unlinking the roots would insure that memory doesn't fill up with all of these old nodes.

But the situation gets a little more complex when we add in persistence. Once the nodes have been Saved into the Merkle Tree we want them to be immutable and stored in the database. If we are traversing through a part of the tree that hasn't been accessed for a while, we need to go back to the database to reload the data.

It's worth noting that not all nodes will or need to persist. Any internal node for a path that intersects with another path will never be persisted, but eventually it will be garbage collected. We can see this quite easily with the roots in between Saves, but it is also true with some of the lower internal nodes or leaf nodes that have been created then removed before the Save. 

To keep track of this, we set a boolean, 'persisted',  whenever we load or store one of the nodes. For any given path, we can easily tell what has been persisted and what will eventually be garbage collected.

What this means is that there are always two types of orphans. Those that are temporarily in memory, and those that need to be clean up out of the database.

## Getting to History

Now on top of this, we'd like to store history. We want to keep any tree that have been Saved, but we don't need the intermediate trees.

We need this because lite-clients will not necessarily be able to get an independent block hash and a proof at the same time. If they are going to two difference sources, there is an inherent race condition that needs to be address. To solve this we require proofs that are somewhat earlier in history.
 
The rather obvious extension is to keep track of the many persistent roots in the database. We want a fixed number of them, say 1000, since we need to be able to constraint the disk usage. 

If we persist any given root, with its new set of paths, then all we really need to do is keep track of this, and at a later point delete all of the persisted nodes. The root hash in this case acts as a unique key, to which we can reach all of the underlying node changes.

While just keeping all of the roots is actually enough to work out the differences in nodes, it would be excessively time consuming, so instead in between saves we just build up a list of any persistent orphans, while ignoring the ones that will be garbage collected.

When we have too many old persistent roots, all we need to do is get any orphan list associated with them from the database and then delete all of the nodes.

## Finality

So the summary of all of this is fairly straightforward. We take the app's key-value pairs, build a binary+ tree on them as a series of paths. Once a Save is issued, we overlay a Merkle Tree on top, push any current persistent orphans into the database and then save the tree values, keyed by the node hashes. We keep this final root, along with the older versions and whenever we have too many we just prune the old roots and their orphan lists out of the database. Again, for convenience we associate the tree version numbers with the block height, specifically because the block Commits are what cause the tree Saves.

The last little wrinkle is that we have to make sure that across a bunch of Saves, each node is still unique. That is, for one version a leaf node may exist, then deleted for another, then return again. This really isn't a problem because we keep that explicit version number. All we need for this is to add that version into the data that is being hashed at the leaf nodes, insuring that the keys will always be different.


