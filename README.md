# evcc (custom build) — Home Assistant add-on repository

A one-add-on Supervisor repository that installs a private build of evcc: release **0.315.2** plus the `schneider-micrologic` meter template from [evcc-io/evcc#33770](https://github.com/evcc-io/evcc/pull/33770), which has not merged yet.

It exists because evcc compiles its device templates into the binary with `//go:embed`, and the official add-on's entrypoint only ever runs `evcc --config <file>` — so an unmerged template cannot be injected into the stock image at runtime. The only way to use one early is to ship a build that contains it.

Image source: [`ilian002/evcc`, branch `build/custom-image`](https://github.com/ilian002/evcc/tree/build/custom-image). Built for **aarch64** only (Home Assistant Yellow). Published to `ghcr.io/ilian002/evcc`.

## Install

Settings → Add-ons → Add-on Store → ⋮ → Repositories → add `https://github.com/ilian002/evcc-addon-custom` → install **evcc (custom build)**.

**Stop the official evcc add-on first.** Both use `host_network: true` with fixed ports (7070, 8887, 5353, …), so whichever starts second crash-loops. Also turn off "Start on boot" for whichever one is not in use.

## Shared database

Both add-ons are meant to point at the same files on the `homeassistant_config` mount:

```
sqlite_file: /homeassistant/evcc.db
config_file: /homeassistant/evcc.yaml
```

This add-on ships those as defaults; **set them on the official add-on too**. Without it, each add-on gets its own `/data` volume and the device configuration built in the UI is stranded in whichever one created it.

## Going back to the official add-on

A meter created from this template is stored in the database as `type: template, template: schneider-micrologic`. The stock image has no such template, so it will fail to load that device.

So: keep the official add-on **installed but stopped**, and only switch back once an official image actually contains the template — after #33770 merges, that is the next nightly (built 02:00 UTC, mirrored to the nightly add-on ~20 minutes later) or the next stable release. Then stop this add-on, start the official one, and remove this repository.

## Rebuilding

Bump `IMAGE_TAG` in `.github/workflows/build-image.yml` on the `build/custom-image` branch and push. Then raise `version:` in `evcc-custom/config.yaml` to match — Supervisor only re-pulls when that string changes, so tags must never be reused.
