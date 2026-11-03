# Pelna Analiza Wszystkich PR-ow i Nieudanych Prob Naprawy Aktualizacji APK

> **Status:** AKTYWNA DOKUMENTACJA PORAZEK (BEZWZGLEDNY ZAKAZ DALSZEGO ZGADYWANIA W KODZIE).  
> **Modul:** Auto-aktualizacja APK (sideload GitHub Releases) dla `lucid_mind` (`com.kaclad.dreamlog`).  
> **Cel:** Udokumentowanie kazdej proby, kazdego PR, hipotez, zmian w kodzie i powodow, dla ktorych kazda z nich poniosla kleske.

---

## 1. Bledne Kolo Architektury (Podsumowanie Cyklu PR)

Analiza historii git wykazuje kuriozalne, zamkniete kolo hipotez miedzy PR-ami:
1. **PR #22** napisal reczny `StreamSubscription` na strumieniu Dio. Stwierdzono, ze to wisi.
2. **PR #34** stwierdzil: *"Reczny stream byl zly i wisial na keep-alive! Zastepujemy go sprawdzonym `_dio.download()`"*.
3. **PR #42** stwierdzil: *"Pobieranie `_dio.download()` wisi na 100% przez Fastly/AWS keep-alive! Wracamy do wlasnego pobieracza `ApkDownloader` i dodajemy `permission_handler`!"*.
4. **PR #45 (wczesna faza)** przywrocil `_dio.download()` w commit `443e4b7`, bo ktos uznal, ze `ApkDownloader` powodowal bledy.
5. **PR #45 (pozniejsza faza)** stwierdzil: *"Dio w ogole nie dziala, `open_filex` nie dziala, cache wewnetrzny nie dziala! Wprowadzamy `HttpClient`, sciezki zewnetrzne, natywny Kotlin, a potem `android_package_installer`"*.
6. **Wersja v0.1.7-beta.12** — po wdrozeniu wszystkich powyzszych zmian problem NADAL wystepuje w dokladnie tej samej postaci.

---

## 2. Szczegolowa Anatomia Kazdego PR

### PR #22: `fix(update): robust stream download, non-blocking dialog and installer resilience (v0.1.5)`
- **Data i commit:** 13.10.2026 (`f4b362c`)
- **Autor i opis:** Przeniesienie pobierania w tle, obsluga cyklu zycia dialogu, uzycie recznego nasluchiwania na strumieniu odpowiedzi HTTP.
- **Zalozona hipoteza:**
  Aplikacja zawieszala sie lub byla ubijana w tle przez brak `WAKE_LOCK` oraz zamkniecie aplikacji przed uruchomieniem instalatora.
- **Wprowadzone zmiany:**
  - W `UpdateService.downloadApk` uzyto `_dio.get<ResponseBody>(..., responseType: ResponseType.stream)`.
  - Reczne podpiecie `response.data!.stream.listen(..., onDone: () { ... })` z `Completer<void>`.
  - Plik zapisywany do `getTemporaryDirectory()/apk_downloads/update.apk`.
  - Wywolanie instalatora przez `OpenFilex.open()`.
- **Dlaczego to sie nie udalo:**
  1. Serwery AWS S3 / GitHub CDN przy naglowku `Connection: keep-alive` nie emituja zdarzenia `onDone` na strumieniu Dio az do uplyniecia pelnego `receiveTimeout` (10 minut). `completer.future` nigdy sie nie rozwiazywal.
  2. Sciezka `getTemporaryDirectory()` na Androidzie 11+ znajduje sie w `/data/user/0/com.kaclad.dreamlog/cache`, do ktorej systemowy proces instalatora nie ma uprawnien odczytu (blokada SELinux).

---

### PR #34: `fix(update): resolve 100% download hang and decouple dialog content`
- **Data:** 14.10.2026
- **Zalozona hipoteza:**
  Reczny `StreamSubscription` z PR #22 byl wadliwy, a rozwiazaniem ma byc standardowa, wbudowana metoda biblioteki Dio: `_dio.download()`.
