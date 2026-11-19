# App Interaktion Workflow

## Workflow Anmeldung

```mermaid
---
config:
  theme: redux-color
---
sequenceDiagram
    actor V as Versicherter
    participant LER_APP as Leistungserbringer-FdV
    participant LER_BE as Leistungserbringer-Backend
    participant KK_APP as Krankenkassen-FdV
    participant BS as Bestätigungssystem
    autonumber

    Note over V,BS: Phase 1 – Fahrtorganisation
    activate V
    V->>V: Genehmigungsinformation von<br>Krankenkasse erhalten
    V->>LER_APP: FdV öffnen
    deactivate V
    LER_APP->>LER_BE: Fahrt planen
    activate LER_APP
    Note over V,BS: Phase 2 – Teilnahmestatus abfragen
    LER_BE->>LER_BE: Pseudonym (UUID) für eKVNR erstellen und speichern<br>sowie Callback-Deep-Link (Universal Link/App Link)<br>für Rücksprung in Leistungserbringer-FdV erzeugen
    LER_BE->>BS: Prüfen ob eKVNR am eLB Verfahren teilnimmt -<br>Übergabe eKVNR, Pseudonym, Webhook-URL, Callback-Deep-Link
    activate BS
    BS->>BS: Callback-Deep-Link<br>gegen Registry prüfen
    break Callback-Deep-Link-Schema nicht in Registry
      rect rgba(220, 0, 0, 0.15)
        BS-->>LER_BE: Fehlermeldung
      end
    end
    BS->>BS: Webhook-URL und<br>Callback-Deep-Link zusammen<br>mit eKVNR + Pseudonym speichern
    BS-->>LER_BE: Teilnahmestatus
    break Versicherter hat zugestimmt
        rect rgba(0, 220, 0, 0.15)
            LER_BE-->>LER_APP: Rückmeldung Zustimmung liegt vor
            LER_APP->>LER_APP: Weiter mit Fahrtplanung
        end
    end

    Note over V,BS: Phase 3 – Zustimmung in Krankenkassen-FdV einholen
    LER_BE-->>LER_APP: Rückmeldung "Zustimmung liegt nicht vor"<br>inkl. Übergabe Pseudonym + Callback-Deep-Link
    LER_APP->>LER_APP: Hinweise auf aktuell fehlende Zustimmung für eLB Verfahren,<br>und notwendige Zustimmung in Krankenkassen-FdV sowie<br>Hinweis auf bevorstehenden Wechsel in Krankenkassen-FdV
    LER_APP->>KK_APP: Krankenkassen-FdV per "eLB consent"-Deep-Link inkl.<br>Pseudonym aufrufen und in den Vordergrund bringen
    V-->>KK_APP: Versicherter nimmt Wechsel in Krankenkassen-FdV wahr
    V-->>KK_APP: Anmeldung (falls nötig)
    V->>KK_APP: Versicherter bestätigt Teilnahme am eLB Verfahren
    KK_APP->>BS: Teilnahmebestätigung inkl. eKVNR + Pseudonym übermitteln
    BS->>BS: Teilnahmestatus speichern
    BS->>BS: Suche per eKVNR + Pseudonym ob ein<br>Callback-Deep-Link gespeichert wurde
    BS-->>KK_APP: Sofern Pseudonym gefunden, Rückmeldung des<br>Callback-Deep-Link für Leistungserbringer-FdV    
    BS->>LER_BE: Leistungserbringer-Webhook-URL inkl. eKVNR und Teilnahmestatus aufrufen
    LER_BE->>LER_BE: Teilnahmestatus speichern

    Note over V,BS: Phase 4 – Rückwechsel zu Leistungserbringer-FdV
    KK_APP->>LER_APP: Leistungserbringer-FdV mittels Callback-Deep-Link in den Vordergrund bringen
    V-->>LER_APP: Versicherter nimmt Wechsel in Leistungserbinger-FdV wahr
    LER_APP->>LER_BE: Teilnahmestatus prüfen
    LER_BE-->>LER_APP: Teilnahmestatus
    LER_APP->>LER_APP: Weiter mit Fahrtplanung
```

## Workflow Bestätigung

```mermaid
---
config:
  theme: redux-color
---
sequenceDiagram
    actor V as Versicherter
    participant LER_APP as Leistungserbringer-App
    participant LER_BE as Leistungserbringer-Backend
    participant KK_APP as Krankenkassen-App
    participant BS as Bestätigungssystem
    autonumber


    Note over V,BS: Phase 1 – Bestätigungsanfrage anlegen
    activate LER_BE
    LER_BE->>LER_BE: Fahrt als beendet markieren
    LER_BE->>LER_BE: Bestätigungsanfrage inkl. ID erzeugen
    LER_BE->>BS: Bestätigungsanfrage anlegen<br>(Anfrage-ID, Fahrtdaten, KVNR,<br>Leistungserbringer-Webhook-URL und<br>Deep Link inkl. Anfrage-ID für die Leistungserbringer-App)
    activate BS
    BS->>BS: Deep Link der Leistungserbringer-App prüfen
    break Deep Link falsch
      rect rgba(220, 0, 0, 0.15)
        BS-->>LER_BE: Fehlermeldung
      end
    end
    BS->>BS: Bestätigungsanfrage speichern
    BS-->>LER_BE: Anlegen bestätigt
    deactivate BS

    Note over V,BS: Phase 2 – Versicherten informieren
    LER_BE-)LER_APP: Push inkl. Anfrage-ID
    deactivate LER_BE
    activate LER_APP
    LER_APP->>V: Aufforderung, die Fahrt in der<br>Krankenkassen-App zu bestätigen
    V->>LER_APP: Tippt auf „Jetzt zur Krankenkassen-App wechseln“
    LER_APP-)KK_APP: Krankenkassen-App per Deep Link öffnen<br>(im Link ist Anfrage-ID enthalten)
    deactivate LER_APP
    activate KK_APP

    Note over V,BS: Phase 3 – Bestätigung in der Krankenkassen-App
    V->>KK_APP: Anmeldung (falls nötig)
    KK_APP->>BS: Anfrage per ID laden
    activate BS
    BS-->>KK_APP: Anfragedaten inkl. Deep Link<br>für die Leistungserbringer-App
    deactivate BS
    KK_APP->>V: Fahrtdaten anzeigen
    V->>KK_APP: Bestätigt die Fahrt
    KK_APP->>BS: Bestätigungsstatus übermitteln
    activate BS
    BS->>BS: Status speichern
    BS-->>KK_APP: OK

    Note over V,BS: Phase 4 – Status zurückmelden (Webhook und Rücksprung laufen parallel)
    BS-)LER_BE: Webhook-URL mit Bestätigungsobjekt aufrufen
    deactivate BS
    activate LER_BE
    LER_BE->>LER_BE: Bestätigungsobjekt speichern
    deactivate LER_BE
    KK_APP-)LER_APP: Deep Link für die Leistungserbringer-App aufrufen
    deactivate KK_APP
    activate LER_APP
    LER_APP->>LER_BE: Status per Anfrage-ID abfragen
    activate LER_BE
    LER_BE-->>LER_APP: Status rückmelden
    deactivate LER_BE
    LER_APP->>V: Bestätigungsstatus anzeigen,<br>Follow-up-Flow starten
    deactivate LER_APP
```
