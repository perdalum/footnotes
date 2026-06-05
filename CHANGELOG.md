# Changelog

All notable changes to this project will be documented in this file.

This project follows the spirit of [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and uses semantic versioning for specification releases.

## [1.0.1] - 2026-06-05

### Changed

- Updated documentation examples to use Wikipedia instead of Outlook.

## [1.0.0] - 2026-06-04

### Added

- Initial Clear-Text Footnotes specification.
- Source footnote annotation syntax using `ƒ(<footnote text>)`.
- Render transformation from source annotations to superscript references and a rendered footnote block.
- Reverse transformation from rendered footnotes back to source annotations.
- Mixed input normalization for text containing both rendered footnotes and new source annotations.
- Conservative rendered-footnote detection rules.
- Portable test scenarios for future implementations.
- Implementation-neutral parser and detection algorithm notes.
- Project README, pitch, context, specification, and implementation design documents.
