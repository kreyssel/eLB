# App Interaktion Workflow

## Workflow Anmeldung

## Workflow Bestätigung

```mermaid
sequenceDiagram
    autonumber
    participant LER_APP as Leistungserbringer Kundenapp
    participant LER_BE as Leistungserbringer Backend
    participant KK_APP as Krankenkassen App
    participant BS as Bestätigungssystem

    activate LER_BE
    LER_BE->>LER_BE: Fahrt beendet
    LER_BE->>LER_BE: neue Bestätigungsanfrage<br>für Fahrt inkl. ID erzeugen
    LER_BE->>BS: Bestätigunganfrage<br>inkl. Callbacks für Leistungserbringer Backend<br>als auch Callback für Krankenkassen App
    activate BS
    LER_BE->>LER_APP: Push "Fahrt beendet"<br>Bestätigung in KK-App notwendig<br>Übergabe ID der Bestätigungsanfrage
    activate LER_APP
    deactivate LER_BE
    BS->>BS: Bestäigungsanfrage inkl. Leistungserbringer<br>Backend Callback speichern
    deactivate BS
    LER_APP->>LER_APP: Aufforderung an Versicherten die<br>Leistung via Krankenkassen App zu bestätigen
    LER_APP->>KK_APP: Krankenkassen App mit ID der Bestätigungsanfrage aufrufen
    activate KK_APP
    deactivate LER_APP
    KK_APP->>BS: Laden der Bestätigunganfrage per ID <br>inkl. Callback für Leistungserbringer App
    KK_APP->>KK_APP: Versicherter bestätigt Anfrage
    KK_APP->>BS: Übermittlung des Bestätigungsstatus per ID
    activate BS
    KK_APP->>LER_APP: Ruft Leistungserbringer App mit übermittelten Callback auf
    deactivate KK_APP
    activate LER_APP
    LER_APP->>LER_APP: Wartet auf Callback vom Bestätigungssystem
    BS->>LER_BE: Übermittlung des Bestätigungsstatus per Callback
    deactivate BS
    activate LER_BE
    LER_BE->>LER_APP: Übermittlung des Bestätigungsstatus 
    deactivate LER_BE
    LER_APP->>LER_APP: Anzeige des Bestätigungsstatus<br>und Follow-up Flow
    deactivate LER_APP

```
