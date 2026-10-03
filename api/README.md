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

### 1.4.3 reference values

| endpoint | serves | ro.build.date.utc |
|----------|--------|-------------------|
| anbernicrgdsplus/core | RG_DS_Plus_..._Core_v1.4.3 | 1790694240 |

### 1.4.4 reference values

Both Core packages serve the same 20261002 system image, so they share one
ro.build.date.utc. The RG DS Lite and Full packages are separate 20261003
system builds, each with its own value.

| endpoint | serves | ro.build.date.utc |
|----------|--------|-------------------|
| anbernicrgds/core     | RG_DS_..._Core_v1.4.4      | 1790977387 |
| anbernicrgds/lite     | RG_DS_..._Core_v1.4.4 (see below) | 1790977387 |
| anbernicrgds/full     | RG_DS_..._Full_v1.4.4      | 1791020312 |
| anbernicrgdsplus/core | RG_DS_Plus_..._Core_v1.4.4 | 1790977387 |

**Why anbernicrgds/lite serves the Core package.** The core variant only exists
since 2026-08-17 (after the 1.4.1 release), so every RG DS on 1.4.1 Core reports
ro.gammaos.variant=lite and polls this endpoint. No Lite build was ever
published for the RG DS at 1.4.1, so everything polling lite today is a Core
device, and it must be given the Core package (which moves it to the core
variant for the next release). A device freshly installed with Lite 1.4.4 also
polls lite but has a later ro.build.date.utc (1791015900) than this entry, so
it is not offered the Core package. Switch this endpoint to the Lite package
only for a release newer than Lite 1.4.4.

## Channel names are the `variant`, not a guess

The Updater polls `api/v1/{ro.gammaos.device}/{ro.gammaos.variant}`, and
`vendor/lineage/build/core/main_version.mk` derives the variant from the product
name: `tv_*` -> **core**, `bgN` -> **full**, everything else -> **lite**. An Android
TV build therefore asks for `.../core`, so that directory has to exist for it.
Pointing a `lite` endpoint at a Core package does not help a Core device: it never
requests `lite`.

The `id` field is the first 16 hex characters of the served zip's SHA-256, and
`size` is that zip's exact byte count.