- **Wprowadzone zmiany:**
  - Calkowite usuniecie recznego strumienia i `Completer`.
  - Wstawienie `await _dio.download(url, savePath, onReceiveProgress: ...)`.
  - Dodanie weryfikacji niepustego pliku: `file.existsSync() && file.lengthSync() > 0`.
  - Podzial dialogu UI na `UpdateAvailableDialog` i `UpdateDialogContent`.
- **Dlaczego to sie nie udalo:**
  1. `_dio.download()` pod spodem w implementacji `IOHttpClientAdapter` robi dokladnie to samo: czeka na zakonczenie strumienia (`asFuture()`). Poniewaz polaczenie TCP z S3 nie wysylalo EOF, `_dio.download()` blokowalo asynchroniczny watek na dokladnie tych samych 10 minutach.
  2. Dialog wyswietlal 100%, poniewaz callback `onReceiveProgress` raportowal `received == total`, ale linijka `await _dio.download()` nigdy nie przeszla dalej.

---

### PR #42: `fix(update): resolve 100% download hang, background notifications, and release v0.1.7-beta.2`
- **Data:** 14.10.2026
- **Zalozona hipoteza:**
  1. Pobieranie wisi przez keep-alive, wiec nalezy natychmiast anulowac socket w momencie gdy `received == totalBytes`.
  2. Instalator nie startuje, poniewaz na Androidzie 8.0+ brakuje uprawnienia `REQUEST_INSTALL_PACKAGES`.
- **Wprowadzone zmiany:**
  - Utworzono modul `ApkDownloader`.
  - Dodano `Permission.requestInstallPackages.request()` z paczki `permission_handler` przed `OpenFilex.open()`.
  - Dodano uprawnienia `WAKE_LOCK` i `POST_NOTIFICATIONS`.
- **Dlaczego to sie nie udalo:**
  1. Na Androidzie 11+ wywolanie `Permission.requestInstallPackages.request()` nie otwiera systemowego okna dialogowego w aplikacji. Zamiast tego wyrzuca uzytkownika do glebokich ustawien systemowych, a w kodzie Fluttera zwraca natychmiast `PermissionStatus.denied`. Kod interpretowal to jako blad krytyczny i natychmiast przechodzil do `UpdateError`.
  2. Jesli uzytkownik nie mial tego uprawnienia wlaczonego z gory, instalacja byla natychmiast przerywana.
  3. Plik nadal byl pobierany do prywatnego cache (`getTemporaryDirectory()`), co uniemozliwialo instalatorowi odczyt pliku nawet po przyznaniu uprawnienia.

---

### PR #45: `fix(update): APK installer stall at 100% on Android 11+`
- **Data i galezie:** 16.10.2026 – 17.10.2026 (`fix/apk-installer-android11-sandbox`)
- **Zalozone hipotezy i etapy:**
  - **Etap A (commit `e5a48bb`):** Zmiana lokalizacji pliku na pamiec zewnetrzna (`getExternalStorageDirectory()`, `getExternalCacheDirectories()`) oraz dopisanie `<external-cache-path>` do `file_provider_paths.xml`.
    *Wynik:* Nie pomoglo. Dalej stalo na 100%.
  - **Etap B (commit `75c3cfc`):** Podejrzenie, ze biblioteka `open_filex` jest zepsuta i gubi intencje na Samsung OneUI. Zastapiono ja wlasnym `MethodChannel` w Kotlin (`MainActivity.kt`) z bezposrednim `startActivity(Intent(ACTION_VIEW))` i petla `grantUriPermission`.
    *Wynik:* Nie pomoglo. Uzytkownik zglosil: *"nadal staje na 100% nie mam zadnego zapytania o instalowanie z nieznanego zrodla... nic"*.
  - **Etap C (commit `443e4b7`):** Dodanie sprawdzania `packageManager.canRequestPackageInstalls()` i przekierowania do `Settings.ACTION_MANAGE_UNKNOWN_APP_SOURCES`.
    *Wynik:* Nie pomoglo, bo w kompilacji beta.10 i beta.11 nadal dzialalo `dio.download`, ktore wisialo w nieskonczonosc zanim kod instalatora w ogole zostal wywolany!
  - **Etap D (commit `d15afca`, wersja v0.1.7-beta.12):**
    - Przepisano pobieranie na `dart:io HttpClient.pipe(sink)`.
    - Dodano biblioteke `android_package_installer: ^0.0.3` (mechanizm z aplikacji Obtainium uzywajacy systemowego API `PackageInstaller.Session`).
    - Dodano deklaracje `<queries>` dla intencji instalatora w `AndroidManifest.xml`.
    *Wynik:* Uzytkownik zainstalowal `v0.1.7-beta.12`, sprobowal zaktualizowac do `v0.1.7-beta.13` i zglosil: *"nadal kurwa to samo, nie dziala"*.

