---
layout: post
title:  Fedora Silverblue: Day 1
---

Getting started with [Fedora Silverblue](https://silverblue.fedoraproject.org) is much more
simpler than you might think.  Since all of the content used by the Silverblue
OS is the same Fedora RPMs that you are used to, you only need to learn a few
new workflows to get up and running in your new workstation.

This guide will lead you through some of the next steps after successfully
installing Fedora Silverblue.  We'll go through the following topics:

  - performing OS updates and package installs with [rpm-ostree](https://github.com/projectatomic/rpm-ostree)
  - the basics of [ostree](https://github.com/ostreedev/ostree)
  - running your first container with [podman](https://github.com/containers/libpod)
  - building your first container with [buildah](https://github.com/containers/buildah)
  - setting up [Flathub](https://flathub.org) and installing Flatpaks
  - switching out the OS entirely

## Exercise 0:  Install Fedora Silverblue

In order to proceed with the remainder of the guide, you'll need a working
Silverblue installation.  You can install directly onto your laptop/workstation or
install to a VM using [virt-manager](https://virt-manager.org/) or [GNOME Boxes](https://wiki.gnome.org/Apps/Boxes).
This guide will not cover the install portion using any of these methods.  It is
recommended that you follow the [install guide](https://docs.fedoraproject.org/en-US/fedora-silverblue/installation-guide/) on the Silverblue docs site
for guidance.

Here's a hint, though...the installer for Fedora Silverblue looks a lot like the
installer for Fedora Workstation because they are the same!  If you've ever done
a [Fedora Workstation install](https://docs.fedoraproject.org/en-US/fedora/f29/install-guide/), you'll be right at home!

Once you have completed the install, you can move on to the next step.

## Exercise 1: Upgrade the OS

Just like Fedora Workstation, the installation media for Fedora Silverblue is
not updated after release.  This means the version of your install is very much
out of date and missing important upgrades.  Obviously, the first thing to do
is to get those updates!

After logging into your Silverblue install, open up a terminal and use `rpm-ostree status`
to check the version of your Silverblue install.  It should look similar to this:

```
$ rpm-ostree status
State: idle
AutomaticUpdates: disabled
Deployments:
● ostree://fedora-workstation:fedora/29/x86_64/silverblue
                   Version: 29.1.2 (2018-10-24 23:20:30)
                    Commit: f17b670fa8cf69144be5ae0c968dc2ee7eb6999a5f7a54f1ee71eec7783e434a
              GPGSignature: Valid signature by 5A03B4DD8254ECA02FDA1637A20AA56B429476B4
```

*NOTE:* if you are following this guide at DevConf 2019, first configure the local mirror
as described in the class, otherwise proceed as written.

Note the timestamp on the `Version` field which states the content is from October 2018.
Additionally, you should notice the `AutomaticUpdates` field is set to disabled and the
'State' field is `idle`; these will soon be changed as part of this exercise.

Now, we could manually upgrade the host via the CLI, but for this exercise we will
configure automatic updates and kick off the service that will fetch and deploy the update.

To enable automatic updates, we will need to edit `/etc/rpm-ostreed.conf` and put the
line `AutomaticUpdatePolicy=stage` into the file.  This instructs the `rpm-ostree` daemon
to fetch any available updates and depoly them to disk.  After making the edit to the
file, use `rpm-ostree reload` to reload the `rpm-ostree` daemon.  Then use
`systemctl enable rpm-ostreed-automatic.timer --now` to enable and start the timer for
the automatic upgrades.

If we check the output of `rpm-ostree status`, we can see that the `AutomaticUpdates` field
now has a different value.

```
$ rpm-ostree status
State: idle
AutomaticUpdates: stage; rpm-ostreed-automatic.timer: inactive
Deployments:
● ostree://fedora-workstation:fedora/29/x86_64/silverblue
                   Version: 29.1.2 (2018-10-24 23:20:30)
                    Commit: f17b670fa8cf69144be5ae0c968dc2ee7eb6999a5f7a54f1ee71eec7783e434a
              GPGSignature: Valid signature by 5A03B4DD8254ECA02FDA1637A20AA56B429476B4
```

Since we don't want to wait for the timer to fire, we can manually kick off the automatic
update service with `systemctl start rpm-ostreed-automatic.service`.  After doing so, inspect
the output of `rpm-ostree status` again and you should see that the `State` field is no longer
`idle`.

```
<< insert output of rpm-ostree status during background upgrade >>
```

The upgrade will run in the background and will take some time, so let it run for a
few minutes and periodically check `rpm-ostree status` to see if it has completed.
When it has completed, you'll see two deployments listed - your current deployment
that you are booted into and the upgraded deployment that is ready to be rebooted
into.

```
<< insert output of rpm-ostree status after upgrade >>
```

Did you catch that?  Your running OS has not been touched by the upgrade process and
you'll need to reboot your system in order to start using the new deployment.  This is
the tradeoff to the `ostree` model, but provides exceptional protection from interupted
uploads that make your system inoperable.

When you are ready to start using the new version of Silverblue, reboot your host and
move on to the next step.

## Exercise 2: OSTree Basics

We did the host OS upgrade first before diving into the basics of OSTree in order to
protect our system by applying the latest versions of packages that likely included
sercurity updates.  Now that our system is that much more secure, we can do a simple
exercise to demonstrate some of the basics of the OSTree model.

As stated on the documentation, `ostree` is a tool for managing multiple versioned
filesystem trees.  `rpm-ostree` utlizes `libostree` to deliver an entire OS that was
built from RPMs.  But we can show off how `ostree` operates with just some simple
files.

To start, we need to initialize an `ostree` repository to store the checksummed
files that we want to track.  We'll create a temporary directory and then two
subdirectories named `repo` and `tree`. Then we will initialize the `ostree`
repo in the `repo` directory.

```
$ cd $(mktemp -d)
$ mkdir {repo,tree}
$ ls
repo  tree
$ ostree --repo=repo init
$ ls repo
config  extensions  objects  refs  state  tmp
$ cat repo/config
[core]
repo_version=1
mode=bare
```

With the repo intialized, we can descend into the `tree` directory and start
populating some files.  Then we will make the first commit into the repo with
the few files we have. For our first commit, we chose the arbitrary branch name
of `test` and gave a simple commit message.

```
$ cd tree
$ echo "foobar" > 1
$ echo "silverblue" > 2
$ cp /usr/share/dict/words words
$ ostree --repo=../repo commit --branch=test --subject="initial commit"
92ee9dc64aea6ef5921be8cc3d27146a6ede18ecfcbaca4c2c252c0ddaac7f8b
```

With the inital files committed, let's make a copy of the `words` file
and commit that. Then we'll inspect the commit log of the repo.

```
$ cp words words2
$ ostree --repo=../repo commit --branch=test --subject="words copy"
ffb53a60265b213ca45e1b3d672f621ebed3d63b1d2bc755e0d39c05ef7c473f
$ ostree --repo=../repo log test
commit ffb53a60265b213ca45e1b3d672f621ebed3d63b1d2bc755e0d39c05ef7c473f
ContentChecksum:  c57c27e0e13aaaa8a4e5bdaed308db5211d981cc8696f45d24c0de1ee56c34a6
Date:  2019-01-23 04:13:27 +0000

    words copy

commit 92ee9dc64aea6ef5921be8cc3d27146a6ede18ecfcbaca4c2c252c0ddaac7f8b
ContentChecksum:  d00d6fb0ef88378610674ac63e15af410330f684a606a269b10834efceb759c3
Date:  2019-01-23 04:10:11 +0000

    initial commit
```

Nothing terribly exciting so far.  So let's do a checkout of our repo and inspect
some of the filesystem.  What we'll find is that we don't have two copies of the
`words` file, but the two copies are hardlinked to the same inode on the disk.

```
$ pwd
/tmp/tmp.7lewRF8Lgt/tree
$ cd ..
$ mkdir checkout && cd checkout
$ ostree --repo=../repo checkout test
$ cd test
$ ls -li
total 9688
73201 -rw-rw-r--. 2 miabbott miabbott       7 Dec 31  1969 1
73199 -rw-rw-r--. 2 miabbott miabbott      11 Dec 31  1969 2
73196 -rw-r--r--. 3 miabbott miabbott 4953680 Dec 31  1969 words
73196 -rw-r--r--. 3 miabbott miabbott 4953680 Dec 31  1969 words2
```

That first number in the output is the inode and you can see that the
two copies of `words` link to the same inode.  This is incredibly useful
for Silverblue (or any `ostree` based OS) because it eliminates the need
to download and write to disk multiple copies of the same file.  Think of
files that may appear in multiple RPMs...on a traditional Fedora host, each
RPM that has that file would write it out separately during install.  But
here on Silverblue, we only download and write the file to disk once, then
hardlink the additional copies to that original inode.

There are many other features to `ostree` that we could get into, but we will
continue with more exercises that will be more relevant to your day to day
usage with Fedora Silverblue.

## Exercise 3: Using Flatpaks


## Exercise 4: Running Containers


## Exercise 5: Building Containers


## Exercise 6: Package Layering


## Extra Credit: Switching Your OS

