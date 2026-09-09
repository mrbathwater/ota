rooted-graphene OTA
===

See [rooted-graphene](https://github.com/schnatterer/rooted-graphene/) for more details.

This repo executes the builds for the actual OTAs.

Only the rooted (magisk) flavor is built and published automatically. Every OTA is signed with
this repo's own AVB and OTA keys (public AVB key: [`avb_pkmd.bin`](avb_pkmd.bin)), so it installs
on a device whose bootloader is locked to that key. Rootless OTAs are no longer produced by the
pipeline.

You can find the OTA server URLs here:  
https://mrbathwater.github.io/ota/
