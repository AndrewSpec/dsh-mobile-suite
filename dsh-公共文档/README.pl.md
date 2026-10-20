# Publiczne kontrakty DSH Remote

[English](README.md) | [Polski](README.pl.md)

Ten katalog jest źródłem prawdy integracji dla trzech niezależnie wersjonowanych aplikacji.

| Dokument | Odbiorca | Przeznaczenie |
| --- | --- | --- |
| [API-v1.md](API-v1.pl.md) | Mobile, Companion, Relay | Kontrakty HTTP dotyczące kont, parowania, urządzeń, biletów i WebView |
| [Tunnel-Protocol-v1.md](Tunnel-Protocol-v1.pl.md) | Companion, Relay | Uwierzytelnianie WSS, multipleksowane przekazywanie HTTP i WebSocket |
| [Tunnel-E2EE-v1.md](Tunnel-E2EE-v1.pl.md) | Mobile, Companion, Relay | Inicjalizacja zaufania przez QR, szyfrowane uzgadnianie, ramkowanie AEAD i polityka obniżania poziomu |
| [Browser-Remote-and-Relay-Admin-v1.md](Browser-Remote-and-Relay-Admin-v1.pl.md) | Browser, Companion, Relay | Dostęp z Safari, brama E2EE w przeglądarce i odizolowana administracja statystykami Relay |
| [Integration-Plan.md](Integration-Plan.pl.md) | Wszystkie zespoły | Lokalna sekwencja testów i sekwencja Railway, odpowiedzialność oraz warunki wydania |
| [DSH-远程MVP-跨端接口.md](DSH-Remote-MVP-Interfaces.pl.md) | Wszystkie zespoły | Zwięzłe podsumowanie kontraktu zgodne z implementacją |

Linki w tabeli prowadzą do tłumaczeń polskich; dokumenty angielskie pozostają źródłem nadrzędnym.

Następujące dokumenty historyczne nie są obecne w tym repozytorium i dlatego nie są linkami:

- 移动端-MVP-修改建议.md — historyczny dokument dla Mobile (nieobecny w tym checkoucie; dokument historyczny).
- WebUI-远程访问管理方案.md — historyczny dokument dla Plugin, Relay i Mobile (nieobecny w tym checkoucie; dokument historyczny).
- Mobile-WebView-Performance.md — historyczny dokument dla Companion, Relay i Mobile (nieobecny w tym checkoucie; dokument historyczny).

Baseline produktu pozostaje w katalogu `../Kimi_Agent_DeepSeek Harness 移动端/` (nieobecny w tym checkoucie; historyczny katalog źródłowy). Gdy szczegół implementacji nie został tam określony, dokumenty API i protokołu w tym katalogu jednoznacznie rozstrzygają decyzję MVP. Gdy przykłady się różnią, `API-v1.md` i `Tunnel-Protocol-v1.md` są źródłem nadrzędnym.

## Instalacja wtyczki komputerowej

Z niezmiennego tagu wydania GitHub:

```bash
dsh plugin --profile web add github:april-jk/dsh-mobile-plugin#v0.1.7
dsh web
```

Do lokalnego rozwoju zbuduj `dsh-plugin` i dodaj zamiast tego jego bezwzględną ścieżkę katalogu. Profile, w których wcześniej zainstalowano nazwę deweloperską, muszą najpierw uruchomić `dsh plugin --profile web remove dsh-mobile-remote-companion`, a następnie dodać bieżący pakiet. Zapobiega to jednoczesnemu uruchamianiu obu tożsamości pakietu.

Nazwa instalowanego pakietu to `@april-jk/dsh-mobile`; jego stabilny identyfikator Cordis i polecenie CLI pozostają `dsh-mobile`. Jego pakiet DSH uruchamia i zatrzymuje Companion wraz z `dsh web`, przypisuje wybór obszaru roboczego do przeglądarkowego selektora katalogów DSH i domyślnie przechowuje dane uwierzytelniające parowania w `~/.dsh-remote/config.json`. Projekt jest nieoficjalny i niezależnie utrzymywany przez społeczność.

Po sparowaniu komputera ustawienia DSH w Companion mogą wygenerować kod QR/link dostępu przeglądarkowego. Każda przeglądarka niezależnie przechowuje klucz E2EE komputera, więc telefon i wiele przeglądarek mogą równocześnie korzystać z tego samego komputera. Dodanie przeglądarki nie tworzy ani nie zastępuje urządzenia komputerowego w Relay.
