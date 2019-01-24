---
layout: post
title: Fedora Silverblue Day 1
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

**NOTE:** This is not intended as a comprehensive guide to every tool and operation
possible on Silverblue, but should be a starting point on how to start adapting your
existing workflows to the ways of a container host.

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

**DEVCONF NOTE:**
If you are following this guide at DevConf 2019, you'll need to re-configure your
remote to point to the local mirror, in order to not overwhelm the conference WiFi.
If you are following these instructions elsewhere, you can skip this reconfiguration of
the remote.

To reconfigure the remote, edit `/etc/ostree/remotes.d/fedora-workstation.conf`
and update the `url` field to the value provided during the DevConf session.  Additionally,
change the value of `gpg-verify` to `false`.  You can reset these values to their defaults
after the DevConf session is over.
**END OF DEVCONF NOTE**

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
`idle`. And there is a `Transaction` field now that shows the upgrade operation that is occurring.

```
$ rpm-ostree status
State: busy
AutomaticUpdates: stage; rpm-ostreed-automatic.service: running
Transaction: upgrade
Deployments:
● ostree://fedora-workstation:fedora/29/x86_64/silverblue
                   Version: 29.1.2 (2018-10-24 23:20:30)
                    Commit: f17b670fa8cf69144be5ae0c968dc2ee7eb6999a5f7a54f1ee71eec7783e434a
              GPGSignature: Valid signature by 5A03B4DD8254ECA02FDA1637A20AA56B429476B4
```

The upgrade will run in the background and may take some time, so let it run for a
few minutes and periodically check `rpm-ostree status` to see if it has completed.
When it has completed, you'll see two deployments listed - your current deployment
that you are booted into and the upgraded deployment that is ready to be rebooted
into.

```
$ rpm-ostree status
State: idle
AutomaticUpdates: stage; rpm-ostreed-automatic.timer: last run 10s ago
Deployments:
  ostree://fedora-workstation:fedora/29/x86_64/silverblue
                   Version: 29.20190119.0 (2019-01-19 00:53:06)
                    Commit: f027d3d70a4da161200382ad85c16ff1b6b5c4c05d357b962ed10fda6f2dc395
              GPGSignature: Valid signature by 5A03B4DD8254ECA02FDA1637A20AA56B429476B4
                      Diff: 389 upgraded, 7 removed, 39 added

● ostree://fedora-workstation:fedora/29/x86_64/silverblue
                   Version: 29.1.2 (2018-10-24 23:20:30)
                    Commit: f17b670fa8cf69144be5ae0c968dc2ee7eb6999a5f7a54f1ee71eec7783e434a
              GPGSignature: Valid signature by 5A03B4DD8254ECA02FDA1637A20AA56B429476B4
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

One of the things you will notice about the set of applications provided
in Silverblue is the absence of some of the default GNOME applications,
which is a notable change from Fedora Workstation.  This choice was made
to slim down the install size of Silverblue because we can get many (if
not all) of our GNOME applications from Flatpaks.

The most popular Flatpak repo at the moment is [Flathub](https://flathub.org).
It houses a wide variety of free and non-free applications.  However, it is
not installed by default as part of Silverblue.  So the first thing to do in
order to start using the available Flatpaks is to enable the Flathub repo.

**NOTE:** Typically you should be able to manage your Flatpak repos and applications
using GNOME Software, but you may encounter errors when trying to use GNOME Software
and Flatpaks on a fresh install of Silverblue.  That experience will be improved
in future releases.  For now, we will use the command line to manage the Flatpaks
and Flatpak repos.

You can browse to the Flathub setup page for Fedora ([https://flatpak.org/setup/Fedora/](https://flatpak.org/setup/Fedora/))
and copy the link for the "Flathub repository file". We install the repo
on the CLI like so:

`$ sudo flatpak --system remote-add flathub https://flathub.org/repo/flathub.flatpakrepo`

This installs the Flathub repo for the entire system.

Now we can install an application from Flathub.  Using your web browser, browse
the available applications on Flathub and pick one to install.  As before, copy
the link from the "Install" button and we'll install it from the command line.
Below, you can see what it looks like to install GIMP:

