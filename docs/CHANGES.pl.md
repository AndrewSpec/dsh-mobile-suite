# Aktualizacja tłumaczeń — opis zmian

Przygotowano: 2026-10-20

Ten dokument opisuje zmiany źródłowe przygotowane do kolejnej publikacji repozytorium na GitHubie. To podsumowanie zmian, a nie podpisana nota wydania: wspólny workflow wydania nadal wymaga, aby wersje pakietów Plugin, Relay i Mobile były zgodne z wypchniętym tagiem `v*`.

## Zrzuty ekranu z fizycznego urządzenia

Zrzuty wykonano na podłączonym telefonie z Androidem (Samsung Galaxy S24 Ultra) z lokalnej kompilacji tego zestawu zmian. Adres e-mail konta, adres Relay, powiadomienia systemowe oraz paski systemowe zostały przycięte lub pominięte. Nazwy komputerów, ścieżki i kody techniczne celowo pozostają nieprzetłumaczone.

| Lista komputerów po polsku | Lista komputerów po angielsku | Lista komputerów po chińsku |
| --- | --- | --- |
| ![Polska lista komputerów](images/mobile-devices-pl.png) | ![Angielska lista komputerów](images/mobile-devices-en.png) | ![Chińska lista komputerów](images/mobile-devices.png) |

| Polskie działania na komputerze | Polski wybór języka |
| --- | --- |
| ![Polskie działania na komputerze](images/mobile-device-actions-pl.png) | ![Polski wybór języka](images/mobile-language-pl.png) |

![Polskie ustawienia z ukrytym adresem e-mail konta](images/mobile-settings-email-hidden-pl.png)

Menu działań pokazuje dodane w wersji 0.1.5 wejście do dostępu przez przeglądarkę obok zmiany nazwy i usunięcia parowania. Wybór języka to nowa kontrola na poziomie sesji, która przełącza angielski, polski i chiński uproszczony bez restartu aplikacji.

## dsh-mobile (klient Flutter)

### Dodano

- Katalogi tłumaczeń `lib/l10n/en.json`, `lib/l10n/pl.json` i `lib/l10n/zh.json`, wczytywane przed pierwszą klatką przez `AppLocalizations`.
- Wybór języka systemowego z deterministycznym powrotem do angielskiego dla nieobsługiwanych języków. Jawny wybór języka i motywu jest zapisywany lokalnie w bezpiecznym magazynie i nie migruje konta ani nie nadpisuje danych logowania.
- Selektory języka i motywu w Ustawieniach. Oba przełączają aplikację bez restartu, a zapisany wybór jest stosowany przed pierwszą klatką przy następnym uruchomieniu.
- Adres e-mail zalogowanego konta w Ustawieniach jest domyślnie ukryty (`Ukryty`) i pokazuje się tylko na czas bieżącej wizyty po naciśnięciu przycisku oka.
- Tłumaczone znaczniki czasu i formy mnogie (`minutę/minuty/minut`, `godzinę/godziny/godzin`) zamiast jednego sztywnego wzorca.
- Mapowanie błędów API na katalog. Proza z backendu nigdy nie trafia do użytkownika; kody przyczyn mapują się na klucze katalogu.
- Zachowanie języka, motywu i żądanej ścieżki przy przekazaniu komputera do przeglądarki lub widoku plików zdalnych (`session_target.dart`, `remote_files_path.dart`).

### Zmieniono

- Zastąpiono zaszyte chińskie teksty interfejsu wpisami katalogu na ekranach logowania, parowania, listy komputerów, ustawień, aktualizacji i sesji.
- Przebudowano warstwę motywu na jeden właściciel palety i tokenów dla jasnych i ciemnych powierzchni. Widgety czytają kolory semantyczne zamiast lokalnych stałych.

### Naprawiono

