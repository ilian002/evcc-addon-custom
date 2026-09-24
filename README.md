# evcc (custom build) — Home Assistant add-on repository

A Supervisor repository that installs a private build of evcc: release **0.316.0** plus the `schneider-micrologic` meter template from [evcc-io/evcc#33770](https://github.com/evcc-io/evcc/pull/33770), which has not merged yet. A second add-on, **evcc (PR 33770 test)**, runs the template as it stands in the PR next to it; see [Test instance](#test-instance).

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

## Test instance

**evcc (PR 33770 test)** (`evcc-pr33770/`) is a second evcc that runs alongside the production add-on, to test the template exactly as it stands at the head of #33770. Image source: [`ilian002/evcc`, branch `build/pr-33770`](https://github.com/ilian002/evcc/tree/build/pr-33770), tag `0.316.0-pr33770.*`.

It shares nothing with production:

- **Own database** `/homeassistant/evcc-pr33770.db`. The PR's template dropped the `tripunit` and `systemtype` params, and evcc rejects a stored device with a key its template no longer has (`invalid key`). Pointed at the production database, it would fail to load every MicroLogic meter there.
- **Own port 7071**, set in `/homeassistant/evcc-pr33770.yaml`. Both instances use `host_network: true`, and evcc only reads the port from the config file while its database has no network settings. So **create that file before the first start**, or the instance comes up on 7070 and collides with production:

  ```yaml
  network:
    port: 7071
    host: evcc-pr33770
  ```

- **Not started on boot**, and nothing to control: add only meters to it. Leave out chargers, loadpoints, circuits and MQTT, so production stays the only instance acting on the site or publishing to the broker.

Both instances poll the same breakers through the Modbus gateway, which doubles the read load on the serial line.

Rebuild the same way as above, on the `build/pr-33770` branch, with `version:` in `evcc-pr33770/config.yaml` raised to match.
