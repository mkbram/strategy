https://coreos.github.io/coreos-installer/customizing-install/

https://coreos.github.io/coreos-installer/iso-embed-ignition/

https://www.redhat.com/en/blog/red-hat-enterprise-linux-coreos-customization

https://github.com/coreos/fedora-coreos-docs/issues/245

https://mirror.openshift.com/pub/openshift-v4/clients/coreos-installer/v0.21.0-3/
https://access.redhat.com/solutions/5670481


###############################################################################################################
```
[root@fedora coreos]# ./coreos-installer  iso  ignition
Embed an Ignition config in a CoreOS live ISO image

Usage: coreos-installer iso ignition <COMMAND>

Commands:
  embed   Embed an Ignition config in an ISO image
  show    Show the embedded Ignition config from an ISO image
  remove  Remove an existing embedded Ignition config from an ISO image

Options:
  -h, --help  Print help
[root@fedora coreos]# ./coreos-installer  iso  ignition show rhcos-4.18.1-x86_64-live.x86_64.iso
Kiran tries an ignition file
[root@fedora coreos]# ./coreos-installer  iso  ignition embed rhcos-4.18.1-x86_64-live.x86_64.iso
This is a bootstrap ignition file
Error: This ISO image already has an embedded Ignition config; use -f to force.
```
