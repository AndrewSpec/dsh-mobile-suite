# Historia zmian

[English](CHANGELOG.md) | [Polski](CHANGELOG.pl.md)

## Nieopublikowane — 2026-10-20

### Tłumaczenia

- Aplikacja mobilna, ustawienia Companion i pliki zdalne, strony Relay oraz publiczna strona obsługują angielski, polski i chiński uproszczony.
- Każda powierzchnia podąża za językiem telefonu lub przeglądarki, wraca do angielskiego i udostępnia widoczny wybór języka.
- Dodano polską dokumentację komponentów i zestawu.

### Naprawiono

- Mobile: okno zmiany nazwy komputera nie zwalnia już kontrolera tekstu, gdy trasa nadal się animuje. Znika błąd `TextEditingController was used after being disposed` i następująca po nim awaria `_dependents.isEmpty`.
- Mobile: usunięto prawdziwy adres LAN z fixture'a testu parsera parowania.

### Dokumentacja

- Dodano [opis zmian](docs/CHANGES.pl.md) / [English](docs/CHANGES.md) według modułów, ze zrzutami z telefonu, przeglądem prywatności i wynikami weryfikacji.

## 0.1.7 — 2026-08-19

### Dostęp przez przeglądarkę i Companion

- Dodano niezależne dołączanie przeglądarki do już sparowanego komputera.
- Dodano w lokalnych ustawieniach DSH możliwość wygenerowania kodu QR i odnośnika dostępu dla przeglądarki.
- Dołączanie przeglądarki akceptuje zarówno standardowy kod QR parowania komputera, jak i kod QR dostępu przeglądarkowego.
- Telefon, Safari na iPhonie i dodatkowe przeglądarki mogą korzystać z tego samego komputera jednocześnie, bez zastępowania swoich kluczy E2EE.
- Dodano sprawdzanie aktualizacji Companion oraz lokalną aktualizację jednym kliknięciem. Po instalacji trzeba ponownie uruchomić DSH.
- Naprawiono przywracanie po odświeżeniu przeglądarki przez wznowienie tunelu Service Worker i odnowienie krótkotrwałego biletu dostępu do WWW.

### Bezpieczeństwo

- Klucze E2EE przeglądarki podczas dołączania pozostają we fragmencie adresu URL i są przechowywane wyłącznie w IndexedDB bieżącej przeglądarki.
- Aktualizacje wykonywane z przeglądarki są dostępne tylko lokalnie i korzystają ze zweryfikowanych tagów oficjalnego repozytorium wtyczki.
- Relay nadal przekazuje nieprzezroczyste, zaszyfrowane ramki bez przechowywania treści żądań i odpowiedzi DSH.

### Wydania

- Companion: [v0.1.7](https://github.com/april-jk/dsh-mobile-plugin/releases/tag/v0.1.7)
- Relay: [v0.1.8](https://github.com/april-jk/dsh-relay/releases/tag/v0.1.8)

Wspólne wydanie Suite pozostaje w wersji `v0.1.5`, dopóki wersje komponentów Android, Relay, Companion i strony internetowej nie zostaną wyrównane na potrzeby jednego podpisanego wydania.
