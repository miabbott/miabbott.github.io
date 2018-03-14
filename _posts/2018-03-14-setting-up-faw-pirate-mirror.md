---
layout: post
title:  Building a Pirate Mirror for Fedora Atomic Workstation
---

In the #atomic channel, I've seen a few complaints about slow speeds when
trying to fetch the [Fedora Atomic Workstation](https://fedoraproject.org/wiki/Workstation/AtomicWorkstation) (FAW) content from the official
sources.  Especially when connecting from a location in Europe.  This is a
two-fold problem.  First, the official FAW content is located in a datacenter in
Phoenix, Arizona in the United States.  Second, the FAW content is not mirrored
as part of the official Fedora mirror network.

It is discouraging to see users who want to participate in the [Project Atomic](https://projectatomic.io)
community being frustrated with slow speeds, so I decided I would investigate
how to mirror the content in the European region.

## Building A Host and Retrieving Content

Since I previously had a [Digital Ocean](https://www.digitalocean.com) account
and they offered Fedora 27 Atomic Host as a VM option, I decided I would explore
setting up a mirror using one of their droplets.  I booted up an F27AH droplet
and immediately used `rpm-ostree upgrade` to get the OS up to date.  Once I had
the OS up to date, I could start to think about how retrieve and host the FAW
content on the host.

Thankfully, the [ostree](https://ostree.readthedocs.io/en/latest/) model for distributing content allows for easy mirroring
of a repo using native functionality and this is covered nicely in the
documentation.  I initialized a repo and began the mirroring process.

```
# mkdir -p /var/srv/workstation
# cd /var/srv/workstation
# ostree --repo=repo init --mode=archive
# ostree --repo=repo remote add --set gpgkeypath=/etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-27-primary onerepo https://kojipkgs.fedoraproject.org/atomic/repo/
# ostree --repo=repo pull --mirror --depth=1 onerepo:fedora/27/x86_64/workstation
```

I'll point out here that it is important to use the `--mirror` flag when
setting up the mirror for this content, as this will properly setup the refs
in the `heads/` directory rather than the `remotes/` directory.  Quoting the
`ostree pull` [manpage](https://github.com/ostreedev/ostree/blob/master/man/ostree-pull.xml#L108):

>This makes the target repo suitable to be exported for other clients to pull
>from as an ostree remote.

Additionally, I've specified a depth of '1' for the pull operation to only
pull the very latest FAW content.  I chose to do this to 1) reduce the amount
of time required to pull/push the content around and 2) because it is not
necessary to mirror every commit in the repo for it to operate as an upgrade
target.  This means that anyone using this 'pirate' repo will only be able
to upgrade to the latest available commit in the mirror, rather than being
able to deploy any commit in the history.

The process of mirroring the content from the official sources took a good
amount of time, but I ended up with about 2 GB of content spread across 112828
objects.  All contained in a two commits:

```
# ostree --repo=repo log fedora/27/x86_64/workstation
commit 88ef5feb77aebc7fec3e4fe6c17c490d1b5dc076927f07aa964a6da6fd336970
ContentChecksum:
bedd5851089322a4064f5eeee4b0b9187cfcd2d2dc4a4561152dfbb10bc7c6ab
Date:  2018-03-01 16:15:51 +0000
Version: 27.86
(no subject)

commit 0feaa33a9e102a24cdc4a18e6a77da218f2d64cec6113ac173196310d1e5ebfc
ContentChecksum:
a13cc181ae63014fe086298b74b99df79008f91640812f3ebb2e84ba87f91ce3
Date:  2018-02-27 17:03:09 +0000
Version: 27.85
(no subject)

<< History beyond this commit not fetched >>
```

## Serving Up the Mirror

With the content pulled to my VM, I needed to make it available via HTTP(S) and
since I'm on F27AH, I needed a container run a web server.  Despite having no
experience using it, I decided I would work with the [nginx container](https://hub.docker.com/_/nginx/)
to serve up the mirror content.

After pulling the container, I spent some time learning how to configure nginx
and the right way to invoke the container with the content mounted into it.  I
ended up with a `docker run` command that looked like this:

`# docker run -v /etc/nginx/default.conf:/etc/nginx/conf.d/default.conf -v /var/srv/workstation/repo/:/usr/share/nginx/html/repo:ro -d -p 80:80 docker.io/nginx`

And the `default.conf` was relatively simple, like so:

```


With the content pulled locally to the VM, the next step was to figure out how
to push it to the object store.  I discovered that the object store had a
RESTful XML API that was interoperable with the AWS S3 API and I would be able
to use the `s3cmd` script to interact with the object store.  Since I was using
a Fedora 27 Atomic Host, it only made sense to build a container that I could
use to execute the `s3cmd` command.

I created a simple Dockerfile that looked like this:

```
# cat s3cmd/Dockerfile
FROM registry.fedoraproject.org/fedora:27
RUN dnf -y install s3cmd gnupg && \
    dnf clean all
COPY ams3 /root/
ENTRYPOINT ["/usr/bin/s3cmd", "-c", "/root/ams3"]
```

In this Dockerfile, I copied the `ams3` config file to the container, which
contained the necessary information to interact with the object store.  The
`ENTRYPOINT` was configured to use this config file by default.  Now I could
pass in any sub-command to the container that I needed to and not worry about
the config file.

To push the FAW content to the object store, I used the `sync` sub-command of
`s3cmd` to perform an `rsync`-like operation from the local repo to the object
store.  During this sync, I made sure to set the ACL on all the files to
'public' so that they could all be read without any need for access keys.  I
also specified a cache file that would hold the MD5 checksums of the files
being sync'ed.  This would allow me to speed up the checksum process in the
future, when I want to push newere content to the object store.  When
running the container, I also mounted in the directory containing the repo so
that the `s3cmd` could access it successfully.

`# docker run -it -v /var/srv:/host/var/srv s3cmd:ams3 -v --acl-public
--recursive --cache-file=/host/var/srv/fedora/atomic/workstation/s3_cache sync
/host/var/srv/fedora/atomic/workstation/ s3://f27-atomic-content/workstation/`

This was another process that took a long, long time to complete (over 10
hours!).  When it completed, I verified that I could configure a temporary repo
on the Fedora 27 AH VM and pull from the object store:

```
# mkdir -p /var/srv/temp
# cd /var/srv/temp
# ostree --repo=repo init --mode=archive
# ostree --repo=repo remote add --set gpgkeypath=/etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-27-primary object-store https://f27-atomic-content.ams3.digitaloceanspaces.com/workstation/repo
# ostree --repo=repo pull --commit-metadata-only object-store:fedora/27/x86_64/workstation



GPG: Verification enabled, found 1 signature:

  Signature made Thu 01 Mar 2018 04:16:02 PM UTC using RSA key ID
F55E7430F5282EE4
  Good signature from "Fedora 27 <fedora-27@fedoraproject.org>"

1 metadata, 0 content objects fetched; 569 B transferred in 2 seconds
```

Success!  Now for a full-fledged test, I spun up anoter Fedora 27 AH VM in the
Digital Ocean London data center and tried to pull all of the FAW content from
the object store.

```

