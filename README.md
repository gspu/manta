<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2015, Joyent, Inc.
-->

# Manta: object storage with integrated compute

Manta is an open-source, HTTP-based object store that uses OS containers to
allow running arbitrary compute on data at rest (i.e., without copying data
out of the object store).  The intended use-cases are wide-ranging:

* web assets (e.g., images, HTML and CSS files, and so on), with the ability to
  convert or resize images without copying any data out of Manta
* backup storage (e.g., tarballs)
* video storage and transcoding
* log storage and analysis
* data warehousing
* software crash dump storage and analysis

Joyent operates a public-facing production [Manta
service](https://www.joyent.com/products/manta), but all the pieces required to
deploy and operate your own Manta are open source.  This repo provides
documentation for the overall Manta project and pointers to the other
repositories that make up a complete Manta deployment.

## Getting started

The fastest way to get started with Manta depends on what exactly one
wishes to do.

* To experiment with Manta, the fastest way is to start playing with [Joyent's 
Manta service](https://www.joyent.com/products/manta); see the [Getting
Started](https://apidocs.joyent.com/manta/index.html#getting-started) guide in
the user documentation for details.

* To see a detailed, real example of using Manta, check out [Kartlytics: Applying Big Data Analytics to Mario Kart](http://www.joyent.com/blog/introducing-kartlytics-mario-kart-64-analytics).

* To learn about installing and operating your own Manta deployment, see the
[Manta Operator's
Guide](https://github.com/joyent/manta/blob/master/docs/manta-ops.md).

* To understand Manta's architecture, see 
[Bringing Arbitrary Compute to Authoritative
Data](http://queue.acm.org/detail.cfm?id=2645649), the
[ACM Queue](http://queue.acm.org/)
article on its design and implementation.

* To understand the
[CAP tradeoffs](http://en.wikipedia.org/wiki/CAP_theorem) in Manta,
see [Dave Pacheco](https://github.com/davepacheco)'s blog entry on
[Fault Tolerence in Manta](http://dtrace.org/blogs/dap/2013/07/03/fault-tolerance-in-manta/) -- which, it must be said, received [the highest possible praise](https://twitter.com/eric_brewer/status/352804538769604609).

## Community

Community discussion about Manta happens in two main places:

* The *manta-discuss* mailing list.  Once you [subscribe to the
  list](https://www.listbox.com/subscribe/?list_id=247448), you can send mail
  to the list address: manta-discuss@lists.mantastorage.org.  The mailing list
  archives are also [available on the
  web](https://www.listbox.com/member/archive/247448/=now).

* In the *#manta* IRC channel on the [Freenode IRC
  network](https://freenode.net/).

You can also follow [@MantaStorage](https://twitter.com/MantaStorage) on
Twitter for updates.

## Dependencies

Manta is deployed on top of Joyent's
[SmartDataCenter](https://github.com/joyent/sdc) platform (SDC), which is also
open-source.  SDC provides services for operating physical servers (compute
nodes), deploying services in containers, monitoring services, transmitting and
visualizing real-time performance data, and a bunch more.  Manta primarily uses
SDC for initial deployment, service upgrade, and service monitoring.

SDC itself depends on [SmartOS](http://smartos.org).  Manta also directly
depends on several SmartOS features, notably: ZFS pooled storage, ZFS rollback,
and
[hyprlofs](https://github.com/joyent/illumos-joyent/blob/master/usr/src/uts/common/fs/hyprlofs/hyprlofs_vfsops.c).


## Building and Deploying Manta

Manta is built and packaged with SDC.  Building the raw pieces uses the same
mechanisms as building the services that are part of SDC.  When you build an SDC
headnode image (which is the end result of the whole SDC build process), one of
the built-in services you get is a [Manta
deployment](http://github.com/joyent/sdc-manta) service, which is used
to bootstrap a Manta installation.

Once you have SDC set up, follow the instructions in the 
[Manta Operator's
Guide](https://github.com/joyent/manta/blob/master/docs/manta-ops.md)
to deploy Manta.  The easiest way to play around with your own Manta
installation is to first set up an SDC (cloud-on-a-laptop) installation in
VMware and then follow those instructions to deploy Manta on it.

If you want to deploy your own builds of Manta components, see "Deploying your
own Manta Builds" below.


## Repositories

This repository is just a wrapper containing documentation about Manta.  Manta
is actually made up of several components stored in other repos.

The front door services respond to requests from the internet at large:

* [muppet](https://github.com/joyent/muppet): haproxy + stud-based SSL
  terminator and loadbalancer
* [muskie](https://github.com/joyent/manta-muskie): Node-based API server
* [mahi](https://github.com/joyent/mahi): authentication cache
* [medusa](https://github.com/joyent/manta-medusa): handles interactive (mlogin)
  sessions

The metadata tier stores the entire object namespace (not object data) as well
as information about compute jobs and backend storage system capacity:

* [manatee](https://github.com/joyent/manatee): high-availability postgres
  cluster using synchronous replication and automatic fail-over
* [moray](https://github.com/joyent/moray): Node-based key-value store built on
  top of manatee.  Also responsible for monitoring manatee replication topology
  (i.e., which postgres instance is the master).
* [electric-moray](https://github.com/joyent/electric-moray): Node-based service
  that provides the same interface as Moray, but which directs requests to one
  or more Moray+Manatee *shards* based on hashing the Moray key.

The storage tier is responsible for actually storing bits on disk:

* [mako](https://github.com/joyent/manta-mako): nginx-based server that receives
  PUT/GET requests from Muskie to store object data on disk.

The compute tier (also called [Marlin](https://github.com/joyent/manta-marlin))
is responsible for the distributed execution of user jobs.  Most of it is
contained in the Marlin repo, and it consists of:

* jobsupervisor: Node-based service that stores job execution state in moray and
  coordinates execution across the physical servers
* marlin agent: Node-based service (an SDC agent) that runs on each physical
  server and is responsible for executing user jobs on that server
* lackey: a Node-based service that runs inside each compute zone under the
  direction of the marlin agent.  The lackey is responsible for actually
  executing individual user tasks inside compute containers.
* [wrasse](https://github.com/joyent/manta-wrasse): job archiver and purger,
  which removes job information from moray after the job completes and saves
  the lists of inputs, outputs, and errors back to Manta for user reference

There are a number of services not part of the data path that are critical for
Manta's operation:

* [binder](https://github.com/joyent/binder): hosts both ZooKeeper (used for
  manatee leader election and for group membership) and a Node-based DNS server
  that keeps track of which instances of each service are online at any given
  time
* [mola](https://github.com/joyent/manta-mola): garbage collection (removing
  files from storage servers corresponding to objects that have been deleted
  from the namespace) and audit (verifying that objects in the index tier
  exist on the storage hosts)
* [mackerel](https://github.com/joyent/manta-mackerel): metering (computing
  per-user details about requests made, bandwidth used, storage used, and
  compute time used)
* [madtom](https://github.com/joyent/manta-madtom): real-time "is-it-up?"
  dashboard, showing the status of all services deployed
* [marlin-dashboard](https://github.com/joyent/manta-marlin-dashboard):
  real-time dashboard showing detaild status for the compute tier
* [minnow](https://github.com/joyent/manta-minnow): a Node-based service that
  runs inside mako zones to periodically report storage capacity into Moray

With the exception of the Marlin agent and lackey, each of the above components
are *services*, of which there may be multiple *instances* in a single Manta
deployment.  Except for the last category of non-data-path services, these can
all be deployed redundantly for availability and additional instances can be
deployed to increase capacity.

Finally, scripts used to set up these component zones live in the
[https://github.com/joyent/manta-scripts](manta-scripts) repo.

For more details on the architecture, including how these pieces actually fit
together, see "Architecture Basics" in the 
[Manta Operator's
Guide](https://github.com/joyent/manta/blob/master/docs/manta-ops.md).


## Deploying your own Manta Builds

As described above, as part of the normal Manta deployment process, you start
with the "manta-deployment" zone that's built into SDC.  Inside that zone, you
run "manta-init" to fetch the latest Joyent build of each Manta component.  Then
you run Manta deployment tools to actually deploy zones based on these builds.

The easiest way to use your own custom build is to first deploy Manta using the
default Joyent build and *then* replace whatever components you want with your
own builds.  This will also ensure that you're starting from a known-working set
of builds so that if something goes wrong, you know where to start looking.  To
do this:

1. Complete the Manta deployment procedure from the [Manta Operator's
Guide](https://github.com/joyent/manta/blob/master/docs/manta-ops.md).
1. Build a zone image for whatever zone you want to replace.  See the
   instructions for building [SmartDataCenter](https://github.com/joyent/sdc)
   zone images using Mountain Gorilla.  Manta zones work the same way.  The
   output of this process will be a zone **image**, identified by uuid.  The
   image is comprised of two files: an image manifest (a JSON file) and the
   image file itself (a binary blob).
1. Import the image into the SDC instance that you're using to deploy Manta.
   (If you've got a multi-datacenter Manta deployment, you'll need to import the
   image into each datacenter separately using this same procedure.)
    1. Copy the image and manifest files to the SDC headnode where the Manta
       deployment zone is deployed.  For simplicity, assume that the
       manifest file is "/var/tmp/my_manifest.json" and the image file is
       "/var/tmp/my_image".  You may want to use the image uuid in the filenames
       instead.
    1. Import the image using:

           sdc-imgadm import -m /var/tmp/my_manifest.json -f /var/tmp/my_image

1. Now you can use the normal Manta zone update procedure (from the [Manta
   Operator's Guide](https://github.com/joyent/manta/blob/master/docs/manta-ops.md).
   This involves saving the current configuration to a JSON
   file using "manta-adm show -sj > config.json", updating the configuration
   file, and then applying the changes with "manta-adm update < config.json".
   When you modify the configuration file, you can use your image's uuid in
   place of whatever service you're trying to replace.

If for some reason you want to avoid deploying the Joyent builds at all, you'll
have to follow a more manual procedure.  One approach is to update the SAPI
configuration for whatever service you want (using sdc-sapi -- see
[SAPI](https://github.com/joyent/sdc-sapi)) *immediately after* running
manta-init but before deploying anything.  Note that each subsequent
"manta-init" will clobber this change, though the SAPI configuration is normally
only used for the initial deployment anyway.  The other option is to apply the
fully-manual install procedure from the 
[Manta Operator's
Guide](https://github.com/joyent/manta/blob/master/docs/manta-ops.md)
(i.e., instead of
using manta-deploy-coal or manta-deploy-lab) and use a custom "manta-adm"
configuration file in the first place.  If this is an important use case, file
an issue and we can improve this procedure.

The above procedure works to update Manta *zones*, which are most of the
components above.  The other two kinds of components are the *platform* and
*agents*.  Both of these procedures are documented in the 
[Manta Operator's
Guide](https://github.com/joyent/manta/blob/master/docs/manta-ops.md), and they work to deploy custom builds as well as the official Joyent
builds.


## Contributing to Manta

Manta repositories use the same [Joyent Engineering
Guidelines](https://github.com/joyent/eng/blob/master/docs/index.md) as
the SDC project.  Notably:

* The #master branch should be first-customer-ship (FCS) quality at all times.
  Don't push anything until it's tested.
* All repositories should be "make check" clean at all times.
* All repositories should have tests that run cleanly at all times.

"make check" checks both JavaScript style and lint.  Style is checked with
[jsstyle](https://github.com/davepacheco/jsstyle).  The specific style rules are
somewhat repo-specific.  See the jsstyle configuration file in each repo for
exceptions to the default jsstyle rules.

Lint is checked with
[javascriptlint](https://github.com/davepacheco/javascriptlint).  ([Don't
conflate lint with
style!](http://dtrace.org/blogs/dap/2011/08/23/javascriptlint/)  There are gray
areas, but generally speaking, style rules are arbitrary, while lint warnings
identify potentially broken code.)  Repos sometimes have repo-specific lint
rules, but this is less common.

To report bugs or request features, submit issues to the Manta project on
Github.  If you're asking for help with Joyent's production Manta service,
you should contact Joyent support instead.  If you're contributing code, start
with a pull request.  If you're contributing something substantial, you should
contact developers on the mailing list or IRC first.


## Manta zone build and setup

You should look at the above instructions for actually building and deploying
Manta.  This section is a reference for developers to understand how those
procedures work under the hood.

Most Manta components are deployed as *zones*, based on *images* built from a
single *repo*.  Examples are above, and include *muppet* and *muskie*.

For a typical zone (take "muppet"), the process from source code to deployment
works like this:

1. Build the repository itself.
2. Build an image (which is basically a zone filesystem template) from the
   contents of the built repository.
3. Publish the image to updates.joyent.com.
4. Import the image into an SDC instance.
5. Provision a new zone from the imported image.
6. During the first boot, the zone executes a one-time setup script.
7. During the first and all subsequent boots, the zone executes another
   configuration script.

There are tools to automate most of this (and again, for using them, see the
links above):

* Mountain Gorilla (MG), part of the [SDC](http://github.com/joyent/sdc) build
  process, takes care of steps (1) through (3).  It does this by cloning the
  repo, using a "make" target to build a tarball to be splatted down onto a bare
  zone, deploys a bare zone, splats down the tarball, and uses the SDC APIs to
  create a new image from that zone.  This image basically represents a template
  filesystem with which instances of this component will be stamped out.  After
  the image is built, it gets uploaded to updates.joyent.com.
* The "manta-init" command takes care of step 4.  You run this as part of any
  deployment.  See the [Manta Operator's Guide](https://joyent.github.io/manta)
  for details.  After the first run, basically all it does is find new images in
  updates.joyent.com, import them into the current SDC instance, and mark them
  for use by "manta-deploy".
* The "manta-adm" and "manta-deploy" commands (whichever you choose to use) take
  care of step 5.  See the Manta Operator's Guide for details.
* Steps 6 and 7 happen automatically when the zone boots as a result of the
  previous steps.

For more information on the zone setup and boot process, see the
[manta-scripts](https://github.com/joyent/manta-scripts) repo.


## Design principles

Manta assumes several constraints on the data storage problem:

1. There should be one *canonical* copy of data.  You shouldn't need to copy
   data in order to analyze it, transform it, or serve it publicly over the
   internet.
1. The system must scale horizontally in every dimension.  It should be possible
   to add new servers and deploy software instances to increase the system's
   capacity in terms of number of objects, total data stored, or compute
   capacity.
1. The system should be general-purpose.  (That doesn't preclude
   special-purpose interfaces for use-cases like log analysis or video
   transcoding.)
1. The system should be strongly consistent and highly available.  In terms of
   [CAP](http://en.wikipedia.org/wiki/CAP_theorem), Manta sacrifices
   availability in the face of network partitions.  (The reasoning here is that
   an AP cache can be built atop a CP system like Manta, but if Manta were AP,
   then it would be impossible for anyone to get CP semantics.)
1. The system should be transparent about errors and performance.  The public
   API only supports atomic operations, which makes error reporting and
   performance easy to reason about.  (It's hard to say anything about the
   performance of compound operations, and it's hard to report failures in
   compound operations.)  Relatedly, a single Manta deployment may span multiple
   datacenters within a region for higher availability, but Manta does not
   attempt to provide a global namespace across regions, since that would imply
   uniformity in performance or fault characteristics.

From these constraints, we define a few design principles:

1. Manta presents an HTTP interface (with REST-based PUT/GET/DELETE operations)
   as the primary way of reading and writing data.  Because there's only one
   copy of data, and some data needs to be available publicly (e.g., on the
   internet over standard protocols), HTTP is a good choice.
1. Manta is an *object store*, meaning that it only provides PUT/GET/DELETE for
   *entire objects*.  You cannot write to the middle of an object or append to
   the end of one.  This constraint makes it possible to guarantee strong
   consistency and high availability, since only the metadata tier (i.e., the
   namespace) needs to be strongly consistent, and objects themselves can be
   easily replicated for availability.
1. Users express computation in terms of shell scripts, which can make use of
   any programs installed in the default compute environment, as well as any
   objects stored in Manta.  You can store your own programs in Manta and use
   those, or you can use tools like curl(1) to fetch a program from the internet
   and use that.  This approach falls out of the requirement to be a
   general-purpose system, and imposes a number of other constraints on the
   implementation (like the use of strong OS-based containers to isolate users).
1. Users express distributed computation in terms of map and reduce operations.
   As with Hadoop and other MapReduce-based systems, this allows the system to
   identify which parts can be parallelized and which parts cannot in order to
   maximize performance.

It's easy to underestimate the problem of just reliably storing bits on disk.
It's commonly assumed that the only components that fail are disks, that they
fail independently, and that they fail cleanly (e.g., by reporting errors).  In
reality, there are a lot worse failure modes than disks failing cleanly,
including:

* disks or HBAs dropping writes
* disks or HBAs redirecting both read and write requests to the wrong physical
  blocks
* disks or HBAs retrying writes internally, resulting in orders-of-magnitude
  latency bubbles
* disks, HBAs, or buses corrupting data at any point in the data path

Manta delegates to ZFS to solve the single-system data storage problem.  To
handle these cases,

* ZFS stores block checksums *separately* from the blocks themselves.
* Filesystem metadata is stored redundantly (on separate disks).  Data is
  typically stored redundantly as well, but that's up to user configuration.
* ZFS is aware of how the filesystem data is stored across several disks.  As a
  result, when reads from one disk return data that doesn't match the expected
  checksum, it's able to read another copy and fix the original one.

For a more detailed discussion, see the ACM Queue article "Bringing Arbitrary
Compute to Authoritative Data".



## Further reading

For background on the problem space and design principles, check out ["Bringing
Arbitrary Compute to Authoritative
Data"](http://queue.acm.org/detail.cfm?id=2645649).

For background on the overall design approach, see ["There's Just No Getting
Around It: You're Building a Distributed
System"](http://queue.acm.org/detail.cfm?id=2482856).

For information about how Manta is designed to survive component failures and
maintain strong consistency, see [Fault tolerance in
Manta](http://dtrace.org/blogs/dap/2013/07/03/fault-tolerance-in-manta/).

For information on the latest recommended production hardware, see [Joyent
Manufacturing Matrix](http://eng.joyent.com/manufacturing/matrix.html) and
[Joyent Manufacturing Bill of
Materials](http://eng.joyent.com/manufacturing/bom.html).

Applications and customer stories:

* [Kartlytics: Applying Big Data Analytics to Mario
  Kart](http://www.joyent.com/blog/introducing-kartlytics-mario-kart-64-analytics)
* [A Cost-effective Approach to Scaling Event-based Data Collection and
  Analysis](http://building.wanelo.com/2013/06/28/a-cost-effective-approach-to-scaling-event-based-data-collection-and-analysis.html)
