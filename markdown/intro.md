## Ceph performance
# demystified
Benchmarks, Tools, and the Metrics That Matter
Note:
- Introductions
- Partners with Inktank before it was called that (and now it's Red Hat!)
- We've done many Ceph-related projects, including building clusters and
  optimizing them.  One of the questions we get asked most often is: how do I
  make head-and-tail of Ceph performance?
- ...and this is why this talk exists.
- I'll tell you about about the benchmarks and tools you can use to appraise
  your Ceph performance.
- What are the real metrics that are important.


<!-- .slide: data-background="https://farm4.staticflickr.com/3113/3117146807_578f059d67_b_d.jpg" -->
`ceph -s`
# HEALTH_OK
Note:
(Credit: [Aidan Jones](https://www.flickr.com/photos/aidan_jones/), CC-BY-SA,
source: https://flic.kr/p/5Ksc9t)
- This is how I feel when I finish deploying my cluster and get HEALTH_OK.
- Whether using ceph-deploy, or Puppet Ceph Deploy, or Ansible, etc, and
  deploying potentially hundreds of OSDs over dozens of nodes, the first thing
  I want to get is this.


But does it
# perform?
Note:
- The question arises: does it actually perform?
- Does it actually do what I expect it to do given the architecture and
  hardware available?
- When analyzing storage performance, whether Ceph or otherwise, there are
  multiple dimensions we need to look at.


# Latency
Note:
- For example, one of the dimensions we typically look at is latency.
- What is latency?  You take the smallest piece of data you can write to your
  device, or read from it, how long does it take.
- Latency is important for applications such as databases.


# Throughput
Note:
- Another metric is throughput.
- Entirely different from latency: it's not about the time that it takes to
  make the smallest possible write, but the total volume of data that we can
  write/read from the device at an arbitrary time span.
- This is important if you are, for example, streaming media storage, or any
  application that requires storing or fetching large amounts of data at a
  time.


# IOPS
Note:
- IOPs are more or less the reciprocal of latency.
- Instead of how long does a single small write take, how many IO operations
  can I get per second?
- These are the metrics we want to look at, and which of these is important for
  your use case or application?  That depends on your use case. :)
- If building streaming media application, you'll want read throughput.
- If app needs to get to different data fast, you want latency.
- If you're building a general-use Ceph cluster, you'll need to find a balance.
- In addition to these multiple dimensions, there are also multiple components
  you'll want to benchmark.


# RADOS
Note:
- You might be building an application that uses the Object Store, RADOS,
  directly.  If so, you'll be interested in its raw performance.
- But you might also be interested in layers that are on top of it, such as:


# RBD
Note:
- If building a virtualization platform, such as is the case with OpenStack,
  CloudStack, of KVM, you'll use Ceph for volume storage, and you're going to
  be interested specifically on RBD performance.
- RBD is a client built on top of RADOS, so you need to take into account the
  performance of both.


# radosgw
Note:
- If you're working with clients that are using primarily the RESTful API to
  interact with Object Storage, you'll be very interested in radosgw
  performance.
- Like with RBD, you'll need to analyze performance of both the underlying
  Object Store, but the radosgw servers (how many you have, how Apache/nginx is
  performing, how fastcgi is performing, or whatever you're using for a proxy
  server).


# CephFS
Note:
- Finally, if running CephFS, another client layer on top of RADOS, you'll also
  going to be interested in the POSIX filesystem performance, whether using the
  Linux kernel as a client, or ceph-fuse, or libcephfs, or Samba, or something
  else.
- Once again, you need to consider multiple performance dimensions (latency,
  IOPs, throughput), but you also need to appraise your Ceph performance in
  it's multiple layers (RADOS, RBD, radosgw, CephFS).


# Block
Note:
- At the very lowest layer, of crucial importance is the performance of the
  block devices to which data is actually being written to, whether these are
  spinners, or SSDs, or Fusion IO drives, or whatever.


# Network
Note:
- And of course, another item of great importance when measure overall
  performance is your network throughput and latency.
- This includes the client network, used for clients to interact with MONs and
  OSDs: critical because Ceph allows any client to talk to multiple OSDs
  directly and simultaneously
- And also, the network connectivity between OSDs is important because Ceph
  puts a lot of replication inteligence into the OSDs themselves.
- Once again, multiple dimensions, multiple application workloads, and
  multiple layers, including the lowest level, notably block device performance
  and network performance.
