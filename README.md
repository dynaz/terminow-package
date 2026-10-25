# TermiNow package repository

Build system and signed APT distribution for [TermiNow](https://terminow.online)
(application id `com.terminow`), a fork of [Termux](https://github.com/termux/termux-app).

This repository is **public** and holds only the package recipes selection, build
scripts, CI and the signed distribution. The TermiNow app source stays in a separate
private repository; nothing here reads or modifies it.

## What it produces

The `TermiNow bootstrap` workflow (`.github/workflows/bootstrap.yml`) builds, from the
pinned [termux-packages](https://github.com/termux/termux-packages) commit in
`scripts/terminow/config.sh`:

- A **bootstrap archive** per architecture, attached to a GitHub Release here, together
  with the proot loader and `bootstrap-sha256sums`.
- A **signed APT repository**, published to the `gh-pages` branch and served at:

  ```
  https://dynaz.github.io/terminow-package/apt/terminow-main
  ```

  `pkg` / `apt` on a device only ever use this URL. Termux's official repositories are
  compiled for `com.termux` and do not work under another package name, so TermiNow needs
  its own build.

The repository package set is defined in `scripts/terminow/config.sh`
(`TERMINOW_EXTRA_PACKAGES`, `TERMINOW_REPO_PACKAGES`) and currently includes
`openssh git tmux proot proot-distro nodejs npm`.

## Signing

The repository is signed with an ed25519 key. Its public key,
`scripts/terminow/terminow-repo.gpg`, is installed on devices by the bootstrap's
`termux-keyring` package; the app trusts that fingerprint. The private key lives only in
the `TERMINOW_REPO_GPG_PRIVATE_KEY` repository secret — never committed or printed. See
`scripts/terminow/generate-repo-key.sh`. Signature verification is always on; the APT
sources use `signed-by`, never `trusted=yes`.

## Releasing

Run the workflow (push to `main` touching `scripts/terminow/**`, or via *Run workflow*).
It builds, publishes `gh-pages`, re-checks the live signed repository, and signature-installs
the packages. Each release also carries a `terminow-bootstrap.properties` asset; the app
repository copies those values into its `app/terminow-bootstrap.properties` after verifying
the release, then builds the next APK. This keeps the two repositories decoupled.

## Migration from the old endpoint

Older installs used `https://dynaz.github.io/termux-app/apt`, which stays available.
`scripts/terminow/migrate-apt-repo.sh` migrates an existing device to the new endpoint; it
probes the new URL with the installed trusted keyring before changing anything and backs up
the replaced files. See `scripts/terminow/APT-MIGRATION.md`.

## License and attribution

See [`LICENSE.md`](LICENSE.md) and [`NOTICE.md`](NOTICE.md). The build scripts derive from
`termux/termux-app` (GPLv3). Built packages carry their own upstream licenses.
