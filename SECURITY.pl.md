# Polityka bezpieczeństwa

[English](SECURITY.md) | [Polski](SECURITY.pl.md)

## Wspierane wersje

W fazie MVP poprawki bezpieczeństwa otrzymuje wyłącznie najnowsze wydanie w GitHub Releases.

## Zgłaszanie podatności

Podatności dotyczące skoordynowanego produktu zgłaszaj za pomocą [prywatnego zgłaszania podatności na GitHubie](https://github.com/april-jk/dsh-mobile-suite/security/advisories/new). Nie otwieraj publicznego zgłoszenia zawierającego poświadczenia, dane osobowe, szczegóły wykorzystania podatności ani działający przykład ataku.

Podaj komponent i wersję, model wdrożenia, kroki odtworzenia, oczekiwane skutki i proponowane środki zaradcze. Społeczność potwierdza otrzymanie zgłoszeń w miarę dostępnych możliwości.

Jeżeli podatność dotyczy tylko jednego komponentu, można też użyć jego repozytorium, o ile ma włączone prywatne zgłaszanie podatności.

## Ograniczenia MVP

Obecne klienty Mobile i Browser korzystają z profilu szyfrowania od końca do końca w warstwie aplikacji `sealed-tunnel-v1`. Relay przekazuje szyfrogramy, ale nadal może obserwować powiązania kont i urządzeń, stan dostępności, czas połączeń, długość szyfrogramów i charakterystykę czasową ruchu.

JavaScript klienta Browser dostarczany przez Relay jest częścią zaufanego punktu końcowego. Szyfrowanie E2EE przeglądarki chroni przed analizą danych przez warstwę przekazywania Relay i przypadkowym ich utrwaleniem, ale nie przed złośliwym wdrożeniem Relay podmieniającym kod klienta. Jeżeli host WWW nie mieści się w granicy zaufania, użyj osobno podpisanego klienta natywnego lub rozszerzenia przeglądarki.

## Obsługa poświadczeń

- Produkcyjne `JWT_SECRET`, `ADMIN_PASSWORD`, hasła podpisywania Androida, tokeny npm i poświadczenia Railway przechowuj wyłącznie na platformie wdrożeniowej lub w menedżerze sekretów GitHub Actions.
- Nigdy nie zatwierdzaj plików `.env*`, `android/key.properties`, kluczy podpisywania, profili aprowizacji, plików usług Firebase, baz SQLite ani konfiguracji Companion. Śledzone pliki `.env.example` i `key.properties.example` mogą zawierać wyłącznie wartości przykładowe.
- Poświadczenia Companion przechowuj w domyślnej lokalizacji `~/.dsh-remote/config.json`. Nie ustawiaj `DSH_REMOTE_CONFIG` na ścieżkę wewnątrz drzewa roboczego Git.
- Przed wypchnięciem wydania uruchom Gitleaks dla repozytorium nadrzędnego i każdego komponentu:

```bash
for repo in . dsh-mobile dsh-plugin dsh-relay dsh-website; do
  gitleaks git --redact "$repo"
done
```

Opublikowany wektor testowy E2EE o znanym wyniku zawiera deterministyczne dane testowe i może być wskazywany przez ogólne heurystyki wykrywania sekretów. Oceniaj wyniki według ścieżki i reguły; nie wyłączaj globalnie danej reguły.

Jeżeli prawdziwe poświadczenie trafi do repozytorium, natychmiast je unieważnij lub zmień, zanim przepiszesz historię Git. Usunięcie go w późniejszym commicie nie usuwa go z wcześniejszych commitów, forków, pamięci podręcznych ani dzienników CI.
