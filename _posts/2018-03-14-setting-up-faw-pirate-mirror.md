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

## Serving Up the Mirror via HTTP

With the content pulled to my VM, I needed to make it available via HTTP and
since I'm on F27AH, I needed a container run a web server.  Despite having no
experience using it, I decided I would work with the [nginx container](https://hub.docker.com/_/nginx/)
to serve up the mirror content.

After pulling the container, I spent some time learning how to configure nginx
and the right way to invoke the container with the content mounted into it.

My `default.conf` file looked like this:

```
$ cat /etc/nginx/conf.d/default.conf
server {
    listen       80;
    server_name  faw.piratemirror.party;

    root   /usr/share/nginx/html;
    location /repo {
        root /usr/share/nginx/html;
        autoindex on;
    }
}
```

Additionally, I had to setup the SELinux labeling for the mounts to be passed into
the container, so I did:

```
$ sudo chcon -R -h -t container_file_t /etc/nginx/conf.d/
$ sudo chcon -R -h -t container_file_t /var/srv/workstation/repo/
```

Then I was finally able to invoke the container like this:

`$ sudo docker run -v /etc/nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf:ro -v /var/srv/workstation/repo/:/usr/share/nginx/html/repo:ro -d -p 80:80 docker.io/nginx`


This allowed me to access the ostree content from http://faw.piratemirror.party/repo
successfully!

## Securing the Transport Layer with Let's Encrypt!

I was encouraged with my success thusfar and wanted to take the next step of securing
the transport layer via HTTPS.  Of course, I was going to use [Let's Encrypt](https://letsencrypt.org/)
to get my free SSL certificate.  The question was how to do it on an Atomic Host
using containers.

The great folks at the [EFF](https://www.eff.org) have created a project called [certbot](https://certbot.eff.org/)
that automates the process of requesting an SSL cert from Let's Encrypt.  And they even
have a [container](https://hub.docker.com/r/certbot/certbot/) that we can use!

The [certbot container directions](https://certbot.eff.org/docs/install.html?highlight=docker#running-with-docker)
are pretty clear, but I still tried them a few times using the [staging environment](https://letsencrypt.org/docs/staging-environment/)
to make sure I understood how the process would work.

The resulting `docker run` command looked like this:

`$ sudo docker run -it --rm -p 443:443 -p 80:80 --name certbot  -v /etc/letsencrypt:/etc/letsencrypt -v /var/lib/letsencrypt-lib/:/var/lib/letsencrypt docker.io/certbot/certbot certonly`

As before, I also had to set the SELinux label on my mounts to `container_file_t`.

The process was successful and I ended up with the necessary certificates in `/etc/letsencrypt`.

Now I needed to take those certificates and modify the nginx config file to use
them.  I studied the [certbot documentation](https://certbot.eff.org/docs/using.html) to
understand where the certificates were located and how to configure nginx to use them.

The result was a config file that looked like this:

```
$ cat /etc/nginx/conf.d/default.conf
server {
    listen       80;
    server_name  faw.piratemirror.party;
    return 301 https://$host:$uri;
}

server {
    listen      443 ssl;
    server_name faw.piratemirror.party;

    ssl_certificate /etc/letsencrypt/live/faw.piratemirror.party/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/faw.piratemirror.party/privkey.pem;

    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';

    ssl_prefer_server_ciphers on;

    ssl_dhparam /usr/share/nginx/dhparams.pem;

    location / {
        root /usr/share/nginx/html/repo/;
        autoindex on;
    }
    root /usr/share/nginx/html/;
    location /repo {
        autoindex on;
    }

}
```

The first `server` stanza tells ngninx to listen on port 80, but redirect the client
to the HTTPS version of the URI.  The second `server` stanza tells nginx to listen on
port 443 and which certificates to use.  I also found some documentation about how to
configure nginx on how to [deploy Diffie-Hellman for TLS](https://weakdh.org/sysadmin.html)
that seemed like good advice.

The advice instructed me to generate a strong DH group via `openssl` and configure
nginx to disable export grade cipher suites.  These are reflected in the `server` stanza
via the parameters `ssl_ciphers` and `ssl_dhparam`.

The last thing to do is put it all of this together to run the nginx container:

`$ sudo docker run --restart always -v /etc/nginx/dhparams.pem:/usr/share/nginx/dhparams.pem:ro -v /etc/letsencrypt:/etc/letsencrypt:ro -v /etc/nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf:ro -v /var/srv/workstation/repo/:/usr/share/nginx/html/repo:ro -d -p 443:443 -p 80:80 docker.io/nginx`

I checked that accessing faw.piratemirror.party via HTTP and HTTPS was successful
(and always ended up using HTTPS).

## Automating the Mirror

The last thing I wanted to do automate as much of as this as possible...using
containers, of course!

After a few experiments, I was able to create a solution using a container, a
bash script, and some systemd functionality.

I started with a bash script that would handle the mirroring of the content.
This was just a wrapper around some of the `ostree` commands I had used earlier.

```
$ cat mirror.sh
#!/bin/bash
set -xeou pipefail

# You need to define a 'prod' and 'stage' directory for the script to run
# properly.  If you don't pass in those arguments to the script, it assumes
# you have your directories at '/host/{prod,stage}'.  This is because the
# script is normally executed in a container with directories bind mounted
# into the container.
prod=${1:-/host/prod/}
stage=${2:-/host/stage/}
ref="fedora/27/x86_64/workstation"

if [[ ! -d "$prod" ]] || [[ ! -d "$stage" ]]; then
    echo "Must have 'staging' and 'repo' directories present"
    exit 1
fi

# Add the source of truth, mirror the latest commit, prune anything older
# than 7 days, generate the summary and then rsync to prod.
#
# NOTE: because this is typically run from a container, we assume the
# location of the 'rsync-repos' script
ostree --repo=$stage remote add --if-not-exists --set gpgkeypath=/etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-27-primary onerepo https://kojipkgs.fedoraproject.org/atomic/repo/
ostree --repo=$stage pull --mirror --depth=1 onerepo:$ref
ostree --repo=$stage prune --keep-younger-than="7 days ago" $ref
ostree --repo=$stage summary -u
/root/rsync-repos --src $stage --dest $prod
```

We define 'stage' and 'prod' locations for the mirroring.  This is done to avoid
any porential race conditions where a client may be updating from the mirror at
the same time new content is pulled in (or being pruned).  So we end up mirroring
content to the 'stage' location, then using the [rysnc-repos](https://github.com/ostreedev/ostree-releng-scripts/blob/master/rsync-repos)
script to intelligently sync the data from the 'stage' location to the 'prod'
location.

Next up was to create a container to run this script for us.  (I mean, I could have
just stuck the script in `/usr/local/bin`, but where is the excitement in that?)

```
$ cat Dockerfile
FROM registry.fedoraproject.org/fedora:27
LABEL maintainer="Micah Abbott <miabbott@redhat.com>"
RUN dnf -y install ostree python2 rsync && \
    dnf clean all && \
    curl -L -o /root/rsync-repos https://raw.githubusercontent.com/ostreedev/ostree-releng-scripts/master/rsync-repos && \
    chmod +x /root/rsync-repos
COPY mirror.sh /root/
ENTRYPOINT ["/root/mirror.sh"]
```

Really simple, no?  Just installing some packages, copying in the `rsync-repos`
script, and setting an entrypoint.

When we invoke the container, we'll mount in our 'stage' and 'prod' locations
so that the `mirror.sh` knows how to find them, like this:

`$ sudo docker run -v /var/srv/workstation/stage:/host/stage:ro -v /var/srv/workstation/prod:/host/prod:ro docker.io/miabbott/piratemirror`

The last part of the solution is the `systemd` portion.  I knew I could configure
a `systemd.timer` to kick off a `systemd.service`, so I went looking for examples
of both.  It was a bit harder to find an example of a `systemd.service` running
a container with mounts, but I was able to sort that all out.  And I used the
[rpm-ostreed-automatic.timer](https://github.com/projectatomic/rpm-ostree/blob/master/src/daemon/rpm-ostreed-automatic.timer)
as reference for my `systemd.timer`.

```
$ cat piratemirror.service
[Unit]
Description=FAW Pirate Mirror
Requires=docker.service
After=network.service

[Service]
EnvironmentFile=/etc/sysconfig/piratemirror
Type=oneshot
ExecStartPre=-/usr/bin/docker \
              pull \
              docker.io/miabbott/piratemirror
ExecStart=/usr/bin/docker \
          run \
          $STAGE_MNT \
          $PROD_MNT \
          docker.io/miabbott/piratemirror

$ cat piratemirror.sysconfig
STAGE_MNT="-v /path/to/stage/directory:/host/stage "
PROD_MNT="-v /path/to/prod/directory:/host/prod "

$ cat piratemirror.timer
[Unit]
Description=FAW Pirate Mirror Timer

[Timer]
OnBootSec=1h
OnUnitInactiveSec=12h

[Install]
WantedBy=timers.target
```

The `piratemirror.service` is a `oneshot` sevice that reads in the config file
at `/etc/sysconfig/piratemirror` to populate the values of `STAGE_MNT` and `PROD_MNT`
that can be used to run the `docker.io/miabbott/piratemirror` container.  These
values provide the volume mounts for the container (including the actual `-v` flag).

As you might have guessed, `piratemirror.sysconfig` gets copied to `/etc/sysconfig/piratemirror`.

And finally, the `piratemirror.timer` defines a timer that will start 1h after
boot and will run again every 12h.

With all those in place, the mirror is basically running itself!  Here's what the
`piratemirror.service` looks like in action:

```
$ sudo systemctl status piratemirror.service
● piratemirror.service - FAW Pirate Mirror
   Loaded: loaded (/etc/systemd/system/piratemirror.service; static; vendor preset: disabled)
   Active: inactive (dead) since Thu 2018-03-15 12:48:13 UTC; 2h 32min ago
  Process: 15460 ExecStart=/usr/bin/docker run $STAGE_MNT $PROD_MNT docker.io/miabbott/piratemirror (code=exited, status=0/SUCCESS)
  Process: 15448 ExecStartPre=/usr/bin/docker pull docker.io/miabbott/piratemirror (code=exited, status=0/SUCCESS)
 Main PID: 15460 (code=exited, status=0/SUCCESS)
      CPU: 73ms

Mar 15 12:47:59 f27ah-ams3-01.localdomain docker[15460]: 2 metadata, 0 content objects fetched; 1 KiB transferred in 7 seconds
Mar 15 12:47:59 f27ah-ams3-01.localdomain docker[15460]: + ostree --repo=/host/stage/ prune '--keep-younger-than=7 days ago' fedora/27/x86_64/workstation
Mar 15 12:48:02 f27ah-ams3-01.localdomain docker[15460]: Total objects: 120474
Mar 15 12:48:02 f27ah-ams3-01.localdomain docker[15460]: No unreachable objects
Mar 15 12:48:02 f27ah-ams3-01.localdomain docker[15460]: + ostree --repo=/host/stage/ summary -u
Mar 15 12:48:02 f27ah-ams3-01.localdomain docker[15460]: + /root/rsync-repos --src /host/stage/ --dest /host/prod/
Mar 15 12:48:13 f27ah-ams3-01.localdomain docker[15460]: Executing: rsync -rlpt --include=/objects --include=/objects/** --include=/deltas --include=/deltas/** --exclude=* /host/stage/ /host/prod/ --ignore-exist
Mar 15 12:48:13 f27ah-ams3-01.localdomain docker[15460]: Executing: rsync -rlpt --include=/refs --include=/refs/** --include=/summary --include=/summary.sig --exclude=* /host/stage/ /host/prod/ --delete --ignore
Mar 15 12:48:13 f27ah-ams3-01.localdomain docker[15460]: Executing: rsync -rlpt --include=/objects --include=/objects/** --include=/deltas --include=/deltas/** --exclude=* /host/stage/ /host/prod/ --ignore-exist
Mar 15 12:48:13 f27ah-ams3-01.localdomain systemd[1]: Started FAW Pirate Mirror.
```

## Conclusion

After all that, I've managed to setup an automated mirror of the Fedora 27 Atomic
Workstation ostree content in the European region!

(Well, I still need to automate the Let's Encrypt renewal process, but maybe
that will be another post!)

I've made most of the code available at https://github.com/miabbott/piratemirror

In the future, I may describe some of the failures I ran into when exploring this
project, but for now this project is complete!

Let me know what you think!
