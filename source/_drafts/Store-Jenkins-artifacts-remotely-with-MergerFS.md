---
title: Store Jenkins artifacts remotely with WebDAV and MergerFS
date: 2023-12-27 17:37:19
tags:
---
## Motivation
I'm running Jenkins on a Hetzner Cloud instance, a CAX11 (Arm-based), which combined with the Hetzner Cloud plugin that lets Jenkins dynamically spin up instances as needed,
is more than enough for my needs.

It does however, not have sufficient storage for the build artifacts. At the time of writing they amount to 34.4 GiB, so the 40 GB provided with the instance won't last me much longer.

Normally I'd use a larger instance, or add some block storage depending on what makes the most sense. But as it happens I have a 1 TB storage box with them that has plenty available space.
So why not use the storage I'm already paying for anyway?

(Side note: I did attempt to use S3 based storage, but had issues with the "Artifact Manager for S3" plugin and custom endpoints, so I gave up on that idea for now)

## Planned setup
Hetzner's storage boxes support SMB, SFTP/SSH and WebDAV to access the files. None of these can be used directly in Jenkins for artifact storage, or to be more exact you can _publish_ artifacts to SSH at least,
but that won't show up on the build page, so it doesn't quite cover by needs.

Based on some reading online and some simple tests on my own, WebDAV with davfs2 seemed to be the most performant option for transferring files, so that's what I'll be using.
Unfortunately davfs2 isn't quite enough, because artifacts live in the jobs folder together with the builds, so just mounting the storage box on /var/lib/jenkins/jobs would just unnecessarily slow things down.

So I'll be throwing mergerfs into the mix. Mergerfs allows us to merge different filesystems into one. Thus, what we'll be doing is setting up mergerfs with two branches:
* /var/lib/jenkins/jobs_local – Local storage on the instance.
* /var/lib/jenkins/jobs_remote – Remote storage through davfs
These will be merged into /var/lib/jenkins/jobs which Jenkins will use for its jobs.

For performance reasons, I would like artifacts to be stored locally and then later moved to remote storage automatically.
To do this I will set up a systemd oneshot service + timer to regularly move artifact files to the remote storage.
Due to mergerfs "hiding" which filesystem the files reside on, Jenkins won't know the difference.

## Caveats
* davfs2 does not support symlinks. I'm not sure if Jenkins allows archiving symlinks as artifacts, but if it does and you have those in our artifacts you'll have to use something else, or possibly fuse-posixovl on top to make it work
    * SMB can be used here, but at least from my testing the storage boxes doesn't support the Unix extensions, so the `mfsymlinks` option need to be enabled. This will work, but keep in mind that reading those symlinks through another protocol will yield files in the Minshall+French format instead of actual symlinks (see [here](https://wiki.samba.org/index.php/UNIX_Extensions#Minshall.2BFrench_symlinks) for details)