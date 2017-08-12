---
layout: post
title: "Ceph BlueStore with bcache in Comparison to FileStore"
date: 2017-08-12
description: Notes on choosing a device layout for BlueStore and Linux Block Layer Caching
---

# Introduction

In this post I would like to explore Linux block layer caching (bcache) usage with Ceph.

There are many changes coming to Ceph Luminous including a change of a default and recommended rados object storage backend to BlueStore.

There is a great feature [summary presentation](http://events.linuxfoundation.org/sites/events/files/slides/20170323%20bluestore.pdf) by Sage Weil which discusses what's coming for Luminous - I highly advise to go through it.

# Issues with FileStore

FileStore with XFS as a default backend has several issues but one of the most pressing ones in my opinion is double-write for all data and metadata that's coming to an OSD. A journal has several modes of operation but it really comes down to a single mode because XFS is the [recommended](https://github.com/ceph/ceph/blame/jewel/doc/rados/configuration/filesystem-recommendations.rst#L35) backend for data storage with FileStore - this mode is called the [writeahead](https://github.com/ceph/ceph/blob/jewel/doc/rados/configuration/filestore-config-ref.rst#journal) mode. The [doc](https://github.com/ceph/ceph/blob/jewel/doc/rados/configuration/filestore-config-ref.rst#journal) is confusing as it says `Default: false` and at the same time `default for xfs` but the code is clear on that part: if there is [no explicit configuration](https://github.com/ceph/ceph/blob/jewel/src/os/filestore/FileStore.cc#L1562-L1564) and a backend [does not support checkpointing](https://github.com/ceph/ceph/blob/jewel/src/os/filestore/FileStore.cc#L1565) a [journal will be set](https://github.com/ceph/ceph/blob/jewel/src/os/filestore/FileStore.cc#L1566) to the writeahead mode. The XFS backend [does not](https://github.com/ceph/ceph/blob/jewel/src/os/filestore/XfsFileStoreBackend.h#L22) override the [default can_checkpoint](https://github.com/ceph/ceph/blob/jewel/src/os/filestore/GenericFileStoreBackend.h#L39) method, therefore, a writeahead journal is enabled by default.

A FileStore journal [may use](https://github.com/ceph/ceph/blob/jewel/src/os/filestore/FileJournal.h#L35) a raw block device or a file.

If a block device is used (which may be a partition) there will be:

* a double-write penalty for all data and metadata;
* metadata journaling ([logging](https://git.kernel.org/pub/scm/fs/xfs/xfsprogs-dev.git/tree/man/man5/xfs.5?h=v4.12.0&id=0602fbe880e59c3aff13a081632b79532a40a328#n46)) at the XFS level.

If a file is used there will be:

* a double-write penalty for all data and metadata;
* metadata journaling at the file system level for all data that goes into the journal file;
* metadata journaling at the XFS level.


Additionally, there are multiple layouts possible and one of them is collocating a journal with data on a single block device with two different partitions. Ceph documentation for Jewel [recommends](https://github.com/ceph/ceph/blame/jewel/doc/start/hardware-recommendations.rst#L105-L106) to use separate devices for a journal and data. And there is a very good reason for that which is illustrated by an example below.

In the output below, `/dev/sdb1` is a data partition with an XFS file system and `/dev/sdb2` is a journal partition which is a raw block device.

{% highlight bash %}
{% raw %}

$ sudo lsblk -f
NAME   FSTYPE LABEL UUID                                 MOUNTPOINT
sdb
├─sdb2
└─sdb1 xfs          022a7c91-cedf-4c05-8bfc-882eaf47f436 /var/lib/ceph/osd/ceph-6
sda
└─sda1 ext4   root  2b708114-354c-4ece-9d2e-3b7c202cbf17 /


{% endraw %}
{% endhighlight bash %}

If an osd receives some data, it will first be written to the journal partition and then to the data partition. "Some data" is a vague definition: Ceph operates with rados objects which [include](https://github.com/ceph/ceph/blame/jewel/doc/architecture.rst#L66-L70) an identifier, binary data, and metadata. Object storage semantics are [abstracted](https://github.com/ceph/ceph/blob/jewel/src/os/filestore/FileStore.h#L90) from a particular implementation via [ObjectStore](https://github.com/ceph/ceph/blob/jewel/src/os/ObjectStore.h#L225) and [JournalingObjectStore](https://github.com/ceph/ceph/blob/jewel/src/os/filestore/JournalingObjectStore.h#L23).

In other words, the data flow is as follows:

* write objects to `/dev/sdb2` (journal);
* read objects from `/dev/sdb2` (journal);
* write objects to `/dev/sdb1` (data partition).

Clearly, there is a **double write problem**. Two obvious recommendations for FileStore deployments can be inferred from that:

* do not store journal and data on the same block device in different partitions, otherwise you will severely cut available I/O bandwidth (at least [in half](https://www.sebastien-han.fr/blog/2014/02/17/ceph-io-patterns-the-bad/#I-1-3-Design-penalty));
* do not use hardware with the same speed for journals and data (e.g. do not do an all-flash deployment with identical SSDs for everything);
	* first of all, latency will be increased by at least a factor of two for a single OSD;
	* the more OSDs are added, the more partitions on a journal device will need to be created - having the same bandwidth available on the journal device as on individual data devices simply leads to an I/O bottleneck further increasing latency and removing benefits of parallel processing of transactions for different data devices by different OSD threads.


# FileStore with bcache

A journal may [speed up small writes by coalescing them and ensure consistency](https://github.com/ceph/ceph/blame/jewel/doc/rados/configuration/journal-ref.rst#L7-L26) but a journal is not a cache and a natural idea is to use a block layer cache. When it comes to using bcache it is very important to remember how FileStore works and avoid the double-write penalty.

With a single fast device in a bcache cache set and multiple slow devices there will be several `/dev/bcache<i>` block devices.

Partitioning of bcache devices is [possible as of](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b8c0d911ac5285e6be8967713271a51bdc5a936a) the version 4.10 of the upstream Linux kernel.

It will look as follows when set up:

{% highlight bash %}
{% raw %}

$ lsblk
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda 8:0 0 64G 0 disk
└─sda1 8:1 0 64G 0 part /
sdb 8:16 0 64G 0 disk
└─bcache0 251:0 0 64G 0 disk
  ├─bcache0p1 251:1 0 4G 0 part
  └─bcache0p2 251:2 0 4G 0 part
sdc 8:32 0 64G 0 disk
└─bcache0 251:0 0 64G 0 disk
  ├─bcache0p1 251:1 0 4G 0 part
  └─bcache0p2 251:2 0 4G 0 part

{% endraw %}
{% endhighlight bash %}

bcache has the following modes of operation: writethrough, writeback, writearound and none

For the following two examples I assume that the writeback mode is used:

* collocating data with a journal on a bcache device will result in the double-write issue combined with cache pollution at the bcache level;
* having 3 types of devices with different speeds (a very fast device, a fast device and a slow device) and using the fastest one for a journal is a better solution in terms of performance as it has a benefit of coalescing small writes without causing cache pollution as above.

Using write-around bcache mode is (probably) the least intrusive, especially if you just want to improve read workloads: write to the slowest device directly from a fast journal but cache frequently read data via bcache. This will only improve read performance and will not make things worse for writes than they already are with FileStore.

So, in my view, bcache can result in some performance improvement with FileStore when 3 types of storage devices with different speed are present, e.g.:

* several SATA or SAS HHDs for data;
* one or multiple medium-speed SSDs for bcache;
* one or multiple small-size but high-bandwidth and low-latency NVMe devices for journaling.

Note that bcache as of Linux kernel 4.12 [does not support](http://elixir.free-electrons.com/linux/v4.12.6/source/Documentation/bcache.txt#L80) multiple cache devices per cache set but with multiple cache sets it should be possible to use multiple SSDs for bcache purposes. As for journal devices, there will be a partition on a journal device per an OSD so there is nothing wrong with using multiple journal devices (say, 8 data devices, 2 journal devices, 4 partitions per journal device).

# BlueStore with bcache

BlueStore eliminates the double-write problem and metadata journaling overhead at the file system level - it uses an underlying block device directly and does not journal data. A few throughput and IOPS graphs are available in the [Sage's presentation](http://events.linuxfoundation.org/sites/events/files/slides/20170323%20bluestore.pdf) starting with page 29 - some test results show almost doubled throughput in comparison to FileStore. In essence, it provides a [metadata-only journal](http://tracker.ceph.com/projects/ceph/wiki/Rados_-_metadata-only_journal_mode/16) called WAL (Write-Ahead Log) and its own on-disk data layout that does not use an existing file system implementation like XFS.

WAL abbreviation is there due to [RocksDB](https://github.com/facebook/rocksdb/wiki/Write-Ahead-Log-File-Format) usage. RocksDB's persistent data is [stored](https://github.com/facebook/rocksdb/wiki/A-Tutorial-of-RocksDB-SST-formats) in Static Sorted Tables (SSTs) which can be classified into cold and warm SSTs based on usage and stored on different devices.

Caching metadata stored in SSTs does not require large devices which opens up several possibilities when it comes to using bcache (writeback) with BlueStore.

* use bcache directly (2 types of devices): one or multiple fast devices for cache sets and several slow devices as backing devices for bcache block devices;
* 2 types of devices: store WAL on a very fast (but small) device and store SSTs with object data on bcache block devices;
* 2 types of devices: store WAL and warm SSTs on a fast device and cold SSTs and data on bcache devices;
* 3 types of devices: WAL on a very fast device, warm SSTs on a slower device, cold SSTs and data on bcache devices.

In the latter case two devices with the same speed but different size can be used for warm SSTs and a bcache cache set.
