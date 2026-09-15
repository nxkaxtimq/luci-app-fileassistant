# luci-app-fileassistant

A lightweight file manager for LuCI, the OpenWrt web interface. Browse,
upload, download, rename and delete files on your router — and install
`.apk` packages straight from the browser.

## Why this fork exists

File Assistant and I go back to the opkg days: it lived in the
immortalwrt LuCI feed, it did one job, and it did it well. When OpenWrt
switched to the apk package manager the plugin was dropped from the
feeds, and no ready-made `.apk` ever showed up to take its place. I went
looking for another file manager I would like just as much and came up
empty-handed. Since I don't write code myself, I did what the moment
asks of all of us — I let AI do the heavy lifting, brought the plugin
up to date for the apk era, and compiled it myself.

## Where the code comes from

This tree is the package exactly as it last existed upstream at
[immortalwrt/luci/applications/luci-app-fileassistant@d7bf7363](https://github.com/immortalwrt/luci/tree/d7bf73637a678a245370bfa9f41af3f7e37f6655/applications/luci-app-fileassistant):
the parent of the removal commit `b0d7db5e` ("luci-app-fileassistant:
drop package", 2024-08-29), which is byte-for-byte identical to
`e976b441`, the last commit that ever touched the app (verified with
git blob-hash comparisons against the upstream repository). Every
commit on top of that import documents one self-contained change.

## Changes for the apk era

- packages are installed with `apk add` instead of `opkg`; only `.apk`
  files are accepted and the opkg/.ipk code paths are gone for good
- the installer reports the real exit status instead of always
  claiming success
- paths carrying shell metacharacters are rejected before anything is
  executed
- two latent Lua bugs fixed: an invalid `"\\ "` escape that silently
  disabled space escaping (and breaks Lua 5.3+ parsers), and an
  assignment to a read-only `for` loop variable
- the legacy `rm -rf /tmp/luci-*` cache cleanup after installation was
  dropped: the package's own postinst clears the LuCI caches and
  reloads rpcd, and the menu cache self-heals across upgrades anyway
  (it is keyed by file size/inode)
- file operations run on nixio filesystem calls instead of shell
  interpolation, with upload size limits, filename sanitization and a
  browsable-path whitelist (/mnt, /etc, /root, /tmp, /www — edit
  ALLOWED_PATHS in the controller to adjust)
- the menu registers through menu.d on current LuCI, with a legacy
  fallback for older releases
- the Makefile builds the package standalone from `package/` as well
  as from inside the luci feed

## Building it yourself

```bash
# grab a snapshot SDK for your target from
# https://downloads.openwrt.org/snapshots/targets/<target>/<subtarget>/
./scripts/feeds update luci
./scripts/feeds install luci-base
ln -s /path/to/repo/luci-app-fileassistant package/luci-app-fileassistant
echo 'CONFIG_PACKAGE_luci-app-fileassistant=m' >> .config
make defconfig
make package/luci-app-fileassistant/compile -j$(nproc) V=s
# result: bin/packages/<arch>/luci/luci-app-fileassistant-1.0-r4.apk
```

Snapshot SDKs sign every package they build with the key pair they
generate on first use (`private-key.pem` / `public-key.pem` in the SDK
root). Keep the private key to yourself; the public key is what makes
installation pleasant below.

## A note on signatures

Packages from the official snapshot repositories carry **no
per-package signature** — the trust lives in the signed repository
index (`packages.adb`), which is why `apk add` straight from a
repository just works. The moment you feed apk a plain file instead, it
checks the file's own signature, and official files do not have one.
That is not a bug in this package; it is how the official distribution
model works. The install paths below deal with it in different ways.

## One prerequisite for installing local files

apk-tools 3.x refuses a bare local-file install with a "would be lost
on next reboot" guard unless package caching is enabled. Enable it
once and the plain commands below work as-is:

```bash
mkdir -p /etc/apk/cache
```

The app does this automatically before installing, so the Install
button never needs your help.

## Installing

Pick whichever path fits the situation. The examples assume the
package and the matching `public-key.pem` are already on the router
(see *Downloading* below).

### Quick test — no key setup

```bash
apk add --allow-untrusted /tmp/luci-app-fileassistant-1.0-r4.apk
```

Skips signature verification entirely. Fine for a throwaway VM or a
one-off experiment; do not make a habit of it on a production router.

### Recommended — trust the build key

```bash
# once per router:
cp /tmp/public-key.pem /etc/apk/keys/fileassistant-build.pem
# every install or upgrade:
apk add /tmp/luci-app-fileassistant-1.0-r4.apk
```

With the public key in place the signature verifies normally — no
trust flags, nothing else.

### Local repository index

For offline or batch installs, build a proper little repository on the
machine that holds the signing key:

```bash
mkdir repo && cp luci-app-fileassistant-1.0-r4.apk repo/
apk mkndx --sign private-key.pem --output repo/packages.adb repo/*.apk
# copy repo/ to the router, then:
apk add --repository /mnt/repo/packages.adb luci-app-fileassistant
```

Note the asymmetry at index time: packages signed by your SDK key go
in as shown, but *unsigned* packages — which includes everything from
the official snapshot repositories — make `mkndx` refuse the index
unless you add `--allow-untrusted` and point it at a keys directory
holding your public key. That flag vouches for contents apk cannot
verify; only use it knowingly.

### One-shot key trial

```bash
mkdir /tmp/keys && cp /tmp/public-key.pem /tmp/keys/
apk --keys-dir /tmp/keys add /tmp/luci-app-fileassistant-1.0-r4.apk
```

`--keys-dir` replaces (not extends) the trusted key set for this one
transaction — a way to try the build key before committing it to
`/etc/apk/keys/`. Being a global option it must precede `add`.

## Downloading

Release assets live at
https://github.com/nxkaxtimq/luci-app-fileassistant/releases — the
signed `.apk` plus `public-key.pem` and `SHA256SUMS`. On the router
itself, `wget` is uclient-fetch and needs `libustream-ssl` plus
`ca-bundle` for HTTPS; without them, download on a PC and `scp` both
files over — that pairs naturally with the recommended install path
above.

```bash
curl -L -o /tmp/luci-app-fileassistant-1.0-r4.apk \\
  https://github.com/nxkaxtimq/luci-app-fileassistant/releases/download/v1.0-r4/luci-app-fileassistant-1.0-r4.apk
curl -L -o /tmp/public-key.pem \\
  https://github.com/nxkaxtimq/luci-app-fileassistant/releases/download/v1.0-r4/public-key.pem
```

## How the app installs packages

The Install button first tries the strict path — the package signature
must be trusted by the router, exactly like the recommended install
above. If that fails, a confirmation dialog explains that signature
verification will be skipped and, only on explicit confirmation,
retries the install with `--allow-untrusted`. Nothing is remembered
between clicks: every untrusted install is a fresh decision, visible
as a separate request in hindsight.

After installation the page lives under the **NAS** menu in LuCI.

## License

Apache License 2.0, as the original LuCI application — see `LICENSE`,
with the upstream attribution notice in `NOTICE`. Modifications in
this repository are documented at repository level: the git history
and the change list above describe every deviation from the imported
upstream tree (Apache-2.0 §4(b) is thereby satisfied at repository
granularity).