---

## 3. Dlaczego Wszystkie Zmiany Ponosza Porazke (Ujawnione Czynniki Krytyczne)

### Czynnik 1: Brak Rzeczywistej Telemetrii (Zgadywanie w Ciemno)
Nigdy nie pobrano logow z urzadzenia uzytkownika (`adb logcat`, zrzut wyjatkow, stan watkow). Wszystkie PR-y opieraly sie na dedukcji zza biurka. Nie wiemy, czy wiesza sie:
- watek sieciowy,
- zapis i zamykanie pliku na partycji pamieci flash,
- wywolanie kanalu platformowego `MethodChannel`,
- czy moze instalator jest wywolywany, ale Samsung Knox / Google Play Protect zabija intencje w ciszy.

### Czynnik 2: Iluzja UI "100%"
Stan UI we Flutterze:
```dart
onProgress: (p) {
  if (p == 1.0) state = UpdateDownloading(progress: 1.0);
}
```
Pasek postepu osiaga 100% ZANIM plik zostanie fizycznie sflushowany na dysk, zanim zostanie sprawdzony rozmiar, zanim zostanie sprawdzone uprawnienie i zanim zostanie uruchomiony instalator. Jesli DOWOLNA z tych operacji rzuci niewylapany wyjatek lub zawiśnie w `await`, uzytkownik widzi tylko zamrozony pasek 100%.

### Czynnik 3: Potencjalna Pula Problemow z `android_package_installer` w Beta 12
W bibliotece `android_package_installer`:
```kotlin
val intent = Intent(context, activity!!.javaClass)
intent.action = packageInstalledAction
val pendingIntent = PendingIntent.getActivity(context, 0, intent, PendingIntent.FLAG_MUTABLE)
session.commit(pendingIntent.intentSender)
```
- `session.commit()` wysyla intencje do tej samej aktywnosci. Jesli aplikacja jest na wierzchu, na niektorych wersjach Androida (Android 12–15) `onNewIntent` NIE zostaje wywolane bez flagi `FLAG_ACTIVITY_SINGLE_TOP` w intencji.
- W rezultacie `OnNewIntentListener` nigdy nie odpala `startActivity(confirmIntent)`, a `callResult` w Kotlin nigdy nie zwraca sukcesu ani bledu do Dart.
- Dart czeka na `await AndroidPackageInstaller.installApk(...)` w nieskonczonosc!

### Czynnik 4: Ograniczenia OEM (Samsung Knox, Restrykcje Android 13+)
- Na urzadzeniach Samsung OneUI aplikacje pobrane poza Sklepem Play lub z nieznanych zrodel moga miec nalozona flage "Ograniczone ustawienia" (*Restricted settings*). Wtedy uzytkownik nie moze nawet wlaczyc zgody na nieznane zrodla bez odblokowania menu pod 3 kropkami w informacjach o aplikacji.
- Jesli system cicho blokuje instalacje, zadna zmiana kodu Dart/Kotlin bez uprzedniej zgody uzytkownika nie wywola okna instalatora.

