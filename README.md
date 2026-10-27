# Orbi downloads

Orbi keeps your coding agents at the edge of your screen. This repository holds public downloads, release notes, setup documentation, and support issues. Development source is kept in a separate private repository.

[Website](https://orbi.chrystalesia.com/) · [Install](install.md) · [Support](https://orbi.chrystalesia.com/support.html) · [Report an issue](https://github.com/RaflyLesmana3003/orbi-releases/issues)

| Desktop | Download |
|---|---|
| Windows x64 | [Portable app and MCP bridge](https://github.com/RaflyLesmana3003/orbi-releases/releases/tag/windows-latest) |
| macOS Apple Silicon | [DMG](https://github.com/RaflyLesmana3003/orbi-releases/releases/tag/macos-latest) |
| Linux x86_64 | [AppImage](https://github.com/RaflyLesmana3003/orbi-releases/releases/tag/linux-latest) |

See [all releases](https://github.com/RaflyLesmana3003/orbi-releases/releases) for versioned packages and release notes. Linux runtime validation is pending. macOS builds are not notarized.

## Install with your coding agent

Copy this prompt into your desktop coding agent:

```text
Install Orbi for me by following https://orbi.chrystalesia.com/install.md
```

The guide checks download hashes, preserves existing settings, and connects the local MCP bridge. See the [bridge guide](bridge.md), [tool reference](tools.md), and [agent instruction block](agent-instructions.md).

## Verify a download

GitHub's [release metadata](https://api.github.com/repos/RaflyLesmana3003/orbi-releases/releases) provides a SHA-256 digest for each asset. Compare it with `Get-FileHash -Algorithm SHA256`, `shasum -a 256`, or `sha256sum` before running a download. Versioned Linux packages also include `SHA256SUMS`.

## Notices

Existing code licence and copyright notices are retained in [LICENSE](LICENSE). The Orbi name, character, icons, sounds, and media are covered separately by [LICENSE-ASSETS.md](LICENSE-ASSETS.md). Making the development repository private does not change these notices.

GitHub's automatically generated source archives contain this documentation repository. Use the named release assets to install Orbi.
