# Współtworzenie projektu

[English](CONTRIBUTING.md) | [Polski](CONTRIBUTING.pl.md)

DSH Mobile Suite jest podzielony na niezależnie wersjonowane repozytoria. Propozycje zmian kodu kieruj do repozytorium odpowiedzialnego za dany komponent:

- Aplikacja mobilna: <https://github.com/april-jk/dsh-mobile>
- Wtyczka DSH: <https://github.com/april-jk/dsh-mobile-plugin>
- Relay: <https://github.com/april-jk/dsh-relay>
- Wspólne kontrakty i skoordynowane wydania: to repozytorium.

Pobierz repozytorium razem z podmodułami:

```bash
git clone --recurse-submodules https://github.com/april-jk/dsh-mobile-suite.git
```

Jeżeli zmiana zachowania wpływa na kontrakt między komponentami, zaktualizuj dokumentację publicznego kontraktu w repozytoriach odpowiednich komponentów przed zmianą kodu lub razem z nią. Zachowaj zgodność koperty tunelu `v: 1`, chyba że dołączono plan migracji.

Przed otwarciem pull requestu uruchom kontrole opisane w README zmienianego komponentu. Nie zatwierdzaj w Git poświadczeń, kluczy podpisywania Androida, plików `.env`, wygenerowanych baz SQLite ani tokenów Railway.

To nieoficjalny projekt społecznościowy. Wkład w projekt nie może sugerować weryfikacji, certyfikacji, rekomendacji ani wsparcia ze strony DeepSeek.
