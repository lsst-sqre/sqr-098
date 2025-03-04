# File Storage Options at Google Cloud

```{abstract}
A collection of experiments and observations from various approaches to POSIX file storage for the Google Cloud RSP instances.
```

## Overview and Motivations

We have been using Google Filestore, and the default NFS driver, to provide POSIX file storage to the RSP instances hosted at Google Cloud.

This has worked functionally well enough.
However, there is one huge drawback that makes it completely inappropriate for deployment in DP1 and beyond: there is no support for user quotas.
That means that a single user's runaway process can fill storage (exacerbated for the moment by the fact that `/project`, `/scratch`, and `/home` are all on the same filestore instance) and prevent any user labs from spawning.

The usual recommendation for people who want advanced functionality in their NAS-like solutions in Google Cloud is to use Google Cloud NetApp Volumes.
This is our attempt to document functionality and performance considerations while comparing Filestore and NetApp Volumes.

## Performance

We did a fair bit of performance testing using https://github.com/lsst-sqre/fs-vs-netapp .
The output of the [IOZone](https:www.iozone.org) benchmarking tool can be found in that repository, as can a notebook that generates some fairly understandable graphs from it.

Some of those will be reproduced here.
However, in general, because the graphs are not just static plots, but rich [plotly](https://plotly.com) Javascript objects, rendering that notebook's output to static graphics has proven difficult.
Hence the screenshots you will find in this document.

### Summary

It performs as expected.  Notably, once you get data large enough to exhaust the cache on the Kubernetes container, and files sufficiently large that it is in fact the transfer and not the open/close/metadata manipulation that dominates the time, you run into the defined service limits.

For NetApp we chose the `Premium` tier, as the closest approximation to the `BASIC_SSD` we're using with FileStore.
FileStore `Zonal`, `Regional`, or `Enterprise` would be required to use NFSv4 with Filestore.

For NetApp, the performance limit is 64KiB/s per GiB per volume, with a maximum of 4.5GiBps for a standard volume and 12.5GiBps for large capacity volumes (starting at 15,360 GiB).
Filestore has a fixed maximum read speed of 1200 MiB/s and write speed of 350 MiB/s for the current `BASIC_SSD` tier.
In the `Zonal` tier, Filestore can do 260MiB/s per TiB read, 88MiB/s write, which is, effectively, "A little faster writing, 4x the speed reading."

For production volumes, which are the only ones about whose performance
we will care, we should expect NetApp to be somewhat faster than the
current state of the RSP (although given our workload, usage of
extremely large files should be rare in any event).

Given that moving to a Filestore service tier that could support NFSv4
would require just as intrusive a migration as going to NetApp, that
NetApp is not significantly more expensive, and that NetApp does support
individual quotas and Filestore does not and likely never will, there
doesn't seem to me to be much point in investigating NFSv4 Filestore further.

## User Quota Support

As it stands, if the volume that `/home` is on is full, no user fileservers will start, as RSP startup relies on being able to write files to user home space.
This is obviously completely unacceptable along several orthogonal axes.
The first thing to solve is to not refuse to start if the home directory is not writeable for some reason.
I feel it is acceptable to refuse to start if the home directory does not exist or is not readable, but failure to write should not prevent Lab spawn.

However, that then begs the question of why that space would not be writeable.
At USDF this is usually because the directory ownership is wrong.
At IDF (that is, Google Cloud), the directory ownership is never wrong, because we provision the filesystem on the user's first usage of it.
There are some subtleties regarding file migration from users who have changed their IDF account, perhaps because they moved from one institution to another.
These are relatively rare, and for now can continue to be managed manually.

The next reason the space could not be written is the one we're worried about.
At the time of writing, all our remote storage is on a single volume--so it's not just a runaway process in a user's home directory that could break spawns for everyone--it's something that runs away in `/scratch` too!
We will address that failure mode by separate volumes for `/home`, `/scratch`, and `/project`, at least in production.

That still leaves the problem that a single user, who fills the volume that home directories are on, can prevent operation for everyone.

Individual-user quotas are the obvious mechanism to protect against that.

There is another alternative--individual user home volumes.
NetApp Volumes, with its Flex volume allocation and the Trident operator, would let us do that.
That seems like overkill.

### Default user quota support

NetApp Volumes default user quota support works as advertised.
This is really all we need to go to DP1, but we want something more sophisticated in the longer term.
Nevertheless, it solves our largest problem.

### Group Support

NetApp Volumes have a limit of 100 quota rules per volume.
Since we will have considerably more users than that, the naive approach of giving each user an independent quota cannot work.
It seems likely to be the case that we will want more than 99 users who have something other than the default quota.

It is not yet clear whether a `group` quota refers to the `group` as which a file is written or to the `group` to which the user belongs.
Since NFS supports a list of user groups (historically short) we are hoping that it is the second, but experimentation is required.
NFSv4 `AUTH_SYS` might require that our client-local groups also be known to the NetApp server, which in turn would require that, since we have no access to the local `/etc/group` file or moral equivalent, we end up with LDAP, possibly in conjunction with Kerberos, which all seems distressingly heavyweight.
Filestore supports NFSv4, but only if we migrate to Zonal, Regional, or Enterprise storage tiers (we are currently on Basic SSD).
NetApp certainly does.

## NFS v3 vs v4

NetApp can definitely run NFSv4.
Filestore could if we moved to a more expensive (and featureful) storage tier.

Simply because of the number-of-group restrictions (assuming that NFSv4
is happy with a numerical list that it doesn't have to resolve), it is clear that NFSv4, all other things being equal, would be preferable.

First and foremost among those things that might not be equal is performance.
We need to mount filesystems under each implementation (if possible) and compare their throughput under benchmark tests.
It is my strong suspicion that whatever the actual limits, the actual performance limitation will come from the service throttles (as it appears to under NFSv3 now) rather than any intrinsic capability.

We are currently using the in-tree NFS storage provisioner.
That does not directly support NFSv4~~, but it might if `/etc/nfsmount.conf` is configured to request `vers=4.1`~~.
~~It is certainly the case that a helm-templated ConfigMap to populate `/etc/nfsmount.conf` would be less work than dealing with a CSI, a custom StorageClass, and provisioning and deprovisioning PVs directly.~~
If we are willing to go to a different CSI (and thus worry about provisioning PVs and then PVCs atop them), then we will be able to use NFSv4.
This was the approach we took at NCSA, and it was workable, albeit inconvenient and slightly fragile.
Given the much more methodical way we are handling container spawn now as compared to then (standalone controller providing spawn methods, rather than shoehorning a bunch of stuff into KubeSpawner) it should be less difficult and less fragile this time around.

## Addendum

See the [Documenteer documentation](https://documenteer.lsst.io/technotes/index.html) for tips on how to write and configure your new technote.
