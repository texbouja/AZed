# AZed

AZed is a scientific text editor supporting Markdown, Typst and LaTeX, forked from [Zed](https://zed.dev).

## Project Status

This is an early-stage fork. The goal is to add:
- **Webview-based HTML rendering** (via wry) for rich previews
- **PDF viewer** with PDF.js and SyncTeX
- **Markdown** powered by marksman LSP + KaTeX
- **Typst** powered by tinymist LSP
- **LaTeX** powered by texlab LSP + SyncTeX
- **Graphical settings UI**
- **Obsidian-like project vaults** (`.azed/` config folders)

## Compatibility

AZed keeps `APP_NAME = "Zed"` internally, meaning:
- Config and data paths remain `~/.config/zed/`, `~/.local/share/zed/`
- All existing Zed extensions work without reinstallation
- AZed-specific data lives in `~/.config/az/` and `~/.local/share/az/`

## License

AZed is a fork of [Zed](https://zed.dev), which is licensed under GPL-3.0-or-later and Apache-2.0.
The original code is copyright Zed Industries, Inc.
