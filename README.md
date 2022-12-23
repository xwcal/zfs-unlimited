# zfs-unlimited
An **experimental** patch.

## Problem

Although ZFS boasts support for virtually unlimited datasets, in practice the number of datasets a pool can comfortably accommodate is restricted by a simple limitation: certain important (and inevitable) operations, such as `zpool import`, take time linear in the number of datasets. In my tests, a few thousand datasets are enough to make the slowdown unbearable.

Just as git doesn't limit the number of branches we are allowed to have, there are use cases where practical support for unlimited datasets is desirable:
- virtualization/containerization
- machine learning/AI training
- forensics/bug hunting, and in my case
- [ZFS Time Shuttle (ZTS)](https://github.com/xwcal/zfs-time-shuttle)

I designed ZTS under the impression that it costs nothing to spawn a large number of datasets -- in fact, originally each state had a lot more dimensions, giving a much larger growth factor.

It became clear later that every dataset takes a little time to process during `zpool import` (without mounting, just to clear), and the reason is that `zil_claim` runs on each dataset even if the dataset hasn't been written to for a long time.

## Solution

### Overview

As suggested above, the key to practical support for unlimited datasets is to maintain performance regardless of the number of datasets.

In case of `zpool import`, it's not unreasonable to ask `zpool import` not to bother with dormant datasets that are known to stay unchanged.

This would give us constant time `zpool import` assuming the number of active datasets remains below a small constant.

### Details

The consistency guarantee of ZFS requires a fortiori the consistency of space management details, which is achieved by periodically sealing the pool state including the "space maps" in a new uberblock.

This combined with durability requirements raises the question: where does ZIL (ZFS's write ahead log) keep its data?

If data writen to the disk between uberblocks have to be persisted, then we have to either create a new uberblock for each ZIL extension, or allow incoming writes to stay in unallocated (as of the last uberblock) space!

ZFS chooses the latter approach and deals with the conundrum by running `zil_claim` upon `zpool import`, so that if ZIL has extended into unallocated space in the previous session, then the used space gets "claimed" before all the writing starts.

So, how do we bypass this important procedure?

My patch provides two mechanisms: "lazy import" switch (`zpool import -L`) and "blocked" property of datasets.

If a dataset is blocked, then during a lazy import, the recursive procedure running `zil_claim` simply doesn't descend past this dataset. (Other recursive procedures are handled similarly, but let's focus on `zil_claim` for now.)

So how does this help us?

We can create a blocked dataset, say `archive`, and stash below it everything we want to keep but don't use frequently, for example, some useful experiments we did on a virtual machine / container that we might want to look at again a year later. This allows us to avoid paying a toll for these dormant datasets during `zpool import`.

Simple, right?

However, there is one catch. When an active dataset becomes dormant with its ZIL still full, and if this dataset misses the next `zil_claim`, then not only are the logged writes in limbo, any future zil_claim can protentially corrupt the pool by claiming already allocated space (which gets freed twice later).

The patch handles this case by adding a check that runs during each sync and clears a ZIL if it has missed its `zil_claim`. No attempt is made to salvage the logged writes (although it shouldn't be difficult to replay those writes on already allocated ZIL blocks). If a user wants to avoid losing ZIL backed writes to a dataset, they should wait until after `zil_claim`, ie. after the pool has been exported and imported while the dataset is active, to make it dormant.

## Getting Started (wip)

The patch was written against `zfs-linux` `0.7.12-1ubuntu5` (the lastest Ubuntu release when I started hacking ZFS back in 2018), which produces `zfs-dkms` when built. I have been using my custom `zfs-dkms` in production and so far my data have been intact :)

The patch can also be applied to the stock [`zfs-0.7.12`](https://github.com/openzfs/zfs/releases/tag/zfs-0.7.12) (the result however is not tested). One simple way is to download the release, and then:
```
$ tar -xf zfs-0.7.12.tar.gz
$ patch -d zfs-0.7.12/ -p1 --no-backup-if-mismatch < 5000-Add-lazy-import-feature.patch
```

## Known/Possible issues
The name length check under `dsl_dir_rename_check` should be allowed to finish (spotted this bug in 2021 when I attempted to reboot this project).

My reading of CDDL 1.0 is that section 3.3 doesn't apply until the patch is applied publicly. However, if anyone decides to apply the patch to their public repo then the result is immediately non-compliant ...

## Possible Future Directions

I have made every effort to minimize the footprint of this patch. Unfortunately, it still touches quite a few things.

As ZFS grows, it takes a lot time to keep track of all the changes to make sure no harmful interactions occur, and make adjustments if necessary. For example, right now on the TODO list (from my 2021 effort) are checkpoints and encryption.

The ideal outcome is thus for upstream to take an interest and make improvements in this direction. Hope they find this patch or at least the ideas useful.

A lesser alternative is to keep a perpetual "feature branch", for which I will probably need either community effort or some financial support, short of which I may have to eventually abandon this project.

Please let me know what you think in [issues](../../issues/1).

## Anecdote

Many years ago at a Freemason gathering, when I was introduced as a Windows expert, a gentleman asked: My Windows keeps getting slower and slower. How do I stop that?
The attempt to explain to a novice the performance penalty of unchecked entropy growth solidified the association in my head between a system that slows down over time and poor design.
This is the driving force behind this long detour when I was in the middle of ZTS, the design of which I preferred not to modify.

