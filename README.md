<div align="center">

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/github/banner-dark.png">
  <img src="assets/github/banner-light.png" alt="Carafe" width="447">
</picture>

**Windows-Spiele für die Nintendo Switch verpacken – unter CachyOS / Arch Linux**

[![Plattform](https://img.shields.io/badge/Plattform-CachyOS%20%2F%20Arch%20x86__64-blue)](#installation-auf-cachyos)
[![Status](https://img.shields.io/badge/Status-experimentell-orange)](#aktueller-stand)
[![Lizenz](https://img.shields.io/badge/Lizenz-GPL--3.0-green)](LICENSE)

</div>

Dieser Fork von [Carafe](https://github.com/carafe-nx/carafe) ergänzt Linux-Unterstützung
mit Schwerpunkt **CachyOS auf x86_64**. Carafe verpackt den vollständigen Ordner eines
installierten Windows-Spiels zusammen mit einer Switch-Runtime und deinen Einstellungen
in eine `.nsp`-Datei. Nach der Installation erscheint das Spiel im HOME-Menü der Switch.

„Konvertieren“ bedeutet hier **Verpacken mit einer Kompatibilitäts-Runtime**: Der
Windows-Programmcode wird nicht in einen nativen Switch-Port umgeschrieben. Die Runtime
basiert auf [Autorun](https://github.com/autorunhq/autorun), Wine, FEX/Box64 und DXVK/Mesa.
Ein erfolgreich erstelltes NSP garantiert deshalb noch nicht, dass das Spiel läuft.

## Aktueller Stand

- Die Linux- und CachyOS-Anpassungen liegen im Branch `feat/linux-packaging`, siehe [PR #1](https://github.com/nongrips/carafe-Linux/pull/1).
- Enthalten sind ein nativer Linux-Packer, eine Arch-Paketdefinition und ein Linux-Build-Workflow.
- Syntax und Paketstruktur wurden geprüft. **Ein vollständiger Build und Starttest auf CachyOS sowie ein NSP-Praxistest stehen noch aus.**
- Unter Linux funktionieren die direkte USB-Installation und das Abrufen von Switch-Logs noch nicht. Installiere die erzeugten NSPs separat mit DBI.
- Automatische App-Updates sind unter Linux deaktiviert. Aktualisiere vorerst durch erneutes Bauen des Pakets.
- Ein fertiges, getestetes CachyOS-Paket wird hier noch nicht angeboten. Die `.exe`-Installer des Upstream-Projekts sind Windows-Versionen.

## Voraussetzungen

- CachyOS / Arch Linux, x86_64, vollständig aktualisiert.
- Node.js 24 oder neuer, Rust mindestens 1.90, Git und Docker mit Compose.
- Ausreichend freier Speicher für Docker-Images, Runtime-Build, Spielkopie und fertige NSPs.
  Der benötigte Platz hängt vom Spiel ab; plane deutlich mehr als die reine Spielgröße ein.
- Eine Nintendo Switch mit eingerichteter Atmosphère-CFW und Unterstützung für die Installation eigener NSPs.
- [DBI](https://github.com/rashevskyv/dbi) auf der Switch.
- Deine eigene `prod.keys` für das Packen. Carafe prüft unter anderem `header_key` und
  `key_area_key_application_00`. Schlüssel sind nicht im Repository enthalten.
- Eine vollständige, bereits installierte Windows-Version eines Spiels, das du besitzt.

## Installation auf CachyOS

### 1. System und Build-Abhängigkeiten installieren

```sh
sudo pacman -Syu --needed base-devel git rust nodejs npm docker docker-compose \
  webkit2gtk-4.1 gtk3 libayatana-appindicator openssl xdg-utils
sudo systemctl enable --now docker
```

Falls du Rust bereits über rustup verwaltest, verwende diese Installation anstelle eines
zweiten Rust-Pakets. Prüfe anschließend die Versionen:

```sh
node --version
rustc --version
docker compose version
docker info
```

`docker info` muss als dein normaler Benutzer funktionieren. Falls dafür die Berechtigung fehlt:

```sh
sudo usermod -aG docker "$USER"
```

Danach vollständig abmelden und wieder anmelden und `docker info` erneut prüfen.
Die Docker-Gruppe gewährt rootähnliche Rechte. `makepkg` und `npm` führst du als normaler
Benutzer aus; das Paket wird am Ende über `sudo pacman` installiert.

### 2. Linux-Branch holen und Paket bauen

```sh
git clone --branch feat/linux-packaging https://github.com/nongrips/carafe-Linux.git
cd carafe-Linux/packaging/cachyos
makepkg -si
```

`makepkg` lädt den Quellcode des Linux-Branches, baut die Switch-Runtime und den Packer in
Docker und kompiliert die Oberfläche auf deinem CachyOS-System. Der erste Build kann lange
dauern. Bestätige die Installation des erzeugten Pakets, wenn `makepkg` danach fragt.

Ergebnis: `carafe-git-*.pkg.tar.zst` im aktuellen Verzeichnis und ein Menüeintrag **Carafe**.
Ein vorhandenes Paket lässt sich auch mit `sudo pacman -U ./carafe-git-*.pkg.tar.zst` installieren.
Es werden weder FUSE noch Wine auf dem Host benötigt, um die installierte Carafe-App auszuführen.

### 3. Carafe starten

Starte **Carafe** im Anwendungsmenü oder im Terminal:

```sh
carafe
```

Weitere Build-Optionen und Hinweise zu anderen Distributionen stehen in [docs/linux.md](docs/linux.md).

## Spiel konvertieren: Schritt für Schritt

Die folgenden Bezeichnungen entsprechen der deutschen Oberfläche. Stelle bei Bedarf unter
**Einstellungen → Sprache** auf Deutsch um.

### 1. Den richtigen Spielordner vorbereiten

Du brauchst das **installierte Windows-Spiel mit allen zugehörigen Dateien**. Eine einzelne
`.exe`, ein Downloadarchiv oder `setup.exe` genügt nicht. Entpacke portable Spiele vollständig;
installiere andere Spiele zunächst über den jeweiligen Installer. Dafür kann auf deinem PC
separat Wine/Proton nötig sein – Carafe führt keinen Spielinstaller aus.

Ein geeigneter Ordner sieht beispielsweise so aus:

```text
/home/deinname/Games/MeinSpiel/
├── game.exe
├── data/
├── audio/
└── weitere DLLs und Spieldateien
```

Die eigentliche Startdatei kann auch in einem Unterordner wie `bin/` liegen. Wähle trotzdem
den gemeinsamen Spielordner, damit alle benötigten Daten mitgepackt werden.
Bei einer Proton-Installation wählst du den Spielordner, nicht den gesamten Proton-Prefix
mit `drive_c`, Benutzerprofilen und anderen Programmen. Eine Linux-ELF-Datei oder ein
`.sh`-Launcher ist keine geeignete Startdatei.

Am besten beginnt man mit einem kleinen, älteren 32-Bit-Spiel, das ohne zusätzlichen
Launcher auskommt. Spiele mit Steam-Abhängigkeit, DRM, Anti-Cheat oder externen Diensten
können trotz erfolgreichem Build scheitern. Ein erfolgreicher Start mit Proton auf CachyOS
ist keine Garantie für die Switch-Runtime.

### 2. Carafe einmalig einrichten

1. Starte Carafe und gehe im Einrichtungsassistenten zu **Konsolenschlüssel**.
2. Wähle über **Durchsuchen…** deine Datei `prod.keys` aus. Fahre erst fort, wenn die
   benötigten Schlüssel erkannt wurden. Veröffentliche die Datei nicht.
3. Wähle bei **Bibliotheksordner** einen beschreibbaren Ordner auf deinem PC.
   Carafe ergänzt den Unterordner `Carafe`, sofern der ausgewählte Ordner nicht bereits so heißt.
   Wählst du beispielsweise `/home/deinname/Spiele-NSP`, landen die Dateien unter
   `/home/deinname/Spiele-NSP/Carafe`.
4. Lies die Hinweise, bestätige **Verstanden** und beende die Einrichtung mit **Los geht’s**.

Schlüsseldatei und Bibliotheksordner kannst du später unter **Einstellungen → Dateien** ändern.

### 3. Neues Spiel und Ordner auswählen

1. Klicke in der **Bibliothek** auf **Neues Spiel**.
2. Klicke im Schritt **Ordner** auf **Spielordner wählen**.
3. Wähle den vorbereiteten Ordner, beispielsweise `/home/deinname/Games/MeinSpiel`.
4. Warte, bis Carafe die `.exe`-Dateien gelesen hat, und klicke auf **Weiter**.

Findet Carafe nur Installer oder keine passende `.exe`, prüfe zunächst den gewählten Ordner.

### 4. Die Startdatei festlegen

1. Wähle im Schritt **Start** die `.exe`, die das Spiel tatsächlich startet,
   beispielsweise `game.exe` oder `bin/game.exe`.
2. Kontrolliere den Vorschlag von Carafe. Wähle keinen Updater, Uninstaller oder `setup.exe`.
3. Lass **Startargumente** zunächst leer, sofern das Spiel keine bestimmten Argumente benötigt.
4. Lass unter **Erweitert → Bit-Version** zunächst **Auto** aktiv.
5. Lies angezeigte Warnungen und gehe mit **Weiter** zum nächsten Schritt.

64-Bit-Spiele sind in der aktuellen Runtime noch nicht stabil. Falls dein Spiel eine
32-Bit-Version anbietet, verwende diese. Eine 64-Bit-EXE wird durch die manuelle Auswahl
von 32 Bit nicht zu einem 32-Bit-Programm.

### 5. Titel und Icon wählen

Im Schritt **Aussehen**:

1. Prüfe den **Titel**, der im HOME-Menü erscheinen soll.
2. Ergänze bei Bedarf **Publisher** und **Version**. Die Version darf leer bleiben;
   Carafe verwendet dann die Build-Nummer.
3. Wähle ein Icon: **Aus .exe**, **Eigenes Bild** oder **SteamGridDB**.
4. Bei einem eigenen Bild: Datei auswählen und den quadratischen Ausschnitt festlegen.
5. Prüfe die Vorschau und klicke auf **Weiter**.

Für SteamGridDB brauchst du einen eigenen API-Schlüssel unter **Einstellungen → Dateien**.
Die Optionen **Aus .exe** und **Eigenes Bild** benötigen diesen Dienst nicht.

### 6. Steuerung und Grafik einstellen

Starte im Schritt **Einstellungen** mit den empfohlenen Werten bzw. der Voreinstellung
**Ausgewogen**. Ändere für den ersten Versuch möglichst wenig:

- **Steuerung:** Controller-Modus für Spiele mit Gamepad-Unterstützung; bei reiner
  Tastatur-/Maussteuerung die passenden Switch-Tasten zuordnen.
- **Grafik / DirectX:** Zunächst die Vorgaben belassen. Ein Spiel mit niedriger Auflösung
  und reduzierten Details ist für die Switch ein sinnvoller erster Test.
- **Prozessor / System:** FEX und die empfohlenen Vorgaben zuerst testen. Box64 ist eine
  Alternative für 32-Bit-Spiele, falls der Start mit FEX scheitert.

Feineinstellungen ersetzen keine fehlenden Spieldateien oder nicht unterstützte Laufzeitabhängigkeiten.
Klicke danach auf **Weiter**.

### 7. NSP erstellen und finden

1. Kontrolliere im Schritt **Erstellen** Spiel, Startdatei, Title ID und Steuerung.
2. Klicke auf **Erstellen** und warte auf den Abschluss. Carafe prüft die Runtime,
   kopiert das Spiel, schreibt Metadaten und verpackt alles.
3. Bei Problemen lies das **Build-Log**. Ein abgebrochener oder fehlgeschlagener Build
   ist kein fertiges NSP.
4. Nach **Fertig. Das Spiel ist in deiner Bibliothek.** klicke auf **Bibliothek öffnen**.
5. Öffne deinen konfigurierten Bibliotheksordner im Dateimanager.

Die fertige Datei heißt **`<Title-ID>.nsp`**, nicht zwingend wie der Spieltitel.
Beispiel: `056694dd13640000.nsp` – die konkrete ID deines Spiels steht in Carafe.
Die Einstellungen werden im NSP mitgespeichert. Temporäre `.nsp.part`-Dateien und der
versteckte Ordner `.carafe` sind nicht zur Installation bestimmt.

### 8. Auf der Switch installieren und testen

**Der Button „Auf Switch installieren“ und „Logs abrufen“ funktionieren in diesem Linux-Fork
noch nicht.** Verwende DBI unabhängig von Carafe.

Ein möglicher Weg ist die Installation von der microSD-Karte:

1. Kopiere das fertige NSP mit deiner üblichen Methode in einen Ordner auf der microSD-Karte.
   Beende Dateizugriffe und wirf die Karte bzw. USB-Verbindung sauber aus.
2. Starte die Switch in ihrer eingerichteten CFW und öffne DBI.
3. Öffne in DBI **Browse SD Card**, navigiere zur NSP-Datei und wähle die Installation.
   Menütexte können je nach DBI-Version und Sprache abweichen.
4. Wähle den gewünschten Installationsort und warte, bis DBI erfolgreich fertig ist.
5. Kehre ins HOME-Menü zurück und starte das Spiel.

Auf einer FAT32-Karte kann eine einzelne Datei nicht 4 GiB oder größer sein. Bei größeren
NSPs verwende einen separat eingerichteten USB-/Netzwerk-Installationsweg, den deine
DBI-Version unterstützt; Carafe stellt dafür derzeit keinen Linux-Transfer bereit.
Das bloße Kopieren eines NSPs auf die Karte installiert es noch nicht.

### 9. Einstellungen ändern und erneut erstellen

Wähle das Spiel in der Bibliothek und klicke auf **Neu erstellen**. Passe die Einstellungen
an, erstelle das NSP erneut und installiere die neue Datei mit DBI über die vorhandene Version.
Beim Neuerstellen behält Carafe die Title ID bei; dadurch bleibt die Zuordnung der
Spielstände erhalten. Sichere wichtige Spielstände trotzdem vor Änderungen.

Bewahre den ursprünglichen Spielordner auf: Carafe braucht ihn beim Neuerstellen wieder.
Wenn du ihn verschoben hast, wähle im Assistenten seinen neuen Ort.

## Häufige Probleme

| Problem | Was du prüfen solltest |
| --- | --- |
| `docker info` meldet fehlende Berechtigung | Docker starten, Gruppenzugehörigkeit prüfen und nach der Änderung neu anmelden. |
| `makepkg` bricht ab | Die erste konkrete Fehlermeldung im Build-Log prüfen; der vollständige CachyOS-Build ist noch ungetestet. |
| Das Fenster öffnet sich unter Wayland nicht | Im Terminal `carafe` starten. Zum Vergleich mit installiertem XWayland `GDK_BACKEND=x11 carafe` versuchen. |
| Keine passende `.exe` gefunden | Installierten Windows-Spielordner statt Installer, Linux-Version oder Proton-Prefix wählen. |
| Schlüssel fehlen | Die ausgewählte eigene `prod.keys` prüfen; nicht mit einer anderen Konfigurationsdatei verwechseln. |
| Build bricht wegen Speicherplatz ab | Freien Platz sowohl für Docker als auch auf dem Laufwerk der Bibliothek prüfen. |
| Switch wird in Carafe nicht erkannt | Erwartete Einschränkung unter Linux: das fertige NSP separat mit DBI installieren. |
| NSP erstellt, Spiel startet aber nicht | Start-EXE, vollständige Daten, Bit-Version und mögliche Launcher-/DRM-Abhängigkeiten prüfen. Danach jeweils nur eine Runtime-Einstellung ändern und neu erstellen. |
| Geringe Bildrate oder Grafikfehler | Zuerst Details und Auflösung reduzieren; ältere und leichtere Spiele bevorzugen. Nicht jedes Windows-Spiel ist kompatibel. |

## Entwicklung und weitere Dokumentation

- [Linux-Builds und weitere Distributionen](docs/linux.md)
- [Beitragen und Projektstruktur](CONTRIBUTING.md)
- [Technische Dokumentation](docs/README.md)
- [Kompatibilitätsberichte im Upstream-Projekt](https://github.com/carafe-nx/carafe/discussions/categories/compatibility)

## Credits

Carafe stands on the work of others:

- [Autorun](https://github.com/autorunhq/autorun) — the Windows runtime for Horizon OS, with
  [Wine](https://www.winehq.org), [Box64](https://github.com/ptitSeb/box64), [FEX](https://github.com/FEX-Emu/FEX),
  [DXVK](https://github.com/doitsujin/dxvk) and [Mesa](https://www.mesa3d.org);
- [hacBrewPack](https://github.com/rlaphoenix/hacBrewPack) — builds the NSP;
- [nx-hbloader](https://github.com/switchbrew/nx-hbloader) and [devkitPro](https://devkitpro.org) — the loader
  and the Switch toolchain;
- [Tauri](https://tauri.app) — the desktop app.

## Legal

Carafe is made for games you own. It ships no firmware or games and does not help to obtain them.
Carafe is not affiliated with or endorsed by Nintendo.

## License

Carafe is [GPL-3.0-or-later](LICENSE). The runtime overlay and the Autorun patches are LGPL-2.1-or-later
([`runtime/overlay/LICENSE`](runtime/overlay/LICENSE), [`runtime/patches/LICENSE`](runtime/patches/LICENSE)),
the loader is ISC ([`runtime/loader/LICENSE`](runtime/loader/LICENSE)). Third-party code in `external/` keeps
its own licenses.
