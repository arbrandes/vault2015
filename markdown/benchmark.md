Ceph cluster
# Benchmarks
Note:
- When benchmarking Ceph (and this is where it gets challenging), you need to
  take into account all these dimensions, use cases, and layers.


<!-- .slide: data-background="images/ceph_benchmarks_bottom_to_top.svg" data-background-size="auto 95%" -->
Note: 
- Overview: What can influence your Ceph performance, at what level.
- Lowest: block devices that store data.  Your OSD can never get any faster
  than your underlying block device. Typically, a raw block device for the
  journal, and the OSD file store on a filesystem.
- Journal performance: As a cost/performance tradeoff, you'll see relatively
  fast, but small, journal devices, versus slower but larger file stores.  Per
  server, you'll see a maximum of 12-18 spinners as file stores, with a smaller
  number of SSDs (2-3) to host those journals.  Journal performance = streaming
  performance of device.
- File store performance: Filesystems influence Ceph performance: 3 filesystems
  to choose from: BTRFS, XFS, ext4 (most people will use XFS, as it tends to
  outperform ext4, and BTRFS is still not production-ready).  All have
  tunables.
- OSD daemon performance: it uses the file store and journal, but also a fair
  amount of CPU and memory.  There are also many tunables, which are critical
  to performance.
- Network performance: client (or public) network, and the cluster network (for
  replication and backfilling).
- Top layer: what uses librbd, such as RBD itself, QEMU with RBD driver, or
  CephFS, or, if going through radosgw, Swift or S3 clients.
- Luckily, good tools to evaluate the system, bottom to top.


## Block device benchmarks
(simple)
```sh
dd if=/dev/zero of=/dev/sdh1 bs=1G count=1 oflag=direct

dd if=/dev/zero of=/dev/sdh1 bs=512 count=1000 oflag=direct
```
Note:
- One of the things you should do is benchmark performance of the underlying
  block devices BEFORE actually deploying the cluster, as some benchmarks are
  destructive (such as this really simple one).
- This writes some data to a block device.  The top one is a simple
  micro-benchmark for throughput: writes a full gig of data in o_direct mode
  (oflag=sync, oflag=dsync).  The flag is necessary, otherwise you're just
  measuring your page cache in RAM.
- The bottom one is a super-simple one for latency: smallest amount of data you
  can write.  If a spinner, typically 512 bytes; SSDs, 4K.
- If the block device has a cache, make sure to overwhelm it with the amount of
  data you're writing.


<iframe src="https://asciinema.org/api/asciicasts/13104?size=medium&amp;theme=solarized-light&amp;speed=2" id="asciicast-iframe-13104" name="asciicast-iframe-13104" scrolling="yes"></iframe>
Note:
- Here, on a live Ceph cluster, on the OSD directory, I'm writing 1G to the
  journal device directly (a symlink to a partition, as you can see), in direct
  mode.  2GB/s is a little high, though.
- Next, we test SSD latency: 0.03s/1000 = 0.03milliseconds.
- Next, we write directly to the spinner, but we need to overwhelm the cache
  (with 2GB) to get the real value.


## Block device benchmarks
(advanced)
```sh
fio --size=100m \
	--ioengine=libaio \
	--invalidate=1 \
	--direct=1 \
	--numjobs=10 \
	--rw=write \
	--name=fiojob \
	--blocksize_range=4K-512k \
	--iodepth=1
```
Note:
- You can do the same benchmark in a more elaborate fashion using FIO.
- Universal benchmark tool, maintained by Jens Axboe, who also maintains the
  Linux kernel's block layer.
- In this case, we're trying to duplicate the typical write load for a Ceph OSD
  journal: async IO (libaio engine), direct IO, random block size, writing
  100MB 10 times (1GB)


<iframe src="https://asciinema.org/api/asciicasts/13105?size=medium&amp;theme=solarized-light&amp;speed=2" id="asciicast-iframe-13105" name="asciicast-iframe-13105" scrolling="yes"></iframe>
Note:
- We get 6.3GB/s aggregate bandwidth, which is cool.
- You can play around with the parameters as I'm doing here.  Usually, a
  spinner will give you 15-80 MB/s of bandwidth, while an SSD will get you
  something in the 300-400 MB/s range.
- Don't expect Ceph performance to match journal performance, though!  OSDs
  will periodically flush journals to disk (of course), and when that happens,
  you're constrained by spinner performance.


## Network benchmarks
```sh
netperf -t TCP_STREAM -H <host>
```
Note:
- I don't have a screencast for this, as most of you are probably familiar with
  netperf.
- Ceph connections use regular TCP sockets, so you could also dd into netcat,
  for instance.


## OSD benchmark
```sh
ceph tell osd.X bench
```
Note:
- At this point we have benchmarks for block and network level, and now we can
  actually benchmark the OSD.
- This is really useful because it comes bundled with Ceph.