```
$ sudo flatpak install https://flathub.org/repo/appstream/org.gimp.GIMP.flatpakref
Required runtime for org.gimp.GIMP/x86_64/stable (runtime/org.gnome.Platform/x86_64/3.28) found in remote flathub
Do you want to install it? [y/n]: y
Installing in system:
org.gnome.Platform/x86_64/3.28             flathub 6d1d0ebbd724
org.freedesktop.Platform.ffmpeg/x86_64/1.6 flathub d757f762489e
org.gnome.Platform.Locale/x86_64/3.28      flathub 2823e3d81b74
org.gimp.GIMP/x86_64/stable                flathub 1c130cbfaec9
  permissions: ipc, network, x11
  file access: /tmp, host, xdg-config/GIMP, xdg-config/gtk-3.0
  dbus access: org.freedesktop.FileManager1, org.gtk.vfs, org.gtk.vfs.*
  tags: stable
Is this ok [y/n]: y
Installing: org.gnome.Platform/x86_64/3.28 from flathub
[####################] 975 metadata, 20492 content objects fetched; 292154 KiB transferred in 94 seconds
Now at 6d1d0ebbd724.
Installing: org.freedesktop.Platform.ffmpeg/x86_64/1.6 from flathub
[####################] 8 metadata, 12 content objects fetched; 2788 KiB transferred in 1 seconds
Now at d757f762489e.
Installing: org.gnome.Platform.Locale/x86_64/3.28 from flathub
[####################] 4 metadata, 1 content objects fetched; 14 KiB transferred in 0 seconds
Now at 2823e3d81b74.
Installing: org.gimp.GIMP/x86_64/stable from flathub
[####################] 511 metadata, 4671 content objects fetched; 75918 KiB transferred in 24 seconds
Now at 1c130cbfaec9.
```
As you can see, the `flatpak` command automatically pulled the required runtimes
for GIMP and has installed them (and the app) for all users on the system.

Now if you examine the installed applications via GNOME, you'll see (in this case)
GIMP installed and able to be run.

Congratulations!  You've installed your first Flatpak!

## Exercise 4: Running Containers

Containers are the logical means of running software that is not installed as part of
the base OS.  In case you didn't realize it, running your Flatpak'ed application was
one of the ways to run a container.  While Flatpaks are nice for applications with
graphical interfaces, they can be a bit heavyweight for running a applicaition that
typically runs on the command line or as a daemon-like process.

The default way of running and managing containers in Fedora Silverblue is through
the command line utility called `podman`.  If you have ever used the `docker` command
line, using `podman` will be very familiar to you.

To start, we will pull a container image from the Fedora registry and then run it,
all using `podman`.  We'll also display the contents of `/etc/os-release` on the host
and in the container, to demonstrate we are actually in a container.

```
$ cat /etc/os-release | grep VERSION
VERSION="29.20190119.0 (Workstation Edition)"
VERSION_ID=29
VERSION_CODENAME=""
REDHAT_BUGZILLA_PRODUCT_VERSION=29
REDHAT_SUPPORT_PRODUCT_VERSION=29
OSTREE_VERSION=29.20190119.0
$ sudo podman pull registry.fedoraproject.org/fedora:28
Trying to pull registry.fedoraproject.org/fedora:28...Getting image source signatures
Copying blob e69e955c514f: 85.99 MiB / 85.99 MiB [==========================] 8s
Copying config ded494ce3076: 1.27 KiB / 1.27 KiB [==========================] 0s
Writing manifest to image destination
Storing signatures
ded494ce3076e8f2d264235fdb09da5970921d8317f8fd024ab65821bf13e29f
[miabbott@localhost ~]$ sudo podman run -it registry.fedoraproject.org/fedora:28
[root@c4f481a4a4ed /]# cat /etc/os-release | grep VERSION
VERSION="28 (Twenty Eight)"
VERSION_ID=28
REDHAT_BUGZILLA_PRODUCT_VERSION=28
REDHAT_SUPPORT_PRODUCT_VERSION=28

```

And now you have run a container on the command line!

If you wanted to delete the container you just run, you would use `podman rm`
and deleting the container image would be `podman rmi`.

```
$ sudo podman ps -a
CONTAINER ID  IMAGE                                 COMMAND    CREATED            STATUS                         PORTS  NAMES
c4f481a4a4ed  registry.fedoraproject.org/fedora:28  /bin/bash  About an hour ago  Exited (0) About a minute ago         practical_lewin
 sudo podman rm c4f481a4a4ed
c4f481a4a4ed973b19141a9adcc463dcb8e171444d0f927bdf6a6ddf2495aa27
$ sudo podman rm c4f481a4a4ed
c4f481a4a4ed973b19141a9adcc463dcb8e171444d0f927bdf6a6ddf2495aa27
$ sudo podman images -a
REPOSITORY                          TAG   IMAGE ID       CREATED        SIZE
registry.fedoraproject.org/fedora   28    ded494ce3076   3 months ago   264 MB
$ sudo podman rmi ded494ce3076
ded494ce3076e8f2d264235fdb09da5970921d8317f8fd024ab65821bf13e29f
```

