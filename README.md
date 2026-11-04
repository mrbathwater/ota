rooted-graphene OTA
===

See [rooted-graphene](https://github.com/schnatterer/rooted-graphene/) for more details.

This repo executes the builds for the actual OTAs.

Every OTA is signed with this repo's own AVB and OTA keys (public AVB key: [`avb_pkmd.bin`](avb_pkmd.bin)),
so it installs on a device whose bootloader is locked to that key.

| Device | magisk | rootless |
|---|---|---|
| tangorpro (Pixel Tablet) | yes | no |
| tokay (Pixel 9) | yes | yes |
| bluejay (Pixel 6a) | yes | no |
| panther (Pixel 7) | yes | no |

Which devices get a rootless build is set per device in
[`release-multiple.yaml`](.github/workflows/release-multiple.yaml) (`rootless: 'true'`).

Custota OTA URLs:

* magisk: `https://mrbathwater.github.io/ota/magisk`
* rootless: `https://mrbathwater.github.io/ota/rootless`

See also https://mrbathwater.github.io/ota/
