# Fotofinder

**Fotofinder** ist eine Android-App, die das Wiederfinden von Fotos in großen Sammlungen einfach macht – komplett **auf dem Gerät** und **ohne Cloud**. Sie durchsucht die Galerie nach Stichwörtern/Tags, nach **Text im Bild** (OCR), nach **Aufnahmedatum** und nach **Aufnahmeort** (GPS) und lässt sich zugleich als vollwertige Galerie nutzen.

> Privatsphäre zuerst: Die gesamte Indexierung (OCR, GPS) passiert lokal. Es werden keine Bilder oder Daten an einen Server gesendet.

---

## Funktionen

- **Galerie-Start** – beim Öffnen werden alle Fotos angezeigt.
- **Suche**
  - Stichwörter / **Tags** (manuell vergeben, Mehrfachauswahl)
  - **Text im Bild (OCR)** mit Optionen „Ganzes Wort" und „Groß-/Kleinschreibung"
  - **Datumsbereich**
  - **Standort**: mit/ohne GPS, Umkreis um aktuellen Standort, einen Ort oder einen Kartenpunkt
- **Sortierung** nach Datum, Name oder Entfernung (auf-/absteigend)
- **Gruppierung passend zur Sortierung** mit Abschnitts-Überschriften und angehefteter Kopfzeile
  - Datum → Monat · Name → Herkunftsordner · Entfernung → Abstandsbänder
- **Raster-Dichte** 2–5 Spalten per Schieberegler
- **Schnell-Scrollleiste** mit Sprungmarke zum jeweiligen Gruppenkriterium
- **Vollbild-Viewer**: Wischen zum Blättern, Doppeltipp und **Pinch-Zoom**, Teilen, Tag bearbeiten, Info (mit gerenderter Mini-Karte), Löschen
- **Kartenansicht** aller verorteten Fotos (OpenStreetMap), Tippen auf einen Marker öffnet das Foto
- **Tags verwalten**: hinzufügen/entfernen, Mehrfachauswahl, Stapel-Tagging
- **Hintergrund-Indexierung** (OCR + GPS) als Vordergrunddienst mit Fortschrittsanzeige
- **Material You**: dynamische Farben, Hell-/Dunkelmodus

---

## Screenshots

> Platzhalter – hier können Screenshots ergänzt werden.

| Galerie | Suche & Filter | Viewer | Karte |
|--------|----------------|--------|-------|
| _tbd_  | _tbd_          | _tbd_  | _tbd_ |

---

## Voraussetzungen

- **Android 10 (API 29)** oder neuer
- Android Studio (aktuelle stabile Version)
- JDK 17

---

## Bauen & Ausführen

```bash
# Repository klonen
git clone <REPO-URL> PhotoFinder
cd PhotoFinder

# Debug-APK bauen
./gradlew assembleDebug
# Ergebnis: app/build/outputs/apk/debug/

# Oder direkt auf ein verbundenes Gerät installieren
./gradlew installDebug
```

Alternativ in **Android Studio** öffnen, Gradle synchronisieren und auf einem Gerät/Emulator mit Android 10+ starten.

---

## Berechtigungen

| Berechtigung | Wozu |
|---|---|
| `READ_MEDIA_IMAGES` (bzw. `READ_EXTERNAL_STORAGE` bis Android 12) | Zugriff auf die Fotos der Galerie |
| `ACCESS_MEDIA_LOCATION` | Auslesen der GPS-Daten aus den Foto-EXIF-Informationen |
| `ACCESS_FINE_LOCATION` / `ACCESS_COARSE_LOCATION` | Umkreissuche und Entfernungssortierung relativ zum aktuellen Standort |
| `INTERNET`, `ACCESS_NETWORK_STATE` | Laden der OpenStreetMap-Kartenkacheln |
| `FOREGROUND_SERVICE` (+ `DATA_SYNC`) | Indexierung im Hintergrund |
| `POST_NOTIFICATIONS` | Fortschrittsanzeige der Indexierung |

