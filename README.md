# rootless-ctr
playground for rootless container experimentation

# First exp: play with runC

let's see what we can do now that runC support rootless.

## Installation

First install Go and Docker (for skopeo build) on your system.

Then install the following components:

* runc > 1.0.0-RC4

  ```
  $ go get github.com/opencontainers/runc
  ```

* skopeo

  ```
  $ go get -d github.com/projectatomic/skopeo/cmd/skopeo
  $ cd $GOPATH/github.com/projectatomic/skopeo/cmd/skopeo
  $ make binary
  $ cp skopeo $GOPATH/bin/
  ```

* oci-image-tool

  ```
  $ go get github.com/opencontainers/image-tools/cmd/oci-image-tool
  ```

## Run your first rootless container !!

Create a playground directory and cd into it.

Use `skopeo` to download the latest alpine image:

```
$ skopeo --insecure-policy copy docker://alpine:latest oci:alpine:latest
```

Unpack the image with `oci-image-tool`:

```
$ mkdir -p alpine-bundle/rootfs
$ oci-image-tool unpack --ref name=latest alpine alpine-bundle/rootfs
```

Generate a rootless ready runc spec:

```
runc spec --rootless -b alpine-bundle
```

If you look at the generated *alpine-bundle/config.json*:

* `process` dict contains the defintion of what will be run in the container
  and how: here an *sh* shell will be run with user **0:0**, some capabilites
  and a specific environment.
* the most interesting part is the `linux.uidMappings` and `linux.gidMappings`:
  here, it shows that your host uid and gid (1000 in my case) will be mapped
  to root uid and gid in the container.

Now run your container:

```
$ runc --root /tmp/runc -b alpine-bundle myctr
```

In another terminal, you can check if you container is running:

```
$ runc --root /tmp/runc list
$ runc --root /tmp/runc status myctr
```

> **Note**
> by default, runC stores container states in /run/runc but it would
> need priviledge access. To run rootless, yu have to specify an
> unpriviledge directory, /tmp/runc for example

Go back to your shell inside your container and type:

```
$ ip a
```

You'll see your host interface and it's ok: if you look at the *config.json*, you'll see that no network namespace have been specified. Thus runC
builds the container using the host network by default.

Now try to create file:

```
$ touch /tmp/toto
```

Ah that's right: the rootfs is configured to be read-only.

Nevermind, let's try to mount your home (because it would be nice).
Exit the shell and Edit the *config.json* and add the following dict in the
mounts list:

```
{
  "destination": "/home/yourusername",
  "type": "bind",
  "source": "/home/yourusername",
  "options": ["rbind", "rw"]
}
```

Now restart the container. You should now be able to see your home directory
from the container and even create a file in it.

Notice that all your files seems to have root rights but remember that there
is a mapping of your host uid/gid to the root uid/gid in the container. Look at
the newly created file on your host and you'll see that it belongs too you.

That quite impressive !!! we can now run rootless container and even share
our host home without any extra-stuff.
