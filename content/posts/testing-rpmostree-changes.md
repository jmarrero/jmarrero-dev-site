---
title: "Change/Build/Test repeat process for rpm-ostree"
date: 2021-12-10T12:50:30-04:00
draft: false
---

# Setting up dev environment for rpm-ostree changes

These are processes that I have been learning from [Colin Walters](https://github.com/cgwalters) on how to setup my development process for rpm-ostree changes.

As a general rule we want to use containers to handle build dependencies and isolate any mistake from the host. In fedora Silverblue I am using toolbox.

## 1. Inside toolbox, install COSA from source. 

This is documented in the following [link](https://github.com/coreos/coreos-assembler/blob/main/docs/devel.md#installing-cosa-inside-an-existing-container).

## 2. Once COSA is setup, make an initial build.

From toolbox with cosa installed just do the standard process for a build

In a new directory i.e.

`~/Development/cosa-builds/fcos-build-rpm-ostree`

### a) get the fedora config. 

```
cosa init https://github.com/coreos/fedora-coreos-config 
```

```
cosa fetch
```

### b) do you initial build
```
cosa build
```

Without a initial build cosa fastbuild will not be able to overlay the changes on top of the original build.


## 3. Setup your rpm-ostree repo and build your changes

### clone the repo and do your changes
```
git clone git@github.com:YOURFORK/rpm-ostree.git
```

... Do you changes...


### time to build rpm-ostree:

### a) Install dependencies:
```
./ci/installdeps.sh
```

### b) Init the submodule:
```
git submodule update --init
```

### c) Run autoconf with aditional flags that will enable faster builds by turning off optimizations:
```
env CFLAGS='-ggdb -Og' CXXFLAGS='-ggdb -Og' ./autogen.sh --prefix=/usr --libdir=/usr/lib64 --sysconfdir=/etc
```

### d) run make to build
```
make
```

## **Until here the process is the same for the two methods. Now we diverge.**

## 3. VM option:

We don't want to do make install, we want to just run cosa build-fast now.

### a) Set the COSADIR which is the directory where we did the cosa build.

```
export COSA_DIR=~/Development/cosa-builds/fcos-build-rpm-ostree/
```

### b) Build the vm

```
cosa build-fast
```

### c) run the vm

```
cosa run
```

Once FCOS is running we can just call ```rpm-ostree``` to test whatever changes have been done.



## 4. Using OCI images

We want to push a base OCI image to a repository. When we ran `cosa build` that generated an oci archive that we can push to our local registry or even quay.

For example on my build, the image is under:

```
~/Development/cosa-builds/fcos-build-rpm-ostree/builds/35.20211208.dev.0/x86_64/fedora-coreos-35.20211208.dev.0-ostree.x86_64.ociarchive
```

Now I can push that oci archive which we will use as base with skopeo like this:

```
skopeo copy oci-archive:fedora-coreos-35.20211208.dev.0-ostree.x86_64.ociarchive containers-storage:localhost/myfcos
```

After we ran `make` we should have the `rpm-ostree` binary in the rpm-ostree/target/debug directory.

Using my new base image that we just pushed, we can just have a Dockerfile like:

```
FROM localhost/myfcos
ADD rpm-ostree /usr/bin/
RUN rpm-ostree --version
```

which then is built like:

```
podman build -t localhost/rpm-ostree .
```

we can see it's running the changed rpm-ostre:

```
STEP 3/3: RUN rpm-ostree --version
rpm-ostree:
 Version: '2021.14'
 Git: v2021.14-42-g1fd37f5c12fb7584e0f9783279024433da58b19a
 Features:
  - bin-unit-tests
  - compose
  - rust
  - fedora-integration
```


## 5. An additional option which uses `make install DESTDIR=` can be used too...

However, this will require building a new image each time you do a change, which is a slower option. COSA overrides are documented [here](https://coreos.github.io/coreos-assembler/working/#using-overrides).


Basically we would go back to our rpm-ostree build and after the `make`, run:
```
make install DESTDIR=/path/to/cosa-workdir/overrides/rootfs
```

and do a 
```
cosa build
```

which will give us the option to either use the VM or the oci image to test.

This post is trying to expand and be more verbose on the rpm-ostree hacking [doc](https://github.com/coreos/rpm-ostree/blob/main/docs/HACKING.md)