---

## 4. Wnioski i Zobowiazania
1. **Koniec zgadywania:** Nie bedzie zadnego kolejnego bezmyslnego PR-u ani podbijania wersji "na oslep".
2. **Koniec cichego UI:** Kazdy krok musi byc widoczny na ekranie (tekst: pobrano bajty, sprawdzanie pliku, sprawdzanie uprawnien, uruchamianie sesji instalatora).
3. **Koniecznosc zebrania danych:** Wywiad z uzytkownikiem o dokladny model telefonu, wersje Androida i dokladne zachowanie ekranu.

---

## 5. Wyniki Wywiadu z Uzytkownikiem (Grill-Me) i Przelom Diagnostyczny

W trakcie procedury `/grill-me` zebrano twarde fakty z fizycznego urzadzenia:
1. **Urzadzenie:** Samsung Galaxy, najnowszy One UI (Android 14, OneUI 6+).
2. **Funkcja Auto Blocker:** Wylaczona w ustawieniach systemowych.
3. **Stan aplikacji:** Aplikacja byla odinstalowana i zainstalowana na czysto (brak pre-autoryzacji uprawnien).
4. **Stan UI podczas zawieszenia:**
   - Dialog wisi na: `Pobieranie... 100%`.
   - Przycisk to nadal `Anuluj` (NIGDY nie przechodzi do `Instalowanie...`).
   - Przycisk `Anuluj` JEST AKTYWNY i reaguje na dotyk (watek glowny UI nie jest zamrozony).

### Ostateczny Techniczny Dowod Awarii w `v0.1.7-beta.12`
Fakt, ze przycisk to nadal `Anuluj`, a tekst to `Pobieranie... 100%`, dowodzi w 100%, ze kod:
1. Nigdy nie dotarl do metody `installApk(path)`.
2. Nigdy nie wykonal `state = UpdateInstalling(path)`.
3. Jest bezposrednio zablokowany wewnatrz `await ApkDownloader.download()`.

**Zrodlo bledu w `apk_downloader.dart` (`v0.1.7-beta.12`):**
Zastosowano `await response.map(...).pipe(sink)`.
Metoda `Stream.pipe(sink)` w Dart SDK jest czarna skrzynka, ktora bezwarunkowo czeka na zdarzenie `done` strumienia wejsciowego.
Gdy pobierany jest plik z AWS S3 / CloudFront przez polaczenie `Connection: keep-alive`:
- Wszystkie bajty (`totalBytes == 94180428`) zostaja odebrane i zapisane do pliku.
- Pasek postepu raportuje `1.0` (100%).
- Jednak serwer S3 NIE zamyka polaczenia TCP (pozostawia otwarty socket keep-alive).
- Na Androidzie `HttpClientResponse` nie emituja zdarzenia `done`, poniewaz gniazdo TCP jest otwarte.
- `await ... .pipe(sink)` czeka w nieskonczonosc na zamkniecie socketu przez serwer.
- Kod za `pipe(sink)` nigdy sie nie wykonuje, funkcja nigdy nie zwraca `savePath`, a UI wisi na 100% z przyciskiem `Anuluj`.

---

## 6. Proba w `v0.1.7-beta.14` (Reczna petla `await for` ze sprawdzeniem bajtow)
- **Data i wersja:** 17.10.2026, `v0.1.7-beta.14` -> aktualizacja do `v0.1.7-beta.15`.
- **Wprowadzona zmiana:**
  Zastapienie `pipe(sink)` przez jawne `await for (final chunk in response)` i `break` gdy `received >= total`, a nastepnie `sink.flush()`, `sink.close()` i `client.close(force: true)`.
  Dodano rowniez licznik bajtow `(89.8 MB / 89.8 MB)` w UI.
