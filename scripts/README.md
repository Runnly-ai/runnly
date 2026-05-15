# Runnly Scripts

This directory contains helper scripts for building, packaging, and installing
Runnly release artifacts.

## Release Pipeline

To stage the npm release tarballs for a version:

```shell
python scripts/stage_npm_packages.py \
  --release-version 0.1.0 \
  --package codex
```

This produces the release tarballs under `dist/npm/`:

- `runnly-npm-0.1.0.tgz`
- `runnly-npm-linux-x64-0.1.0-linux-x64.tgz`
- `runnly-npm-linux-arm64-0.1.0-linux-arm64.tgz`
- `runnly-npm-darwin-x64-0.1.0-darwin-x64.tgz`
- `runnly-npm-darwin-arm64-0.1.0-darwin-arm64.tgz`
- `runnly-npm-win32-x64-0.1.0-win32-x64.tgz`
- `runnly-npm-win32-arm64-0.1.0-win32-arm64.tgz`

Publish the platform tarballs first, then publish the root tarball last:

```shell
npm publish dist/npm/runnly-npm-linux-x64-0.1.0-linux-x64.tgz --access public
npm publish dist/npm/runnly-npm-linux-arm64-0.1.0-linux-arm64.tgz --access public
npm publish dist/npm/runnly-npm-darwin-x64-0.1.0-darwin-x64.tgz --access public
npm publish dist/npm/runnly-npm-darwin-arm64-0.1.0-darwin-arm64.tgz --access public
npm publish dist/npm/runnly-npm-win32-x64-0.1.0-win32-x64.tgz --access public
npm publish dist/npm/runnly-npm-win32-arm64-0.1.0-win32-arm64.tgz --access public
npm publish dist/npm/runnly-npm-0.1.0.tgz --access public
```

If you invoke `codex-cli/scripts/build_npm_package.py` directly, run
`codex-cli/scripts/install_native_deps.py` first and pass `--vendor-src`
pointing to the populated `vendor/` tree.
