---
layout: post
title: Using Automatic Updates on Fedora Silverblue
---

One of the [oldest issues](https://github.com/projectatomic/rpm-ostree/issues/247)
and most desired pieces of functionality in the `rpm-ostree` stack has been
automatic updates.  The ability to have your host download and stage the
upgrade in the background would eliminate some of the system admin headache
that comes when maintaining your host.  (Furthermore, enabling this ability
in a cluster of hosts allows the cluster admin to worry about other things
than scheduling downtime to apply upgrades.)

Earlier this year, the `rpm-ostree` developers made the first step towards the
ultimate goal of automated updates, when the ability to [check for updates](https://github.com/projectatomic/rpm-ostree/pull/1147)
in the background landed in the codebase.  The background checking and the
[nicely formatted status](https://github.com/projectatomic/rpm-ostree/pull/1147#issuecomment-355614658)
was a welcome addition to the experience of administering a host with `rpm-ostree`.

In version [v2018.5](https://github.com/projectatomic/rpm-ostree/releases/tag/v2018.5)
of `rpm-ostree`, the next piece of the puzzle landed.  `rpm-ostree` gained the
ability to [download and stage the upgrade in the background](https://github.com/projectatomic/rpm-ostree/pull/1321).
Not only did this further enable automatic updates, but it also allowed the
developers to address another [long standing issue](https://github.com/projectatomic/rpm-ostree/issues/40)
about performing the `/etc/` merge just before reboot, rather than at the time
of the upgrade.

In this post, I'll cover how to enable the various automatic update policies
that are now available in the `rpm-ostree` stack.

## Automatic Update Checking

As mentioned earlier, the first step towards complete automatic updates was
enabling the checking of updates in the background.  This is done using the
`/etc/rpm-ostreed.conf` config file, a systemd timer, and systemd service.

Below you can see the output from `rpm-ostree status` before any of the
update policies have been configured. The `State:` field describes that
the `rpm-ostreed` daemon is idle and automatic updates are disabled.

```
# rpm-ostree status
State: idle; auto updates disabled
Deployments:
● ostree://fedora-workstation:fedora/28/x86_64/workstation
                   Version: 28.20180601.0 (2018-06-01 11:53:40)
                    Commit: 6067049927936b5bf41695263c43f37238603b858506a45c98316501c94841b0
              GPGSignature: Valid signature by 128CF232A9371991C8A65695E08E7E629DB62FB1
```

We can turn on the update checking by editing `/etc/rpm-ostreed.conf`
and uncommenting the `AutomaticUpdatePolicy`.  We want to change the
`AutomaticUpdatePolicy` to `check`.  You can leave the `IdleExitTimeout`
line commented; from `man rpm-ostreed.conf` - "Controls the time in
seconds of inactivity before the daemon exits".

```
# cat /etc/rpm-ostreed.conf
# Entries in this file show the compile time defaults.
# You can change settings by editing this file.
# For option meanings, see rpm-ostreed.conf(5).

[Daemon]
AutomaticUpdatePolicy=check
#IdleExitTimeout=60
```

After configuring the policy, we need to reload the `rpm-ostreed` service
and enable the systemd timer.  Then we can check the output of
`rpm-ostree status` to confirm that the policy is in place and the timer
is active.

```
# systemctl reload rpm-ostreed
# systemctl enable rpm-ostreed-automatic.timer --now
Created symlink /etc/systemd/system/timers.target.wants/rpm-ostreed-automatic.timer → /usr/lib/systemd/system/rpm-ostreed-automatic.timer.
# rpm-ostree status
State: idle; auto updates enabled (check; last run 195ms ago)
Deployments:
● ostree://fedora-workstation:fedora/28/x86_64/workstation
                   Version: 28.20180601.0 (2018-06-01 11:53:40)
                    Commit: 6067049927936b5bf41695263c43f37238603b858506a45c98316501c94841b0
              GPGSignature: Valid signature by 128CF232A9371991C8A65695E08E7E629DB62FB1

Available update:
        Version: 28.20180613.0 (2018-06-13 14:05:43)
         Commit: 106280bdaf280e5c1a2e803f9bb58ce58ab3043ebc69de12c39e030218da2da6
   GPGSignature: Valid signature by 128CF232A9371991C8A65695E08E7E629DB62FB1
           Diff: 79 upgraded, 1 removed, 1 added
```

You can see in the output that the timer was activated and the corresponding
service was run `195ms` ago.  The service found an update was available and
the output from `rpm-ostree status` nicely displays information about the
available upgrade.  A great feature of this new status is that it will
display any CVEs that are addressed by an available update; in this case
there are no CVEs addressed, though.  Having this kind of information
available in the status output allows admins to make better decisions about
taking the downtime hit to reboot a host to apply an upgrade.

## Automatic Downloading and Staging of Updates

Now let's take the next steps and enable the automatic download and staging
of the upgrades in the background.  It's worth discussing the new options
a bit before we start enabling them.

`rpm-ostree` is now able to 'stage' deployments/upgrades.  This allows the host
to perform the `/etc/` merge right before the reboot is completed.  It is possible
to enable this 'staging' for both automatic updates and manual updates.  If the
'staging' feature is not used, the `/etc` merge happens during the upgrade operation
as normal.

We are able to configure the `AutomaticUpdatePolicy` to a new value: `ex-stage`.
This instructs the daemon to automatically download the upgrade and 'stage' it
for deployment, allowing the `/etc` merge to happen before a reboot is completed.
This means the 'staging' feature is only applicable to the automatic updates;
manual updates will perform the `/etc/` merge during the upgrade operation.

If we want to configure the 'staging' feature to be used for *all* updates (automatic
and manual), we can add a new section in `/etc/rpm-ostreed.conf`.

This means we can choose how to use the 'staging' feature on our host.

**!!! WARNING !!! These features are still considered experimental, so you are
accepting some amount of risk by using them.  !!! WARNING !!!**

However, by enabling the feature on your host and [reporting issues](https://github.com/projectatomic/rpm-ostree/issues/),
you are helping to stabilize this feature for production-grade deployments
and actively participating in open source development.

We'll first configure the new `ex-stage` automatic update policy by changing the
value in the `/etc/rpm-ostreed.conf` file.

Additionally, I'm interested in using the 'staging' feature for manual upgrades as
well, so we populate the config file with a new section called
`[Experimental]` where we can enable the `StageDeployments` setting.  This is
what configures the daemon to use the 'staging' feature for all upgrade operations.

To complete the configuration, we reload the daemon and can inspect the output
of `rpm-ostree status` to confirm the settings have taken effect.  The status
output now shows that 'auto updates enabled' and the 'ex-stage' policy is
being used.

```
# cat /etc/rpm-ostreed.conf
# Entries in this file show the compile time defaults.
# You can change settings by editing this file.
# For option meanings, see rpm-ostreed.conf(5).

[Daemon]
AutomaticUpdatePolicy=ex-stage
IdleExitTimeout=60

[Experimental]
StageDeployments=yes
# systemctl reload rpm-ostreed.service
# rpm-ostree status
State: idle; auto updates enabled (ex-stage; no runs since boot)
Deployments:
● ostree://fedora-workstation:fedora/28/x86_64/workstation
                   Version: 28.20180601.0 (2018-06-01 11:53:40)
                    Commit: 6067049927936b5bf41695263c43f37238603b858506a45c98316501c94841b0
              GPGSignature: Valid signature by 128CF232A9371991C8A65695E08E7E629DB62FB1

Available update:
        Version: 28.20180613.0 (2018-06-13 14:05:43)
         Commit: 106280bdaf280e5c1a2e803f9bb58ce58ab3043ebc69de12c39e030218da2da6
   GPGSignature: Valid signature by 128CF232A9371991C8A65695E08E7E629DB62FB1
           Diff: 79 upgraded, 1 removed, 1 added
```

If you are like me, you don't want to wait for the timer to fire off to see
the automatic update in action.  So you can manually start the service and see
how the output of `rpm-ostree status` changes when it is running.

You should recognize that the `State:` line has changed to show that the daemon
is busy and the automatic updates are running.  Additonally, the `Transaction:`
line shows that an upgrade is happening.

```
# systemctl start rpm-ostreed-automatic.service
# rpm-ostree status
State: busy; auto updates enabled (ex-stage; running)
Transaction: upgrade
...
```

When the upgrade has completed, you'll see that the new deployment is staged
first in the list of deployments.  And the original, booted deployment is left
untouched as always.  Now you are able to decide if you want to boot into that
new deployment or wait for the next upgrade.

```
# rpm-ostree status
State: idle; auto updates enabled (ex-stage; last run 58min ago)
Deployments:
  ostree://fedora-workstation:fedora/28/x86_64/workstation
                   Version: 28.20180613.0 (2018-06-13 14:05:43)
                    Commit: 106280bdaf280e5c1a2e803f9bb58ce58ab3043ebc69de12c39e030218da2da6
              GPGSignature: Valid signature by 128CF232A9371991C8A65695E08E7E629DB62FB1
                      Diff: 79 upgraded, 1 removed, 1 added

● ostree://fedora-workstation:fedora/28/x86_64/workstation
                   Version: 28.20180601.0 (2018-06-01 11:53:40)
                    Commit: 6067049927936b5bf41695263c43f37238603b858506a45c98316501c94841b0
              GPGSignature: Valid signature by 128CF232A9371991C8A65695E08E7E629DB62FB1
```

And if you want all the gory details of the upgrade, we can pass the `-v` flag to `rpm-ostree
status` and see the packages that have changed.  This also allows us to see the `Staged` field,
which indicate the pending deployment is using the 'staging' feature that was discusssed earlier.

```
$ rpm-ostree status -v
State: idle; auto updates enabled (ex-stage; last run 1min 13s ago)
Deployments:
  ostree://fedora-workstation:fedora/28/x86_64/workstation
                   Version: 28.20180613.0 (2018-06-13 14:05:43)
                    Commit: 106280bdaf280e5c1a2e803f9bb58ce58ab3043ebc69de12c39e030218da2da6
                            └─ repo-0 (2018-06-13 13:41:56)
                            └─ repo-1 (2018-04-26 11:12:28)
                    Staged: yes
                 StateRoot: fedora-workstation
              GPGSignature: 1 signature
                            Signature made Wed 13 Jun 2018 10:05:51 AM EDT using RSA key ID E08E7E629DB62FB1
                            Good signature from "Fedora 28 <fedora-28@fedoraproject.org>"
                  Upgraded: baobab 3.28.0-1.fc28 -> 3.28.0-2.fc28
                            bluez 5.49-3.fc28 -> 5.50-1.fc28
                            bluez-cups 5.49-3.fc28 -> 5.50-1.fc28
                            bluez-libs 5.49-3.fc28 -> 5.50-1.fc28
                            bluez-obexd 5.49-3.fc28 -> 5.50-1.fc28
                            bolt 0.3-1.fc28 -> 0.4-1.fc28
                            checkpolicy 2.7-7.fc28 -> 2.8-1.fc28
                            container-selinux 2:2.61-1.git9b55129.fc28 -> 2:2.63-1.git284f9e7.fc28
                            coreutils 8.29-6.fc28 -> 8.29-7.fc28
                            coreutils-common 8.29-6.fc28 -> 8.29-7.fc28
                            curl 7.59.0-3.fc28 -> 7.59.0-4.fc28
                            docker 2:1.13.1-56.git6c336e4.fc28 -> 2:1.13.1-59.gitaf6b32b.fc28
                            docker-common 2:1.13.1-56.git6c336e4.fc28 -> 2:1.13.1-59.gitaf6b32b.fc28
                            docker-rhel-push-plugin 2:1.13.1-56.git6c336e4.fc28 -> 2:1.13.1-59.gitaf6b32b.fc28
                            edk2-ovmf 20171011git92d07e4-7.fc28 -> 20180529gitee3198e672e2-1.fc28
                            elfutils-default-yama-scope 0.170-11.fc28 -> 0.171-1.fc28
                            elfutils-libelf 0.170-11.fc28 -> 0.171-1.fc28
                            elfutils-libs 0.170-11.fc28 -> 0.171-1.fc28
                            emacs-filesystem 1:25.3-5.fc28 -> 1:26.1-1.fc28
                            firefox 60.0.1-5.fc28 -> 60.0.1-6.fc28
                            foomatic-db 4.0-57.20180102.fc28 -> 4.0-58.20180102.fc28
                            foomatic-db-filesystem 4.0-57.20180102.fc28 -> 4.0-58.20180102.fc28
                            foomatic-db-ppds 4.0-57.20180102.fc28 -> 4.0-58.20180102.fc28
                            geoclue2 2.4.8-1.fc28 -> 2.4.10-2.fc28
                            geoclue2-libs 2.4.8-1.fc28 -> 2.4.10-2.fc28
                            gnome-boxes 3.28.4-1.fc28 -> 3.28.5-1.fc28
                            gnome-disk-utility 3.28.2-1.fc28 -> 3.28.3-1.fc28
                            gnome-terminal 3.28.2-1.fc28 -> 3.28.2-2.fc28
                            gnutls 3.6.2-1.fc28 -> 3.6.2-3.fc28
                            ibus-typing-booster 1.5.38-2.fc28 -> 2.0.0-1.fc28
                            ipcalc 0.2.2-4.fc28 -> 0.2.3-1.fc28
                            jasper-libs 2.0.14-3.fc28 -> 2.0.14-5.fc28
                            kernel 4.16.12-300.fc28 -> 4.16.14-300.fc28
                            kernel-core 4.16.12-300.fc28 -> 4.16.14-300.fc28
                            kernel-modules 4.16.12-300.fc28 -> 4.16.14-300.fc28
                            kernel-modules-extra 4.16.12-300.fc28 -> 4.16.14-300.fc28
                            krb5-libs 1.16.1-2.fc28 -> 1.16.1-4.fc28
                            libappstream-glib 0.7.8-1.fc28 -> 0.7.9-1.fc28
                            libcurl 7.59.0-3.fc28 -> 7.59.0-4.fc28
                            libinput 1.10.7-1.fc28 -> 1.11.0-1.fc28
                            libselinux 2.7-13.fc28 -> 2.8-1.fc28
                            libselinux-utils 2.7-13.fc28 -> 2.8-1.fc28
                            libsemanage 2.7-12.fc28 -> 2.8-1.fc28
                            libsepol 2.7-6.fc28 -> 2.8-1.fc28
                            libtiff 4.0.9-8.fc28 -> 4.0.9-9.fc28
                            libunistring 0.9.9-1.fc28 -> 0.9.10-1.fc28
                            opus 1.3-0.4.beta.fc28 -> 1.3-0.5.rc.fc28
                            osinfo-db 20180514-1.fc28 -> 20180531-1.fc28
                            p11-kit 0.23.10-1.fc28 -> 0.23.12-1.fc28
                            p11-kit-trust 0.23.10-1.fc28 -> 0.23.12-1.fc28
                            perl-Storable 1:3.09-1.fc28 -> 1:3.11-2.fc28
                            plymouth 0.9.3-6.fc28 -> 0.9.3-9.fc28
                            plymouth-core-libs 0.9.3-6.fc28 -> 0.9.3-9.fc28
                            plymouth-graphics-libs 0.9.3-6.fc28 -> 0.9.3-9.fc28
                            plymouth-plugin-label 0.9.3-6.fc28 -> 0.9.3-9.fc28
                            plymouth-plugin-two-step 0.9.3-6.fc28 -> 0.9.3-9.fc28
                            plymouth-scripts 0.9.3-6.fc28 -> 0.9.3-9.fc28
                            plymouth-system-theme 0.9.3-6.fc28 -> 0.9.3-9.fc28
                            plymouth-theme-charge 0.9.3-6.fc28 -> 0.9.3-9.fc28
                            podman 0.5.4-1.git1f2e2a2.fc28 -> 0.6.1-1.git3e0ff12.fc28
                            policycoreutils 2.7-18.fc28 -> 2.8-1.fc28
                            policycoreutils-python-utils 2.7-18.fc28 -> 2.8-1.fc28
                            poppler 0.62.0-1.fc28 -> 0.62.0-2.fc28
                            poppler-glib 0.62.0-1.fc28 -> 0.62.0-2.fc28
                            poppler-utils 0.62.0-1.fc28 -> 0.62.0-2.fc28
                            python3-gobject 3.28.2-1.fc28 -> 3.28.3-1.fc28
                            python3-gobject-base 3.28.2-1.fc28 -> 3.28.3-1.fc28
                            python3-libselinux 2.7-13.fc28 -> 2.8-1.fc28
                            python3-libsemanage 2.7-12.fc28 -> 2.8-1.fc28
                            python3-policycoreutils 2.7-18.fc28 -> 2.8-1.fc28
                            selinux-policy 3.14.1-30.fc28 -> 3.14.1-32.fc28
                            selinux-policy-targeted 3.14.1-30.fc28 -> 3.14.1-32.fc28
                            shadow-utils 2:4.5-9.fc28 -> 2:4.6-1.fc28
                            skopeo 0.1.29-5.git7add6fc.fc28 -> 0.1.31-3.git5c61108.fc28
                            sos 3.5-2.fc28 -> 3.5.1-1.fc28
                            vim-minimal 2:8.1.026-1.fc28 -> 2:8.1.042-1.fc28
                            wireless-tools 1:29-19.1.fc28 -> 1:29-20.fc28
                            xkeyboard-config 2.23.1-1.fc28 -> 2.24-2.fc28
                            xorg-x11-xkb-utils 7.7-23.fc28 -> 7.7-25.fc28
                   Removed: skopeo-containers-0.1.29-5.git7add6fc.fc28.x86_64
                     Added: containers-common-0.1.31-3.git5c61108.fc28.x86_64

● ostree://fedora-workstation:fedora/28/x86_64/workstation
                   Version: 28.20180601.0 (2018-06-01 11:53:40)
                    Commit: 6067049927936b5bf41695263c43f37238603b858506a45c98316501c94841b0
                            └─ repo-0 (2018-06-01 11:32:53)
                            └─ repo-1 (2018-04-26 11:12:28)
                 StateRoot: fedora-workstation
              GPGSignature: 1 signature
                            Signature made Fri 01 Jun 2018 07:53:47 AM EDT using RSA key ID E08E7E629DB62FB1
                            Good signature from "Fedora 28 <fedora-28@fedoraproject.org>"
```

