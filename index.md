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

Because the graphs are not just static plots, but rich [plotly](https://plotly.com) Javascript objects, rendering that notebook's output to static graphics has proven difficult.

So I redid the tests with [fio](https://github.com/axboe/fio/) and the plots in Matplotlib, and that let me export the notebook as PDF much more easily.

### Summary

NFSv3 performs as expected.  Notably, once you get data large enough to exhaust the cache on the Kubernetes container, and files sufficiently large that it is in fact the transfer and not the open/close/metadata manipulation that dominates the time, you run into the defined service limits.

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

## NFSv4

We want to use NFSv4 in order to be able to represent all the groups a user may be in, rather than only the first 15, as we anticipate users may belong to many collaborations.
NFSv4 can be used with NetApp (or with a higher tier of Filestore).
It is not without its idiosyncrasies.

### Kubernetes difficulties

The problem lies mostly in the complexity of plumbing it through to the Kubernetes layer.
For NFSv3, you can use the `nfs` volume type; Kubernetes then does magic behind the scenes to map the equivalent of `PersistentVolumes` (PVs) and `PersistentVolumeClaims` (PVCs) without the K8s administrator having to be aware of it.
In fact, the `nfs` volume type will also work with NFSv4--this is what we are using at the T&S sites.
The problem is that that requires changes on the Kubernetes nodes, in particular the creation of `/etc/nfsmount.conf` on each node forcing the NFS server to v4.
Our GKE nodes are Container Operating System (COS) images.
COS claims to support cloud-init, and those images claim that `/etc` is mutable.
Thus, in principle, we could inject an appropriate `nfsmount.conf` onto each note.
In practice, I could not find any way to specify a cloud-init server that the GKE nodes could consult at boot time.
Obviously manual alteration of our nodes is a non-starter.

### PVs and PVCs

There is, however, a way to do NFSv4 without any additional support below the Kubernetes layer.
Although you cannot specify mount options with the `nfs` volume type, if you create a `PersistentVolumeClaim` (PVC) and map it to a `PersistentVolume`, you can indeed specify mount options in the persistent volume.
There are two drawbacks to this approach.
The PV-to-PVC mapping is inherently one-to-one.
Thus, we must create a PV and PVC pair for each distinct mount a pod wants to use.
That brings us to the second drawback.
The PVC, as with everything else in the RSP user pod model, is a namespace-scoped object.
That means you don't have to delete it individually when you're done with a user lab; you just destroy the namespace and everything inside it will be cleaned up.
A PV, however, is a cluster-scoped object.
This means that you end up encoding the namespace in the PV name to keep it unique, and that you must track and destroy these objects separate from the rest of the Lab-or-fileserver resources.
This introduces new complexity into the Nublado Controller and into the Safir Kubernetes test framework.
This is the approach we took at NCSA, because they were unable to deliver an NFSv3 implementation that allowed correct file locking.
It is somewhat fragile, although the Nublado Controller makes it far more reliable than trying to do it all within KubeSpawner as we did at NCSA.

### Trident

The [Trident](https://docs.netapp.com/us-en/trident/index.html) operator might be able to help with this.
In particular, [sharing NFS volumes across namespaces](https://docs.netapp.com/us-en/trident/trident-use/volume-share.html) looks fairly promising.
It is obviously doing the same sort of PV-PVC pair management, but if it hides that from the administrator, such that all we have to do is create a PVC for the user mounts and annotate them with the fileserver namespace for each one we want to expose via the fileserver, then maybe that's a workable solution.

I've done some experiments with Trident, and it really doesn't want to be used the way we want to use it.
That said, I may still be able to hammer it into shape. 
While initial volume creation takes a lot longer, its ability to mirror PVCs to other namespaces might offset this.
As of yet I haven't been able to make it respect `no_root_squash`.

### NFS v4 performance

Benchmarking was surprising under IOZone.
NFSv3 and NFSv4 feel roughly similar in terms of speed in interactive use.
For large block sizes, the benchmarks agree pretty well.
For small files, or small ranges of bytes manipulated, NFSv4 performance is dreadful, often 10% or less of the throughput of NFSv3.
While this isn't apparent to the user (because the difference between e.g 2 and 20 milliseconds is not noticeable to a human, I suspect), it is something to consider, particularly as we have done no scale tests with file services.
I don't know enough about NFS protocols (either v3 or v4) to know whether NFSv4 does a lot more per-action setup and teardown, but the numbers certainly suggests that it does.

I opened a support ticket with Google, that got referred to NetApp.  They confirmed that NFSv4 would have worse performance for small files, and especially for large numbers of small files.

Indeed, when I repeated the IO benchmarking with `fio`, the difference between NFSv3 and NFSv4 was much smaller, and I think that is because `fio` uses one file per job and I intentionally serialized jobs.

## User Quota Support

I have fixed the problem that the Lab will silently fail to start if its home directory cannot be written to; a little bit of Lab extension work has given us a modular dialog indicating the problem.
We even try to take corrective action (by clearing caches) if we can.

Failure to write begs the question of why that space would not be writeable.
At USDF this is usually because the directory ownership is wrong.
At IDF (that is, Google Cloud), the directory ownership is never wrong, because we provision the filesystem on the user's first usage of it.
There are some subtleties regarding file migration from users who have changed their IDF account, perhaps because they moved from one institution to another.
These are relatively rare, and for now can continue to be managed manually.

The next reason the space could not be written is the one we're worried about.
At the time of writing, all our remote storage is on a single volume--so it's not just a runaway process in a user's home directory that could break spawns for everyone--it's something that runs away in `/scratch` too!
We will address that failure mode by separate volumes for `/home`, `/scratch`, and `/project`, at least in production.
I have added Terraform support for NetApp Cloud Volumes to [IDF Deploy](https://github.com/lsst/idf_deploy).

That still leaves the problem that a single user, who fills the volume that home directories are on, can prevent operation for everyone.

Individual-user quotas are the obvious mechanism to protect against that.

There is another alternative--individual user home volumes.
NetApp Volumes, with its Flex volume allocation and the Trident operator, would let us do that.
That seems like overkill, although it would be using Trident more in the way it wants to be used.

### Default and multi-volume quota support

NetApp Volumes default user quota support works as advertised.
This is really all we need to go to DP1, but we want something more sophisticated in the longer term.
Nevertheless, it solves our largest problem.

NetApp Volumes have a limit of 100 quota rules per volume.
Since we will have considerably more users than that, the naive approach of giving each user an independent quota cannot work.

We believe that we can manage this by letting almost everyone use a default quota, combined with placing collaborations with large space requirements on their own volume (since volumes can be a minimum of 2TB, if we need a lot of space per user, we will almost certainly want more than 2TB in the volume).

We can still have up to 99 users per volume who are special snowflakes with larger quotas.
I think we have some candidates in mind.

Our `idf_deploy` Terraform now supports default and individual quota rules on NetApp cloud volumes.

## Performance

[PDF Export of Benchmarking Notebook](assets/compare.pdf)

The tl;dr is that the NetApp implementation is a lot slower than Filestore, but that NetApp NFSv3 and NFSv4 are pretty close to one another in speed.

## Addendum

See the [Documenteer documentation](https://documenteer.lsst.io/technotes/index.html) for tips on how to write and configure your new technote.
