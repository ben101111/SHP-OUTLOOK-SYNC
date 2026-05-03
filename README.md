[README.md](https://github.com/user-attachments/files/27321391/README.md)
# SHP Outlook Sync Deployment Package

## Wichtiger Hinweis

Dieses Projekt wurde mit Unterstützung von künstlicher Intelligenz erstellt. Der Code wurde für den beschriebenen Anwendungsfall entwickelt und getestet, ersetzt aber keine professionelle Softwareprüfung, Sicherheitsprüfung oder Herstellerfreigabe.

Der Einsatz in einer Produktivumgebung erfolgt auf eigene Gefahr. Vor produktiver Nutzung müssen Installation, Berechtigungen, Datenbankzugriff, Microsoft-Graph-Zugriff, Fehlerverhalten, Logging, Datenschutz und Wiederherstellbarkeit durch den Betreiber eigenverantwortlich geprüft werden. Es wird dringend empfohlen, zuerst in einer Testumgebung zu testen und vor dem Einsatz aktuelle Backups der betroffenen Systeme und Datenbanken anzulegen.

Dieses Paket installiert den smarthandwerk-pro-zu-Outlook-Sync als Windows Scheduled Task.

## Installation

PowerShell im Paketordner öffnen:

```powershell
.\install\Install-SHPOutlookSync.ps1
```

Der Installer fragt kundenspezifische Werte ab:

- PostgreSQL Host, Port, Datenbank, Schema und Benutzer
- Pfad zu `psql.exe`
- Microsoft Tenant ID und Client ID
- Microsoft Graph Client Secret
- Zielpostfach für den zentralen Outlook-Kalender
- Sync-Intervall

Secrets werden nicht in JSON gespeichert, sondern als DPAPI-verschlüsselte Dateien im `config`-Ordner. Der Scheduled Task muss deshalb unter demselben Windows-Benutzer laufen, der die Installation ausgeführt hat.

Standardmäßig wird der Scheduled Task so angelegt, dass er auch dann läuft, wenn der Windows-Benutzer nicht angemeldet ist. Dafür fragt der Installer zusätzlich das Windows-Kennwort des aktuell angemeldeten Benutzers ab. Dieses Kennwort wird nicht vom Paket gespeichert, sondern nur an die Windows-Aufgabenplanung übergeben.

Empfehlung für Kundeninstallationen:

1. Ein eigenes Windows-Konto für den Sync verwenden, zum Beispiel `SHP-Sync`.
2. Mit diesem Konto einmal am Server anmelden.
3. Das Paket unter diesem Konto installieren.
4. Beim Installer das Windows-Kennwort dieses Kontos eingeben.
5. Den Task danach über die Aufgabenplanung oder per PowerShell aktivieren.

Wenn der Task bewusst nur laufen soll, solange der Benutzer angemeldet ist:

```powershell
.\install\Install-SHPOutlookSync.ps1 -RunOnlyWhenLoggedOn
```

Der Task wird standardmäßig deaktiviert angelegt, damit zuerst das Mitarbeiter-Mapping geprüft werden kann. Wenn der Task sofort aktiviert werden soll:

```powershell
.\install\Install-SHPOutlookSync.ps1 -EnableTask
```

## Mitarbeiter-Mapping

Nach der Installation diese Datei bearbeiten:

```text
config\employee-mapping.csv
```

Format:

```csv
employee_no,display_name,category_name
10001,Max Mustermann,Max Mustermann
10002,Erika Beispiel,Erika Beispiel
```

Die Outlook-Kategorien und Farben werden absichtlich nicht vom Skript verwaltet. Lege die Kategorien mit denselben Namen im Zielpostfach manuell an und setze dort die Farben.

## Test

```powershell
.\install\Test-SHPOutlookSync.ps1
```

Der Test läuft im Dry-Run-Modus und schreibt nichts in Outlook.

Nach erfolgreichem Test den Task aktivieren:

```powershell
Enable-ScheduledTask -TaskName "SHP Outlook Sync"
```

## Betrieb

Der Scheduled Task führt aus:

```powershell
app\Sync-SmarthandwerkCalendarToOutlook.ps1 -ConfigPath config\sync-config.json -Execute
```

Logs:

```text
logs\sync.log
```

Sync-State:

```text
data\sync-map.json
```

Den Task-Status und den Anmeldemodus prüfen:

```powershell
Get-ScheduledTask -TaskName "SHP Outlook Sync" | Select-Object TaskName,State,@{Name="LogonType";Expression={$_.Principal.LogonType}},@{Name="UserId";Expression={$_.Principal.UserId}}
```

Einmal manuell im Hintergrund starten:

```powershell
Start-ScheduledTask -TaskName "SHP Outlook Sync"
Start-Sleep -Seconds 10
Get-Content .\logs\sync.log -Tail 80
```

## psql.exe nicht gefunden

Wenn beim Test erscheint:

```text
Command 'psql' was not found
```

dann ist der PostgreSQL-Binärordner nicht im PATH. Der Installer fragt deshalb den vollständigen Pfad zu `psql.exe` ab und speichert ihn in:

```json
"psqlPath": "C:\\Program Files\\PostgreSQL\\16\\bin\\psql.exe"
```

Falls nötig, kann der Wert nachträglich in `config\sync-config.json` korrigiert werden.

## Deinstallation

Nur Scheduled Task entfernen:

```powershell
.\install\Uninstall-SHPOutlookSync.ps1
```

Auch Config/Secrets/Sync-State entfernen:

```powershell
.\install\Uninstall-SHPOutlookSync.ps1 -RemoveConfigAndData
```
