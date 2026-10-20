# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

PRoot Apps packages desktop apps as Docker images. The `proot-apps` CLI pulls an image from a
registry, extracts its root filesystem to `$HOME/proot-apps/<image>/`, and runs it unprivileged
under [proot-next](https://github.com/selkies-project/proot-next). The install folder name is the
image reference with `/` and `:` changed to `_` (e.g. `ghcr.io_linuxserver_proot-apps_firefox`).
This is not a sandbox. Apps can escape, and do so on purpose through `/system.socket` (see below).

There is no test suite, linter, or local build system. The only verification is a Docker build
of an app, plus a manual run of it.

## Commands

```bash
./build-install-local <app>      # docker build apps/<app>, extract its rootfs into $HOME/proot-apps/
proot-apps install ghcr.io/<owner>/<repo>:<app>   # then run the app's install hook (printed by the script)
proot-apps run <app>             # launch it
PA_DEBUG=1 proot-apps run <app>  # print the NVIDIA bind list
```

`build-install-local` needs Docker as a non-root user in the `docker` group. It takes
`<owner>/<repo>` from `git remote -v` and passes it as the `REPO` build arg. Each app Dockerfile
starts `FROM ghcr.io/${REPO}:<base>`, so a local build pulls the base image from **this fork's**
GHCR namespace. If CI has not yet published that base image there, the build fails.

## The `REPO` templating rule

All build logic is templated to the repository owner and name, so a fork builds and serves its
own images. Never hard-code `linuxserver/proot-apps` in new code. Use the mechanism that fits:

- Dockerfiles: the `REPO` build arg, for the base image and for the desktop file `Exec=` line.
- `proot-apps`: `REPLACE_USER` / `REPLACE_REPO`. `ci-scripts/create-releases.sh` substitutes them
  at release time.
- `apps/gui/Dockerfile`: a `sed` of `linuxserver/proot-apps` to `${REPO}` in `proot-apps-gui`.
- Base images download `xdg-open` and `ncat` from the repo's `bins` branch
  (`raw.githubusercontent.com/${REPO}/bins/...`). The `origin` remote (`harb88/proot-apps`) has no
  `bins` branch now, so base images built from this fork get broken copies of those files.

## Anatomy of an app (`apps/<name>/`)

Copy an existing app that uses the same base image. Each app has these files:

- `Dockerfile`: a two-stage build. Stage `rootfs` is `FROM ghcr.io/${REPO}:<base>`. The final
  stage is `FROM scratch` + `COPY --from=rootfs / /`, so the image is one flat layer. This is
  required: the `proot-apps` CLI downloads only `.layers[0]` as the root filesystem. The rootfs
  stage must:
  - rewrite the desktop file `Exec=` to
    `/bin/sh -c "$HOME/.local/bin/proot-apps run ghcr.io/${REPO}:<name> %U"`;
  - suffix `Name=` with ` PA`, and rename the file to `<name>-pa.desktop` (no clash with host apps);
  - write the CLI binary name to `/bin-name` (e.g. `echo "firefox-pa" > /bin-name`);
  - `COPY` `entrypoint`, `install`, and `remove` to `/`.
- `entrypoint`: runs on each launch inside proot. Put flags that turn off the app's own sandbox
  here (e.g. `MOZ_FAKE_NO_SANDBOX=1`, `--no-sandbox`), because namespaces do not work under proot.
- `install` / `remove`: run at install and uninstall. They copy or delete the icon, the start
  menu entry, and the desktop shortcut under `$HOME`.
- `package_versions.txt`: a syft package table. The nightly bot writes it. Do not edit it by hand.
  Delete it to force a CI rebuild of that app.

Base images live in `base-images/Dockerfile.<name>`. Most apps use `debian-trixie`,
`ubuntu-resolute`, or `alpine-324`. CI builds only the bases listed in the `base` matrix of
`.github/workflows/build-on-commit.yml`. The `alpine-321`, `alpine-322`, `debian-bookworm`, and
`debian-sid` Dockerfiles are not in that matrix, so do not base an app on them.

Runtime facts an app must allow for:

- Install everything the app needs in its image: GPU drivers, fonts, and locales. At run time the
  CLI binds only fonts, the PulseAudio socket, the cursor theme, and the NVIDIA host libraries.
  The NVIDIA binds apply only to glibc images, never to Alpine images.
- To open something on the host (files, URLs), the guest writes a command to
  `/system.socket`, e.g. `echo "xdg-open $@" | nc -N -U /system.socket`. The base images already
  replace `xdg-open` with this wrapper.

## Adding an app

1. Create `apps/<name>/` as above. `<name>` is lowercase and is what the user types.
2. Add an entry to `metadata/metadata.yml` with `name`, `full_name`, `arch`, `category`, `icon`,
   and `description`. Use one of these categories: AI, AudioVideo, Creative, Development,
   Education, Game, Network, Office, Science, System, Utility. `disabled: True` builds the image
   but hides the app from the README and the GUI.
3. Add the logo to `metadata/img/` (SVG preferred, else a 192x192 PNG).

The CI build matrix comes from `metadata.yml`. An app folder with no metadata entry never builds.

## Generated files: edit the source instead

- `README.md` is generated from `ci-scripts/README.template` + `metadata/metadata.yml` by the
  `template-readme` workflow on each push to `master`. Edit the template, never `README.md`.
- `package_versions.txt` files: see above.

## The `proot-apps` CLI

`proot-apps` is a single bash script. It dispatches on `$1` through a series of top-level
`if [ "${TYPE}" == ... ]` blocks: `install`, `get`, `remove`/`uninstall`, `update`, `run`,
`localrepo`, `completion`, and the private `_complete_*` helpers. A bare app name expands to
`ghcr.io/${DEFAULT_GH_USER}/${DEFAULT_GH_REPO}:<name>`. The script talks to the registry API with
`curl` and the bundled `jq`; it does not use Docker.

Environment variables:

- `PA_REPO_FOLDER`: use a local folder of `app.tar.gz` + `SHALAYER` per image instead of the
  registry. The `localrepo` action fills that folder.
- `LOCALREPO`: an internal flag. The `localrepo` action sets it.
- `PA_DEBUG`, `PA_NO_NVIDIA`: control the NVIDIA bind logic.

The GUI installer is the `gui` app. Its source is a Python GTK4/libadwaita script at
`apps/gui/root/usr/bin/proot-apps-gui`. It reads `metadata/metadata.yml` and `metadata/img/`.

## CI and releases (`.github/workflows/`)

- `build-on-commit`: on each push to `master`, it rebuilds all base images. It also builds and
  pushes each app whose `apps/<name>/**` changed. Use `workflow_dispatch` with `build_all` to
  build every app that is not disabled.
- `package-check`: a nightly job. It regenerates each `package_versions.txt` and commits as
  "Package Bot". A changed file then triggers a rebuild of that app. This job needs a `PAT` secret.
- `prerelease-on-commit`: a prerelease of the CLI tarball on each `master` push, with the latest
  proot-next `main` build. It skips bot commits.
- `publish-release`: runs on a tag push. It ships the latest stable proot-next release (or
  `PROOT_TAG`). The release body is `release-notes/current`. Update that file before a tag.

The proot binary is not compiled here. `ci-scripts/create-releases.sh` downloads it from
proot-next and checks its SHA-256 sum.