> **Hinweis zu GPS:** Damit Standortdaten gelesen werden können, ist der **volle** Fotozugriff plus `ACCESS_MEDIA_LOCATION` nötig. Die App fragt den Medienstandort bewusst als **separaten zweiten Schritt** nach dem Fotozugriff ab.

---

## Technologie

- **Sprache/UI:** Kotlin, Jetpack Compose, Material 3 (Material You)
- **Datenbank:** Room (lokaler Index, aktuell Schemaversion 3 mit Migrationen)
- **Hintergrundarbeit:** WorkManager / Vordergrunddienst
- **Bilder:** Coil (mit eigenem Fetcher für schnelle MediaStore-Thumbnails)
- **OCR:** ML Kit Text Recognition (Latin, on-device)
- **Karten:** osmdroid (OpenStreetMap, ohne API-Schlüssel)
- **Dependency Injection:** manuell über einen `ServiceLocator` (kein Hilt)

Wichtige Eckdaten: `minSdk 29`, `targetSdk 34`, JVM-Target 17.

---

## Architektur

Klassische, unidirektionale Schichtung:

```
UI (Jetpack Compose)
   -> ViewModel (StateFlow / UiState)
      -> Repository
         -> MediaStore   (Fotos, EXIF/GPS)
         -> Room          (Index: Tags, OCR-Text, Koordinaten, Ordner)
         -> ML Kit        (OCR)
         -> osmdroid      (Karten)
```

- Die **Suche** läuft als SQL-Vorfilter in Room und wird in Kotlin verfeinert (z. B. „ganzes Wort", Umkreis per Haversine).
- Die **Indexierung** verarbeitet OCR und GPS unabhängig voneinander und parallel.
- **Bild-Performance:** Ein eigener Coil-Fetcher nutzt vom System vorgenerierte MediaStore-Thumbnails statt Originalbilder zu dekodieren.

### Projektstruktur (Auszug)

```
app/src/main/java/com/example/photofinder/
├─ PhotoFinderApp.kt           # Application, ImageLoader, ServiceLocator-Init
├─ MainActivity.kt             # Compose-Host, Berechtigungen
├─ di/ServiceLocator.kt        # manuelle DI
├─ domain/model/               # Photo, SearchQuery, SortOption
├─ data/
│  ├─ db/                      # Room: Entity, DAO, Datenbank + Migrationen
│  ├─ media/                   # MediaStore-Quelle, Thumbnail-Fetcher
│  ├─ ocr/                     # ML-Kit-OCR-Service
│  └─ PhotoRepository.kt
├─ prefs/ConsentManager.kt
├─ work/                       # Indexierungs-Dienst + Status
├─ util/LocationResolver.kt    # Standort, Geocoding
└─ ui/
   ├─ theme/                   # Material-You-Theme
   └─ search/                  # SearchViewModel, SearchScreen (Galerie, Viewer, Karte)
```

---

## Roadmap

Bereits umgesetzt: Galerie, Tag-/OCR-/Datum-/Standortsuche, Sortierung, Gruppierung mit Headern, Raster-Dichte, Schnell-Scrollleiste, Vollbild-Viewer, Kartenansicht, Hintergrund-Indexierung, Thumbnail-Performance.

Offen / Ideen:

- Teilen und Löschen für mehrere markierte Fotos
- Tag-Verwaltung (umbenennen, zusammenführen, Tag-Wolke)
- Gespeicherte Suchen / Suchhistorie
- Favoriten / Ausblenden
- Marker-Clustering auf der Karte
- Duplikat-Erkennung
- Papierkorb / „Zuletzt gelöscht"
- Einstellungen (OCR-Sprache, Standard-Rasterdichte, Theme-Umschalter)

---

## Lizenz & Drittanbieter

Die Lizenz dieses Projekts ist noch festzulegen (z. B. MIT oder Apache-2.0).

Verwendete Open-Source-Bausteine u. a.: **osmdroid** (Apache-2.0), **Coil** (Apache-2.0), **Google ML Kit** (Nutzungsbedingungen von Google). Kartendaten stammen von **OpenStreetMap**-Mitwirkenden (ODbL); bei Veröffentlichung bitte den entsprechenden Quellenhinweis anzeigen.
