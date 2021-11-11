---
title: "Running CoreOS Kola tests on DigitalOcean"
date: 2021-11-10T00:14:20-04:00
draft: false
---


Currently there are no Fedora CoreOS images on DigitalOcean so you will need to upload your own.

Go to <https://getfedora.org/en/coreos> -> `Download Now` -> `For Cloud Operators` -> `DigitalOcean` -> right click on the `download` link and copy the source url. At this time I got: <https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20211031.3.0/x86_64/fedora-coreos-34.20211031.3.0-digitalocean.x86_64.qcow2.gz>


You will also need your DigitalOcean api token you can generate one at: <https://cloud.digitalocean.com/account/api/tokens>


Make sure you have cosa setup on your system an fast and easy way to do it is using the container and an alias as detailed here:
<https://coreos.github.io/coreos-assembler/building-fcos/#define-a-bash-alias-to-run-cosa>

Enter the COSA shell:
```
cosa shell
```

Push the coreos image to DigitalOcean:
```
ore do --token=MYTOKEN create-image --custom --name fcos --url https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20211031.3.0/x86_64/fedora-coreos-34.20211031.3.0-digitalocean.x86_64.qcow2.gz
```

Replace MYTOKEN with your DigitalOcean token and set the `--name` to your preffered image name, on my example it's just fcos.

Finally run the Kola tests by running:
```
kola run --arch=x86_64 -p=do --do-image=fcos --do-size=s-2vcpu-2gb --do-region=nyc3 --do-token=MYTOKEN coreos.ignition.security.tls
```
Replace the region, --do-image and token as needed.

On this example I am just running the test `coreos.ignition.security.tls` if no test is specified the full suite will run.

If you get an error saying that it can't find the image you might need to copy the image to the region you are trying to run the tests on. In this example I was using `--do-region=nyc3` which required me to copy the image to that region on this page:
<https://cloud.digitalocean.com/images/custom_images>