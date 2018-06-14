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
and uncommenting the `AutomaticUpdatePolicy` and `IdleExitTimeout`
lines.  We want to change the `AutomaticUpdatePolicy` to `check`.

```
# cat /etc/rpm-ostreed.conf
# Entries in this file show the compile time defaults.
# You can change settings by editing this file.
# For option meanings, see rpm-ostreed.conf(5).

[Daemon]
AutomaticUpdatePolicy=check
IdleExitTimeout=60
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
of the upgrades in the background.  This feature is still considered
experimental, so you are accepting some amount of risk by using it.  However,
by enabling the feature on your host and [reporting issues](https://github.com/projectatomic/rpm-ostree/issues/),
you are helping to stabilize this feature for production-grade deployments.

Again, we configure this by changing the `/etc/rpm-ostreed.conf` file.  This
time we configure the `AutomaticUpdatePolicy` to a new value of `ex-stage`.
Additionally, we populate the config file with a new section called
`[Experimental]` where we can enable the `StageDeployments` setting.  And
finally, we reload the daemon to make the setting take effect.  After doing
so, the output from `rpm-ostree status` will reflect the new policy.

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