- Okno zmiany nazwy zwalniało `TextEditingController` zaraz po powrocie z `showDialog`, gdy trasa okna nadal się animowała. Powstawał błąd `TextEditingController was used after being disposed`, a następnie asercja frameworka `_dependents.isEmpty`. Kontroler jest teraz własnością stanowego okna i jest zwalniany razem z jego trasą.
- Test regresyjny: `test/rename_dialog_test.dart`.

### Dokumentacja

- Dodano `README.pl.md` oraz odnośniki językowe w README angielskim i chińskim.
- Zastąpiono zrzut listy komputerów aktualnym, oczyszczonym zrzutem.

## dsh-plugin (wtyczka DSH i Companion)

### Dodano

- Tłumaczenia w czasie działania dla strony ustawień wtyczki i menedżera plików zdalnych: angielski, polski i chiński uproszczony, wybierane z języka przeglądarki lub widocznego selektora.
- Wydzielono HTML menedżera plików do `src/file-browser.ts`, a obsługę ciasteczka przeglądarki DSH do `src/dsh-browser-cookie.ts`, aby obie strony miały jednego właściciela i dały się testować osobno.
- Testy obsługi ciasteczka, listowania plików i tłumaczeń (`test/browser-cookie.test.ts`, `test/files-listing.test.ts`, `test/localization.test.ts`).

### Zmieniono

- Przetłumaczono etykiety, stany puste, błędy, formatowanie daty, czasu i rozmiaru oraz nazwy dostępne. Nazwy komputerów, ścieżki plików, treść plików, adresy URL i kody przyczyn pozostają nieprzetłumaczone.
- Preferencja języka jest zapisywana per przeglądarka, jeśli magazyn lokalny jest dostępny; nie dotyka konta, parowania ani danych logowania.
- Przebudowano `dist/` ze scalonych źródeł.

### Dokumentacja

- Dodano `README.pl.md`, poprawiono w nim odwołania do wersji instalacyjnej na `v0.1.9` i podlinkowano z README angielskim i chińskim.

## dsh-relay

### Dodano

- `web/app/i18n.js`, wspólny katalog dla aplikacji przeglądarkowej Relay, strony administracyjnej i strony niedostępnej sesji.
- Widoczny selektor języka przed i po logowaniu, zapisywany jako `dsh-relay-language`, z fallbackiem do `navigator.languages` i angielskim jako ostatnim fallbackiem.
- Tłumaczenia w Service Workerze, w tym odpowiedź `secure_tunnel_failed` z `Content-Language` i `Cache-Control: no-store`.
- `test/i18n.test.ts` oraz test CSP dopuszczający ten sam origin `/app/i18n.js` bez osłabiania polityki.

### Zmieniono

- Scalono naprawę wznawiania sesji przeglądarki z `v0.1.9` z tłumaczoną ścieżką błędu tunelu. Nieudane żądanie nadal próbuje jednego reconnectu dla `GET`/`HEAD`; żądania nieidempotentne kończą się tłumaczoną odpowiedzią 502.
- Język nie trafia do kopert tunelu, żądań uwierzytelniania ani zapisanych danych logowania. To wyłącznie metadana prezentacji.

### Dokumentacja

- Dodano `README.pl.md` i odnośniki językowe.

## dsh-website

### Dodano

- Polskie trasy pod `/pl/`: strona główna, pobieranie, jak to działa, bezpieczeństwo, własny Relay, prywatność, regulamin, indeks bloga i trzy artykuły produktowe.
- Nawigację, odnośniki nagłówka i stopki, metadane i teksty alternatywne zależne od języka.
- `scripts/check-locales.mjs` (`npm run check:locales`) do weryfikacji zgodności tras i treści, linków wewnętrznych, metadanych, nazw dostępnych i oczywistych nieprzetłumaczonych tekstów.

### Zmieniono

- Angielski pozostaje domyślnym językiem, a istniejąca chińska strona główna nadal działa. Strony polskie linkują do odpowiednich tras angielskich, jeśli tłumaczony cel jeszcze nie istnieje.