- **Wynik empiryczny z urzadzenia uzytkownika:**
  Pobrano dokladnie caly plik `89.8 MB / 89.8 MB (100%)`.
  Aplikacja NADAL zawiesila sie na 100% z aktywnym przyciskiem `Anuluj`.
- **Dlaczego to sie nie udalo:**
  1. W Dart SDK (`dart:io`):
     - Wyjscie z `await for` przez `break` nie jest natychmiastowe; Dart Language Specification nakazuje niejawne wywolanie `await subscription.cancel()`.
     - `HttpClientResponse` w Dart na Androidzie przy polaczeniach S3 keep-alive blokuje sie wewnatrz `subscription.cancel()` lub `sink.flush()`.
     - Jesli serwer zakonczyl nadawanie, a socket jest zawieszony w buforze OS, watek Dart nigdy nie przechodzi do linii za `await for`.
  2. **Ostateczny wniosek architektoniczny:**
     Pobieranie 90 MB pliku binarnego przez watek Dart (`dart:io HttpClient`, `Dio`, `StreamSubscription`) na Androidzie jest zrodlem permanentnych zakleszczen watkow sieciowych i I/O.
  3. **Wlasciwe rozwiazanie systemowe (Standard Morphe, ReVanced, NewPipe):**
     Zastapienie pobieracza Dart natywnym **`android.app.DownloadManager`**.
     Usluga systemowa Androida zarzadza pobieraniem bezposrednio w procesie systemowym (`com.android.providers.downloads`), zapisuje plik do `Download/`, wyswietla systemowy postep i po ukonczeniu wysyla broadcast `ACTION_DOWNLOAD_COMPLETE`, natychmiast uruchamiajac instalator.

---

## 7. Ostateczny Sukces: `android.app.DownloadManager` (v0.1.7-beta.16 -> v0.1.7-beta.17)
- **Data i wersja:** 17.10.2026, `v0.1.7-beta.16` -> sukces aktualizacji do `v0.1.7-beta.17`.
- **Status:** **ROZWIAZANE I POTWIERDZONE EMPIRYCZNIE NA FIZYCZNYM URZADZENIU SAMSUNG GALAXY (ANDROID 14 / ONEUI 6+).**
- **Wdrozone rozwiazanie:**
  1. **Natywny systemowy silnik pobierania:**
     Calkowite wycofanie pobierania w Dart. Uzycie `android.app.DownloadManager` w `MainActivity.kt`. Proces pobierania odbywa sie na poziomie jadra systemu Android, calkowicie odporny na usterki buforowania gniazd, S3 keep-alive czy uspienie watku aplikacji.
  2. **Orkiestracja i telemetria UI:**
     Flutter odpytuje `getDownloadStatus` co 350 ms z `DownloadManager.Query()`, raportujac realne megabajty i procenty.
  3. **Podwojna gwarancja uruchomienia instalatora:**
     - Dynamiczny `BroadcastReceiver` w Kotlin nasluchujacy na `DownloadManager.ACTION_DOWNLOAD_COMPLETE`.
     - Logika we Flutterze, ktora przy statusie `successful` weryfikuje plik i wywoluje `launchApkInstaller`.
  4. **Cykl zycia uprawnien instalacji (Samsung OneUI):**
     Gdy uzytkownik nie ma nadanego uprawnienia instalowania nieznanych aplikacji, aplikacja otwiera systemowe `Settings.ACTION_MANAGE_UNKNOWN_APP_SOURCES`. Po wlaczeniu suwaka i powrocie uzytkownika do aplikacji (`AppLifecycleState.resumed`), `UpdateNotifier.didChangeAppLifecycleState` natychmiast wznawia instalacje i uruchamia systemowy instalator pakietow.
  5. **Jednorazowosc uprawnienia:**
     Uprawnienie do instalowania nieznanych aplikacji jest przyznawane raz na instalacje pakietu. Przy kolejnych aktualizacjach monit o Ustawienia juz sie nie pojawi — instalator wystartuje od razu.


