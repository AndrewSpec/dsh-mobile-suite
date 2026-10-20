# DSH Mobile Suite

[English](README.md) | [Polski](README.pl.md) | [简体中文](README.zh-CN.md)

Otwórz na telefonie z Androidem interfejs WWW DeepSeek Harness działający na Twoim komputerze, a następnie twórz zadania i wysyłaj polecenia w znanym interfejsie DSH.

> **Nieoficjalny projekt społecznościowy:** projekt jest niezależnie rozwijany i utrzymywany przez społeczność. Nie jest weryfikowany, rekomendowany ani wspierany przez DeepSeek. Obecne wydania Companion i Relay zapewniają szyfrowanie treści sesji DSH od końca do końca między aplikacją mobilną, klientami przeglądarkowymi a Companion.

Wątek społeczności DeepSeek Harness: [Show Your Plugins! #2520](https://github.com/deepseek-ai/deepseek-harness/discussions/2520). To miejsce prezentacji projektów społecznościowych, a nie oficjalna recenzja lub rekomendacja.

<table>
  <tr>
    <td><img src="docs/images/mobile-login.png" alt="Logowanie i wybór Relay w DSH Mobile" width="320"></td>
    <td><img src="docs/images/mobile-devices.png" alt="Sparowane komputery DSH" width="320"></td>
  </tr>
</table>

## Języki interfejsu

Aplikacja mobilna, ustawienia Companion i zdalny menedżer plików, strony Relay oraz publiczna strona internetowa obsługują angielski, polski i chiński uproszczony. Każda powierzchnia domyślnie podąża za językiem telefonu lub przeglądarki, wraca do angielskiego i udostępnia widoczny wybór języka. Nazwy komputerów, ścieżki plików, adresy URL, identyfikatory protokołu i kody błędów nie są tłumaczone.

Zobacz [opis zmian według modułów](docs/CHANGES.pl.md): tłumaczenia, poprawkę okna zmiany nazwy, zrzuty z telefonu i przegląd prywatności.

## Składniki

To repozytorium przypina cztery niezależnie rozwijane i wydawane komponenty open source jako podmoduły Git:

| Katalog | Przeznaczenie | Repozytorium |
| --- | --- | --- |
| `dsh-mobile/` | Klient Flutter na Androida i iOS | [april-jk/dsh-mobile](https://github.com/april-jk/dsh-mobile) |
| `dsh-plugin/` | Wtyczka DSH i Companion działające na komputerze | [april-jk/dsh-mobile-plugin](https://github.com/april-jk/dsh-mobile-plugin) |
| `dsh-relay/` | Konta, parowanie, krótkotrwałe bilety dostępu i przekazywanie ruchu | [april-jk/dsh-relay](https://github.com/april-jk/dsh-relay) |
| `dsh-website/` | Publiczna strona, treści SEO i wdrożenie w GitHub Pages | [april-jk/dsh-mobile-site](https://github.com/april-jk/dsh-mobile-site) |

Telefon nigdy nie łączy się bezpośrednio z komputerem. Wtyczka nawiązuje wyłącznie wychodzące połączenie WSS z Relay, a DSH nadal nasłuchuje na `127.0.0.1:3080`.

Domyślny publiczny Relay: `https://relay.dshmobile.online`

## Instalacja i użytkowanie

Pobierz następujące pliki z [najnowszego wydania](https://github.com/april-jk/dsh-mobile-suite/releases/latest):

- `dsh-mobile-android.apk`: podpisany instalator na Androida;
- `dsh-mobile-plugin.tgz`: gotowy pakiet wtyczki DSH;
- `dsh-relay-v*.tar.gz`: pakiet źródłowy Relay do samodzielnego wdrożenia;
- `SHA256SUMS`: sumy kontrolne SHA-256 wszystkich plików wydania.

Sprawdź pobrane pliki:

```bash
shasum -a 256 -c SHA256SUMS
```

Zainstaluj przypiętą wersję wtyczki bezpośrednio z GitHuba, a następnie uruchom DSH. Nie wymaga to globalnej instalacji DSH, pobrania kodu źródłowego ani podawania lokalnej ścieżki:

```bash
npx @deepseek-ai/dsh plugin --profile web add "github:april-jk/dsh-mobile-plugin#v0.1.7"
npx @deepseek-ai/dsh web
```

Zainstaluj `dsh-mobile-android.apk` na urządzeniu z Androidem. System może poprosić o zezwolenie przeglądarce lub menedżerowi plików na instalowanie nieznanych aplikacji. Identyfikator aplikacji to `io.github.apriljk.dshremote`.

## Parowanie i uruchamianie zadań zdalnych

1. Zarejestruj się lub zaloguj w aplikacji mobilnej.
2. Na komputerze otwórz w DSH **Settings > Remote Access** i wygeneruj zaszyfrowany kod QR parowania.
3. Naciśnij **+** na liście komputerów w aplikacji i zeskanuj kod QR. Parowanie samym sześciocyfrowym kodem jest wyłączone, ponieważ nie pozwala ustanowić klucza szyfrowania.
4. Wybierz komputer dostępny online, aby otworzyć jego standardowy interfejs WWW DSH.
5. Utwórz zadanie i wyślij polecenia w dotychczasowym interfejsie DSH.

### Dodawanie kolejnej przeglądarki

Po sparowaniu komputera otwórz w DSH **Settings > Remote Access** i wybierz **Generate browser access code**. Otwórz odnośnik w kolejnej przeglądarce lub zeskanuj w niej kod QR i zaloguj się na to samo konto Relay. Każda przeglądarka przechowuje własny klucz E2EE, dzięki czemu telefon i kilka przeglądarek mogą pozostawać połączone jednocześnie.

Wtyczka uruchamia się i zatrzymuje wraz z procesem DSH Web; nie wymaga osobnego procesu w tle. Telefon pokazuje komputer jako niedostępny, gdy komputer lub DSH jest wyłączony albo wtyczka nie ma połączenia z Relay.

## Własny serwer Relay

Pobierz i rozpakuj `dsh-relay-v*.tar.gz` z wydania, a następnie wykonaj:

```bash
cp .env.example .env
# Edytuj .env i zastąp JWT_SECRET długą losową wartością.
docker compose up -d --build
curl http://127.0.0.1:8787/health
```

Przy dostępie publicznym umieść odwrotne proxy HTTPS przed portem `8787` i wykonuj kopie zapasowe danych SQLite z `/data`. Relay w wersji MVP musi działać jako pojedyncza instancja.

Komputer i telefon muszą korzystać z tego samego Relay:

```bash
DSH_RELAY=https://relay.example.com npx @deepseek-ai/dsh web
```

Na ekranie logowania telefonu naciśnij **Relay** albo po zalogowaniu przejdź do ustawień serwera Relay i wpisz ten sam adres HTTPS. Zmiana Relay wylogowuje bieżące konto, ponieważ konta i tokeny są odizolowane między instancjami Relay.

Wszystkie zmienne środowiskowe i limity zasobów opisuje [polski README Relay](dsh-relay/README.pl.md).

## Granice bezpieczeństwa

- Ruch publiczny musi korzystać z HTTPS/WSS; komputer nigdy nie otwiera publicznego portu nasłuchującego.
- Aplikacja mobilna otrzymuje tokeny konta i krótkotrwałe bilety dostępu do WWW, ale nigdy poświadczenia urządzenia komputera.
- Treść HTTP, SSE i WebSocket DSH jest szyfrowana od końca do końca między aplikacją mobilną a Companion. Relay przekazuje nieprzezroczyste ramki i nie otrzymuje klucza treści.
- Relay nadal widzi powiązania kont i urządzeń, stan dostępności, czas połączenia, długość szyfrogramów i charakterystykę czasową ruchu. Obecny mechanizm kluczy przekazywanych kodem QR nie zapewnia utajniania z wyprzedzeniem (forward secrecy).
- Relay przechowuje konta, urządzenia, stan parowania i ograniczone metadane dziennika dostępu. Przed dystrybucją w sklepach nadal trzeba zapewnić usuwanie kont, politykę prywatności i pozostałe wymagania zgodności.

Problemy bezpieczeństwa zgłaszaj prywatnie zgodnie z [polityką bezpieczeństwa](SECURITY.pl.md). Nie publikuj szczegółów umożliwiających wykorzystanie podatności w publicznym zgłoszeniu.

## Rozwój projektu

```bash
git clone --recurse-submodules https://github.com/april-jk/dsh-mobile-suite.git
cd dsh-mobile-suite
git submodule update --init --recursive
```

Zmiany kodu komponentów zatwierdzaj w ich własnych repozytoriach. Repozytorium nadrzędne utrzymuje dokumentację międzykomponentową, wspólny proces wydań i przetestowane rewizje komponentów. Zobacz [zasady współpracy](CONTRIBUTING.pl.md).

Każde wypchnięcie zmian i każdy pull request do `main` uruchamiają kompilację i testy komponentów, sprawdzają zatwierdzony pakiet wtyczki, budują kontener Relay oraz APK na Androida i weryfikują publiczną stronę. Aby opublikować wydanie całego pakietu, najpierw wypchnij zgodne tagi komponentów, zaktualizuj przypięte podmoduły i dopiero wtedy wypchnij taki sam tag w tym repozytorium. Proces wydania odrzuca tag niezgodny z wersjami pakietów Plugin, Relay i Mobile przed przygotowaniem podpisanych plików oraz sum kontrolnych.

## Dokumentacja po polsku

- [Aplikacja mobilna](dsh-mobile/README.pl.md)
- [Wtyczka i Companion](dsh-plugin/README.pl.md)
- [Relay](dsh-relay/README.pl.md)
- [Strona internetowa](dsh-website/README.pl.md)
- [Wspólne kontrakty i integracja](dsh-公共文档/README.pl.md)
- [Historia zmian](CHANGELOG.pl.md)

Tłumaczenia nie zmieniają nazw protokołów, identyfikatorów ani danych użytkownika. Interfejs DSH ładowany z komputera jest osobnym projektem i zachowuje własne ustawienia języka. Zrzuty w [opisie zmian](docs/CHANGES.pl.md) pokazują polski, angielski i chiński interfejs aplikacji na telefonie; zrzuty sesji DSH nadal przedstawiają oryginalny interfejs Harness.

## Licencja

Pakiet i wszystkie cztery komponenty są udostępniane na [licencji MIT](LICENSE).