### Dokumentacja

- Dodano `README.pl.md` i opisano kontrole języków.
- Zastąpiono obraz listy komputerów aktualnym, oczyszczonym zrzutem.

## Dokumentacja zestawu i prywatność

### Dokumentacja

- Dodano polskie dokumenty główne: `README.pl.md`, `CHANGELOG.pl.md`, `CONTRIBUTING.pl.md`, `SECURITY.pl.md`, z odnośnikami językowymi w plikach angielskim i chińskim.
- Dodano ten opis zmian po angielsku i polsku oraz podlinkowano go z głównego README i changelogu.

### Prywatność

- Zrzuty przycięto, usuwając pasek stanu (godzina, powiadomienia, operator) i pasek nawigacji. E-mail konta, adres Relay, bilety dostępu i identyfikatory urządzenia nie występują na zrzutach. Zrzut Ustawień pokazuje domyślny stan `Ukryty`; adres pojawia się dopiero po naciśnięciu przycisku oka.
- Widoczna nazwa komputera to neutralny `Domowy komputer`; nie pokazano danych właściciela konta.
- Metadane PNG zostały usunięte przed commitem.
- Preferencje języka i motywu są zapisywane jako dwie lokalne wartości w bezpiecznym magazynie aplikacji; nie zawierają danych konta, parowania ani Relay.
- Prawdziwy adres LAN w `dsh-mobile/test/pair_payload_parser_test.dart` zastąpiono zakresem dokumentacyjnym `192.0.2.10`.
- Robocze rekordy Aegis w `docs/aegis/` zawierają lokalne ścieżki, adresy LAN i szczegóły urządzenia. Są teraz ignorowane przez git i nie wchodzą do publikacji.
- W śledzonym drzewie roboczym nie znaleziono danych logowania, tokenów, JWT ani kluczy prywatnych.

## Weryfikacja

| Komponent | Polecenie | Wynik |
| --- | --- | --- |
| dsh-mobile | `dart format --set-exit-if-changed`, `flutter analyze`, `flutter test` | formatowanie i analiza bez błędów; 63 testów zaliczonych |
| dsh-plugin | `npm run build` + `npm test` | build OK, 38 testów zaliczonych |
| dsh-relay | `npm run build` + `npm test` | build OK, 31 testów zaliczonych |
| dsh-website | `npm run check` | Astro, ESLint i Prettier bez błędów |
| dsh-website | `npm run build` + `npm run check:locales` | zbudowano 34 strony; sprawdzono 3 kształty treści, 23 zlokalizowane wyniki i 1056 lokalnych linków |

Zrzuty pochodzą z kompilacji debug zainstalowanej na telefonie. Poprawka okna zmiany nazwy jest pokryta testem widgetu; w tym workspace nie zbudowano nowego podpisanego APK.

## Stan wydania i pozostałe kroki

- Cztery repozytoria komponentów zawierają zmiany na lokalnej gałęzi `release/localization`, przebazowanej na bieżący `origin/main`.
- Wskaźniki podmodułów w tym commicie wskazują na commity tych komponentów.
- Nie można jeszcze utworzyć wspólnego tagu wydania: Mobile ma `0.1.5`, a Plugin i Relay `0.1.9`. Workflow wydania odrzuci tag, dopóki wszystkie trzy wersje nie będą zgodne. Przed tagowaniem wyrównaj wersje (na przykład `0.1.10`).
- Z tego workspace nie wykonano pushu komponentów, strony ani zestawu. Najpierw wypchnij gałęzie komponentów, potem gałąź zestawu; inaczej CI nie odnajdzie commitów podmodułów.
- Wdrożenie strony i podpisanie wydania Androida pozostają krokami wydaniowymi.

```bash
git -C dsh-mobile push origin release/localization
git -C dsh-plugin push origin release/localization
git -C dsh-relay push origin release/localization
git -C dsh-website push origin release/localization
git push origin main
```
