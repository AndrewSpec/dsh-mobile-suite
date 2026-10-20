# Changelog

[English](CHANGELOG.md) | [Polski](CHANGELOG.pl.md)

## Unreleased - 2026-10-20

### Localization

- Mobile language and theme choices persist across launches in secure storage and are applied before the first frame.
- Mobile, Companion settings and remote files, Relay web pages, and the public website now support English, Polish, and Simplified Chinese.
- Every surface follows the phone or browser language, falls back to English, and exposes a visible language selector.
- Added Polish component and suite documentation.

### Fixed

- Mobile: the signed-in e-mail in Settings is hidden by default (`Ukryty`) and revealed only on request, so the address is not exposed on screen or in screenshots.
- Mobile: the rename-computer dialog no longer disposes its text controller while the route is still animating. This removes the `TextEditingController was used after being disposed` error and the follow-up `_dependents.isEmpty` failure.
- Mobile: removed a real LAN address from a pairing-parser test fixture.

### Documentation

- Added the per-module [change summary](docs/CHANGES.md) / [opis zmian](docs/CHANGES.pl.md) with device screenshots, privacy review, and verification evidence.

## 0.1.7 - 2026-08-19

### Browser and Companion access

- Added independent browser enrollment for an already paired computer.
- Added a local DSH settings action to generate a browser-access QR code and link.
- Browser enrollment accepts both the normal computer pairing QR and the browser-access QR.
- Phone, iPhone Safari, and additional browsers can use the same computer concurrently without replacing each other's E2EE key.
- Added Companion update checks and a local one-click updater. DSH must be restarted after installation.
- Fixed browser refresh recovery by resuming the Service Worker tunnel and renewing the short-lived web ticket.

### Security

- Browser E2EE keys remain in the URL fragment during enrollment and are stored only in the current browser's IndexedDB.
- Browser update actions are local-only and use validated tags from the official plugin repository.
- Relay continues to route opaque sealed frames without storing DSH request or response content.

### Releases

- Companion: [v0.1.7](https://github.com/april-jk/dsh-mobile-plugin/releases/tag/v0.1.7)
- Relay: [v0.1.8](https://github.com/april-jk/dsh-relay/releases/tag/v0.1.8)

The unified Suite release remains `v0.1.5` until the Android, Relay, Companion, and website component versions are aligned for a single signed release.
