# Rook Ceph

Rook is a management interface for Ceph that can deploy it in a Kubernetes
cluster. If you don't know what Ceph is, I highly recommend watching some of the
videos on it by the guys at 45drives.

- [45drives Ceph Series on YouTube](https://www.youtube.com/watch?v=i5PIFeWPpHM&list=PL0q6mglL88AP4ssmkDn_mMkozuQiLr2dv)

The elevator pitch: "It's like RAID, but it joins multiple servers instead of
just drives."

The more nuanced explanation:
- Imagine if you could take every piece of data you want to store and break it
  down into manageable byte-sized pieces.
- Now imagine that you could replicate each of these pieces of data a number of
  times to create redundancy, so if you lose one of them, you have replicas to
  pull from, and when all is functioning normally, you could run periodic checks
  to make sure that all of your replicas actually have the same data. If two
  replicas say the data is "a", but one says it's "b", then chances are that "b"
  was supposed to be an "a". This solves both availability and integrity
  problems.
- Now imagine that you could build an algorithm that takes into account the
  number of drives you have, the size of each one, the number of hosts you have,
  which drives were in which hosts, as well as your entire datacenter heirarchy,
  and use that to determine to which drive a piece of data should go to maintain
  a certain amount of redundancy across any failure domain of your choice.
- Now imagine that each drive had its own server that you could communicate with
  directly, and there was a server to ensure that not only are your redundancy
  requirements enforced when the data is created, but also as requirements and
  resource availability change.
- Now imagine that you could build interfaces on top of this data storage
  strategy, which exposes a filesystem, a block device, and an S3-compatible
  object store.

That's Ceph.

It's big, and it's complicated, but it is a true marvel of engineering that
solves an extremely complicated problem in a way that *any* application can
interface with and have no idea that anything has changed. One of my personal
biggest use cases for it is CephFS. CephFS is the only open source network
filesystem that I know of which is fully POSIX compliant, and it's built on a
rock-solid data redundancy system that has never fully failed on me except when
I got paranoid and forced it to do something stupid.

I highly recommend learning to manage Ceph properly, but what's even nicer about
all this is that unless you're doing anything crazy like me, you likely won't
have to do anything more than initiate automated repair for occasional data
corruption after heavy use. I'll get into some more Ceph management essentials
in a later section of the book, but Rook takes care of most of it for you.


## Installation

TODO
