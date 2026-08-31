# OTA update API (`api/v1/<device>/<channel>/`)

Each `index.html` is the JSON the LineageOS-style Updater app polls
(`Settings > System > System updates`). Format:

```json
{"response":[{"datetime":<int>,"filename":"...","id":"...","romtype":"unofficial","size":<int>,"url":"...","version":"1.4.1"}]}
```

## The `datetime` field (read this before editing)

`datetime` MUST equal the `ro.build.date.utc` baked into the exact image this
endpoint's `url` serves. The Updater (`packages/apps/Updater`, `Utils.isCompatible`)
offers the update when, and only when:

```
datetime > ro.build.date.utc   (the value on the user's device)
```

The version string is only a `>=` gate, so it does NOT stop a same-version
re-offer. If `datetime` is set to a "release day" wall-clock time (or anything
later than the shipped image's build date), every device already on that build is
offered it again forever. This happened with the 1.4.1 manifests and is the whole
reason this note exists.

### How to get the correct value

It is the `ro.build.date.utc` line in the shipped image's `system/build.prop`
(NOT the OTA packaging time; `gen_ota_package.sh` stamps the zip's internal
`manifest.json` with `date +%s`, which is a different, later timestamp). To read
it from a packaged OTA zip:

```
unzip -p <device>_..._OTA.zip system.img.xz | xz -dc > /tmp/system.img
fsck.erofs --extract=/tmp/sys --no-preserve /tmp/system.img
grep ro.build.date.utc /tmp/sys/system/build.prop
```

Set `datetime` to that number. A device on the exact shipped build then sees
`datetime == ro.build.date.utc` (not offered), while genuinely older builds still
see the update.

### 1.4.1 reference values

| endpoint | serves | ro.build.date.utc |
|----------|--------|-------------------|
| anbernicrgds/lite   | RG_DS_..._Core_v1.4.1     | 1786579075 |
| anbernicrgds/full   | RG_DS_..._Full_v1.4.1     | 1786583905 |
| anbernicrgrotate/full | RG_Rotate_..._Full_v1.4.1 | 1786583905 |
| anbernicrgrotate/lite | RG_Rotate_..._Lite_v1.4.1 | 1786580986 |
