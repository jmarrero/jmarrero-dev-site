---
title: "Enabling rpm-ostree install in a container build"
date: 2021-12-13T00:14:20-04:00
draft: true
---

# Enabling rpm-ostree install in container builds

## Why?

As part of the CoreOS layering features we are working on enabling [rpm-ostree install support in container builds](https://github.com/coreos/rpm-ostree/issues/3227) that will produce oci compliant images. 

The challenge is that in contrast to dnf or yum rpm-ostree depends on D-Bus to work. 
... this needs expansion... for example why do we need dbus?

- Using system D to track the file operations? 
- Since host/container do not share systemd configs? 
- systemd is not available on priv containers or builds?

## Refresher on D-Bus and GLib
The fun part is... that I need a refresher on d-bus to understand exactly what I am tracking down. The journey starts [here](https://dbus.freedesktop.org/doc/dbus-tutorial.html)

That was a nice refresh, we encounter dbus all the time while using Linux but developing is another mater. From there we can see a bit of the [GLib](https://docs.gtk.org/gio/) dbus doc. On rpm-ostree we use GLib extensively so this is probably helpful too.

With that out of the way we can start trying to disable D-Bus.

## Identify the error we get when try to install in a container build.

When we attempt to run rpm-ostree install on a build we currently get:

```
error: This system was not booted via libostree; cannot operate
Error: error building at STEP "RUN rpm-ostree install zsh": error while running runtime: exit status 1
```

which comes from:
```
src/app/libmain.cxx @line 281
```

This conditional block that starts on line 271
```
      if (!opt_sysroot)
```

Based on were it is set:
```  { "sysroot", 0, 0, G_OPTION_ARG_STRING, &opt_sysroot, "Use system root SYSROOT (default: /)", "SYSROOT" },
```

I think I understand that it is trying to see if it's set or not. Which my guess is that on the oci build this is not set.

## Bypass SYSROOT check

bypassing the SYSROOT check gets me:

```
System has not been booted with systemd as init system (PID 1). Can't operate.
Failed to connect to bus: Host is down
```

We are entering this block as part of this failure too:

```
if (!rpmostree_load_sysroot (opt_sysroot, cancellable,
                                   out_sysroot_proxy,
                                   error))
```

Removing that if gives us a nice D-Bus error:

```
STEP 4/4: RUN rpm-ostree install zsh

(rpm-ostree install:1): GLib-GObject-CRITICAL **: 17:13:54.068: g_object_get: assertion 'G_IS_OBJECT (object)' failed
Bail out! rpm-ostreed:ERROR:src/app/libmain.cxx:526:int rpmostreecxx::rpmostree_main(rust::cxxbridge1::Slice<const rust::cxxbridge1::Str>): assertion failed: (error && *error)

(rpm-ostree install:1): GLib-GIO-CRITICAL **: 17:13:54.068: g_dbus_proxy_call_sync_internal: assertion 'G_IS_DBUS_PROXY (proxy)' failed
**
rpm-ostreed:ERROR:src/app/libmain.cxx:526:int rpmostreecxx::rpmostree_main(rust::cxxbridge1::Slice<const rust::cxxbridge1::Str>): assertion failed: (error && *error)
container exited on segmentation fault
Error: error building at STEP "RUN rpm-ostree install zsh": error while running runtime: exit status 1
```

## Looking at DBUS error on: `rpm-ostreed:ERROR:src/app/libmain.cxx:526:int`