Running a container from a base image is not terribly exciting.  In
the next section, we will build our own container from scratch
and run it.

## Exercise 5: Building Containers

While in the `docker` ecosystem, one would use `docker build` or
`docker compose` for building container image.  But in Fedora Silverblue,
we use `buildah` to build our container images.  Since `buildah` supports
building container images from Dockerfiles, we are able to reuse any
existing Dockerfiles to build images on Silverblue.

For this exercise, we'll create a small Dockerfile that installs `strace`
which is not installed as part of the Silverblue OS.  Create a Dockerfile
that looks like this:

```
$ cat Dockerfile
FROM registry.fedoraproject.org/fedora:29
RUN dnf -y install strace && \
    dnf clean all
```

And we can build the container image, using the `buildah bud` command. The
`-t` flag is the way to "tag" your container image.  You can use any string
you choose for your image.

```
$ sudo buildah bud -t miabbott/strace .
[sudo] password for miabbott:
STEP 1: FROM registry.fedoraproject.org/fedora:29
STEP 2: RUN dnf -y install strace &&     dnf clean all
Fedora Modular 29 - x86_64                                                                                                                                                         507 kB/s | 1.5 MB     00:03
Fedora Modular 29 - x86_64 - Updates                                                                                                                                               1.6 MB/s | 2.0 MB     00:01
Fedora 29 - x86_64 - Updates                                                                                                                                                       5.4 MB/s |  20 MB     00:03
Fedora 29 - x86_64                                                                                                                                                                 6.9 MB/s |  62 MB     00:09
Dependencies resolved.
===================================================================================================================================================================================================================
 Package                                          Arch                                             Version                                                 Repository                                         Size
===================================================================================================================================================================================================================
Installing:
 strace                                           x86_64                                           4.26-1.fc29                                             updates                                           956 k

Transaction Summary
===================================================================================================================================================================================================================
Install  1 Package

Total download size: 956 k
Installed size: 2.2 M
Downloading Packages:
strace-4.26-1.fc29.x86_64.rpm                                                                                                                                                       10 MB/s | 956 kB     00:00
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                                                              743 kB/s | 956 kB     00:01
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                                                                           1/1
  Installing       : strace-4.26-1.fc29.x86_64                                                                                                                                                                 1/1
  Running scriptlet: strace-4.26-1.fc29.x86_64                                                                                                                                                                 1/1
  Verifying        : strace-4.26-1.fc29.x86_64                                                                                                                                                                 1/1

Installed:
  strace-4.26-1.fc29.x86_64

Complete!
33 files removed
STEP 3: COMMIT containers-storage:[overlay@/var/lib/containers/storage+/var/run/containers/storage:overlay.mountopt=nodev,overlay.override_kernel_check=true]localhost/miabbott/strace:latest
Getting image source signatures
Skipping fetch of repeat blob sha256:4fbdefa47448c4641a7da56cad347c5efacf26d52f4e5d14703658d4b50ed0c2
Copying blob sha256:6315e0eaa586d98d8e54b5095e5c83f23f41f9645e469202c203b70fb30d62a0
 3.34 MiB / 3.34 MiB [======================================================] 0s
Copying config sha256:6a05d9c836c83d1e16c89570e4782ad5b09266f6e1cbaf4eb3cc4ce4acb3737c
 621 B / 621 B [============================================================] 0s
Writing manifest to image destination
Storing signatures
--> 6a05d9c836c83d1e16c89570e4782ad5b09266f6e1cbaf4eb3cc4ce4acb3737c
$ sudo buildah images
IMAGE NAME                                               IMAGE TAG            IMAGE ID             CREATED AT             SIZE
registry.fedoraproject.org/fedora                        29                   69c5db8b64a7         Jan 9, 2019 01:48      283 MB
localhost/miabbott/strace                                latest               6a05d9c836c8         Jan 24, 2019 08:31     298 MB
$ sudo podman images
REPOSITORY                          TAG      IMAGE ID       CREATED       SIZE
localhost/miabbott/strace           latest   6a05d9c836c8   7 hours ago   298 MB
registry.fedoraproject.org/fedora   29       69c5db8b64a7   2 weeks ago   283 MB
```

As you can see above, the newly built container image lives in a shared container storage
space that is viewable by both `podman` and `buildah`.  Now we can run the container and
specify the `strace` command directly.  You can see that if you try to run `strace` directly
on the host, it does not exist.

We have to use `localhost` as the value for the registry as the image only lives on our
host.

