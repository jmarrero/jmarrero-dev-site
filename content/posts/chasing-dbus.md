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



## Removing initial code paths.
Leaving this branch without squash to track the debug path, but actual code changes will be raised from nicely formated branches.