<iframe src="https://asciinema.org/api/asciicasts/13106?size=medium&amp;theme=solarized-light&amp;speed=2" id="asciicast-iframe-13106" name="asciicast-iframe-13106" scrolling="yes"></iframe>
Note:
- What comes out is something like this.
- I always run benchmarks, as you've seen, at least 3 times, and take an
  average.  This is because you can run into hidden caches, or something to
  that effect.
- By the way, this is a non-destructive benchmark, so you can use it on a
  running cluster (though it might impact THAT OSDs performance temporarily).
- (If you're feeling naughty, run ceph tell osd.asterisk on a cluster you
  don't maintain. ;)
- This runs locally on the OSD.
- Useful for catching configuration errors, such as mounting XFS without
  inode64, which will impact performance.


## rados benchmark
```sh
rados bench -p bench 30 write
```
Do this on a throwaway pool!
Note:
- This DOES take the network into account.  You tell rados to test writing or
  reading, and for how long, from an actual client (such as an OpenStack
  compute node.)
- This will create a bunch of RADOS objects, and while you can use any pool for
  this, it is recommended to create a throwaway one.  Though the benchmark will
  removes objects it creates, if you interrupt it with SIGINT, they'll remain.


<iframe src="https://asciinema.org/api/asciicasts/13107?size=medium&amp;theme=solarized-light&amp;speed=2" id="asciicast-iframe-13107" name="asciicast-iframe-13107" scrolling="yes"></iframe>
Note:
- This is what it looks like.
- Another reason to use a separate pool is that you can play with the number of
  PGS and how they affect performance.
- You can see the expected performance on a 10Gigabit network, which is about
  1.1GB/s.
- You can tune it to the appropriate write size, to get a better latency test.
  If using RADOS, for instance, you'll probably want to set the writes to 4MB
  chunks.


## fio RBD benchmarks
```sh
fio --size=10G \
	--ioengine=librbd \
	--invalidate=0 \
	--direct=1 \
	--numjobs=10 \
	--rw=write \
	--name=fiojob \
	--blocksize_range=4K-512k \
	--iodepth=1 \
	--pool=bench \
	--rbdname=fio-test
```
Note:
- Now we can move up the stack and benchmark one of the popular Ceph use cases,
  which is volume storage for virtualization.  Luckily, in recent versions of
  fio there is a librbd backend engine, so instead of having to jump through
  mounting hoops.
- You can see standard options such as direct IO and the regular benchmarks
  (read, write, random read, random write, streaming write, etc).
- Specific options: the pool, and the rbd volume you want to use.  It won't
  create it for you!


<iframe src="https://asciinema.org/api/asciicasts/13119?size=medium&amp;theme=solarized-light&amp;speed=2" id="asciicast-iframe-13119" name="asciicast-iframe-13119" scrolling="yes"></iframe>
Note:
- Writing 100GB at a time, and getting an aggregate bandwith of 927MB/s.
- 10% under raw RADOS, but a pretty good result for RBD.
- Results fluctuate a bit depending on the benchmark type.
- Of note, is that Ceph performs particularly well with randwrite benchmarks,
  because it doesn't care that IO is all over the place, because it hits OSDs
  all over the place.
- This runs on 200 OSDs on 4 physical nodes, on all-SSD OSDs.


## rbd bench-write
```sh
rbd bench-write -p bench rbd-test \
    --io-threads=10 \
    --io-size 4096 \
    --io-total 10GB \
    --io-pattern rand
```
Note:
- Recently a new option for the "rbd" tool came out: "bench-write".  You can
  use it in the same spirit as fio with the librbd engine.


## Mind the cache!
Note:
- 900MB/s is pretty good, but we have a wonderful writeback and writethrough
  cache in RBD.
- We can measure it with fio.


<iframe src="https://asciinema.org/api/asciicasts/13118?size=medium&amp;theme=solarized-light&amp;speed=2" id="asciicast-iframe-13118" name="asciicast-iframe-13118" scrolling="yes"></iframe>
Note:
- First test is without caching, and we get roughly 900MB/s.
- We get the same with rbdcache=true, though.  Why?
- Ceph has a feature where the client operates in writethrough mode until it
  receives the first "flush" from upper layers.  When you have an intelligent
  VM guest that is capable of sending down a flush request (which Ceph can only
  tell once the first one comes in), Ceph can decide to enter writeback caching
  mode safely.
- A relatively recent version of QEMU with virtio will do this, but not if the
  VM is running an ancient kernel such as 2.6.32, or something like Windows.
  If you're not using virtio, you're also not going to see flushes.  In this
  case, Ceph defaults to protecting your data (i.e., use only writethrough).
- To test this in FIO, you need to set --fsync=10 (after 10 IOs, send an
  fsync), because the RBD fio engine translates fsync into "flush": and now we
  get over 1GB/s, and we can also see the flushes on the client log.
- Further benchmark tools: rest-bench, for radosgw, fio tests on CephFS,
  IOzone and bonnie++ (results are reportedly difficult to interpret, though)