```
$ sudo podman run -it localhost/miabbott/strace strace
strace: must have PROG [ARGS] or -p PID
Try 'strace -h' for more information.
$ strace
-bash: strace: command not found
```

With a few extra flags, we can trace a process on the host.  This requires the use of
the `--privileged` flag, so we have the necessary kernel capabilities to attach to
a running process.  Additionally, we need to pass in the PID namespace of the host
so that we can see the process we want to trace inside the container.

We'll start up the `rpm-ostree` daemon with `rpm-ostree status`, capture the PID of the
process and use `strace` to get a trace of the `rpm-ostree` PID inside the container.

```
$ rpm-ostree status > /dev/null && pidof rpm-ostree
1649
$ sudo podman run -it --privileged --pid=host localhost/miabbott/strace strace -p 1649
strace: Process 1649 attached
restart_syscall(<... resuming interrupted poll ...>) = 0
socket(AF_UNIX, SOCK_DGRAM|SOCK_CLOEXEC, 0) = 16
getsockopt(16, SOL_SOCKET, SO_SNDBUF, [212992], [4]) = 0
setsockopt(16, SOL_SOCKET, SO_SNDBUF, [8388608], 4) = 0
getuid()                                = 0
geteuid()                               = 0
getgid()                                = 0
getegid()                               = 0
sendmsg(16, {msg_name={sa_family=AF_UNIX, sun_path="/run/systemd/notify"}, msg_namelen=21, msg_iov=[{iov_base="STATUS=clients=0; idle exit in 5"..., iov_len=41}], msg_iovlen=1, msg_controllen=0, msg_flags=0}, M$
G_NOSIGNAL) = 41
close(16)                               = 0
poll([{fd=4, events=POLLIN}], 1, 982^Cstrace: Process 1649 detached
 <detached ...>
```

Hopefully, this demonstrates for you how you can containerize applications in a
container and use them as part of Silverblue.

But what if your application isn't well suited for containerization, something like
a host extension?  This is where package layering comes in.

## Exercise 6: Package Layering

The paradigm for Silverblue is to use containers whenever possible.  But sometimes
this just isn't feasible or practical.  In these relatively rare cases, we can
still use the application via package layering.

Package layering is roughly the equivalent of `dnf install` on a traditional Fedora
Workstation host.  The key difference is that the package being installed on Silverblue
is installed into a new deployment and requires a reboot to be able to use the
application.  The benefit to this is that your running host is safely protected from
any kind of malicious behavior that an RPM might have as part of it's install operation.

For this exercise, we are going to re-use the `strace` package as an example.  It doesn't
really fit the use case for package layering, since it is relatively easy to run within
a container, but should illustrate the basics of package layering.

The entrypoint for package layering operations is `rpm-ostree install`.  This command has
the ability to query the DNF repos in Fedora for the package that was requested, then
download and install the package in the separate deployment. Additionally, you can
pass a URL to the `install` operation or even download the RPM local to your host and
provide that as an argument.

We start with the two deployments from the previous exercise: the original deployment
from the Silverblue install and the upgraded deployment from Exercise 1.  We'll do
`rpm-ostree install strace` and see what we end up with.

```
$ rpm-ostree status
State: idle
AutomaticUpdates: stage; rpm-ostreed-automatic.timer: no runs since boot
Deployments:
● ostree://fedora-workstation:fedora/29/x86_64/silverblue
                   Version: 29.20190119.0 (2019-01-19T00:53:06Z)
                    Commit: f027d3d70a4da161200382ad85c16ff1b6b5c4c05d357b962ed10fda6f2dc395
              GPGSignature: Valid signature by 5A03B4DD8254ECA02FDA1637A20AA56B429476B4

  ostree://fedora-workstation:fedora/29/x86_64/silverblue
                   Version: 29.1.2 (2018-10-24T23:20:30Z)
                    Commit: f17b670fa8cf69144be5ae0c968dc2ee7eb6999a5f7a54f1ee71eec7783e434a
              GPGSignature: Valid signature by 5A03B4DD8254ECA02FDA1637A20AA56B429476B4

$ sudo rpm-ostree install strace
Checking out tree f027d3d... done
Enabled rpm-md repositories: updates fedora
Updating metadata for 'updates'... done
rpm-md repo 'updates'; generated: 2019-01-24T03:12:25Z
Updating metadata for 'fedora'... done
rpm-md repo 'fedora'; generated: 2018-10-24T22:20:15Z
Importing rpm-md... done
Resolving dependencies... done
Will download: 1 package (979.2 kB)
Downloading from 'updates'... done
Importing packages... done
Checking out packages... done
Running pre scripts... done
Running post scripts... done
Writing rpmdb... done
Writing OSTree commit... done
Staging deployment... done
Added:
  strace-4.26-1.fc29.x86_64
Run "systemctl reboot" to start a reboot

$ rpm-ostree status
State: idle
AutomaticUpdates: stage; rpm-ostreed-automatic.timer: no runs since boot
Deployments:
  ostree://fedora-workstation:fedora/29/x86_64/silverblue
                   Version: 29.20190119.0 (2019-01-19T00:53:06Z)
                BaseCommit: f027d3d70a4da161200382ad85c16ff1b6b5c4c05d357b962ed10fda6f2dc395
              GPGSignature: Valid signature by 5A03B4DD8254ECA02FDA1637A20AA56B429476B4
           LayeredPackages: strace

● ostree://fedora-workstation:fedora/29/x86_64/silverblue
                   Version: 29.20190119.0 (2019-01-19T00:53:06Z)
                    Commit: f027d3d70a4da161200382ad85c16ff1b6b5c4c05d357b962ed10fda6f2dc395
              GPGSignature: Valid signature by 5A03B4DD8254ECA02FDA1637A20AA56B429476B4

  ostree://fedora-workstation:fedora/29/x86_64/silverblue
                   Version: 29.1.2 (2018-10-24T23:20:30Z)
                    Commit: f17b670fa8cf69144be5ae0c968dc2ee7eb6999a5f7a54f1ee71eec7783e434a
              GPGSignature: Valid signature by 5A03B4DD8254ECA02FDA1637A20AA56B429476B4
$ strace
-bash: strace: command not found
```

Now we have three deployments:  the original, the upgraded, and the upgraded with `strace`
layered on top.  Notice that the `rpm-ostree install` operation printed the differences
between the upgraded deployment and the new deployment in it's output.  This means that
the only change that was made was the addition of that `strace` package.  But if you try
to run `strace`, it is still not available.  So we need to reboot the host to activate
the new deployment and make `strace` available.

```
$ sudo systemctl reboot
Connection to 192.168.124.123 closed by remote host.
Connection to 192.168.124.123 closed.
$ ssh -l miabbott 192.168.124.123
Warning: Permanently added '192.168.124.123' (ECDSA) to the list of known hosts.
miabbott@192.168.124.123's password:
Last login: Thu Jan 24 15:20:39 2019 from 192.168.124.1

$ rpm-ostree status
State: idle
AutomaticUpdates: stage; rpm-ostreed-automatic.timer: no runs since boot
Deployments:
● ostree://fedora-workstation:fedora/29/x86_64/silverblue
                   Version: 29.20190119.0 (2019-01-19T00:53:06Z)
                BaseCommit: f027d3d70a4da161200382ad85c16ff1b6b5c4c05d357b962ed10fda6f2dc395
              GPGSignature: Valid signature by 5A03B4DD8254ECA02FDA1637A20AA56B429476B4
           LayeredPackages: strace

  ostree://fedora-workstation:fedora/29/x86_64/silverblue
                   Version: 29.20190119.0 (2019-01-19T00:53:06Z)
                    Commit: f027d3d70a4da161200382ad85c16ff1b6b5c4c05d357b962ed10fda6f2dc395
              GPGSignature: Valid signature by 5A03B4DD8254ECA02FDA1637A20AA56B429476B4
$ strace
strace: must have PROG [ARGS] or -p PID
Try 'strace -h' for more information.
```

After the reboot one of the deployments was pruned and we are left with the new deployment
with the package layered and the upgraded deployment with no packages.  If we decided we
didn't need the `strace` package on the host, we could use `rpm-ostree rollback` to
roll the host back to the upgraded deployment.  Or we could use `rpm-ostree uninstall`
to remove the package layer from the deployment.  The end result is roughly the same:
you end up with the `strace` package being unavailable on the resulting deployment (after
a reboot, of course).

There are additional package layering operations that you can do, such as replacing a
package in the base set or removing packages from the base set.  But these operations
will not be as commonly used as simple `rpm-ostree install/uninstall`.

## Conclusion

This guide was intended to give you an idea of some of the tasks you would be doing
soon after installing Silverblue on the first day.  Of course, there are many
other things can be done with containers, Flatpaks, and the `ostree/rpm-ostree` stack;
you are encouraged to read (and contribute to!) the [Silverblue documenation](https://docs.fedoraproject.org/en-US/fedora-silverblue/),
the upstream documentation for the various projects and come ask questions in
the [Silverblue Discourse site](https://discussion.fedoraproject.org/c/desktop/silverblue).
