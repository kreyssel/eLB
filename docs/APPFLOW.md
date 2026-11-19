# App Interaktion Workflow

## Workflow Anmeldung "Happy Path"

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

    Note over V,BS: Phase 1 – Organisation
    activate V
    V->>V: Informationen von<br>Krankenkasse erhalten
    V->>LER_APP: Leistungserbringer-FdV öffnen
    activate LER_APP
    deactivate V
    LER_APP->>LER_BE: Informationen zum Versicherten laden
    activate LER_BE

    Note over V,BS: Phase 2 – Teilnahmestatus abfragen
    LER_BE->>LER_BE: Pseudonym (UUID) für eKVNR erstellen und speichern<br>sowie Callback-Deep-Link (Universal Link/App Link)<br>für Rücksprung in Leistungserbringer-FdV erzeugen
    LER_BE->>BS: Prüfen, ob eKVNR am eLB-Verfahren teilnimmt -<br>Übergabe eKVNR, Pseudonym, Webhook-URL, Callback-Deep-Link
    activate BS
    BS->>BS: Callback-Deep-Link<br>gegen Registry prüfen
    break Callback-Deep-Link-Domain nicht in Registry
      rect rgba(220, 0, 0, 0.15)
        BS-->>LER_BE: Fehlermeldung
      end
    end
    BS->>BS: Webhook-URL und<br>Callback-Deep-Link zusammen<br>mit eKVNR + Pseudonym speichern
    BS-->>LER_BE: Teilnahmestatus
    deactivate BS
    break Versicherter hat zugestimmt
        rect rgba(0, 220, 0, 0.15)
            LER_BE-->>LER_APP: Rückmeldung Zustimmung liegt vor
            activate V
            V->>LER_APP: Weiter im Leistungs-Workflow
            deactivate V
        end
    end

    Note over V,BS: Phase 3 – Zustimmung in Krankenkassen-FdV einholen
    LER_BE-->>LER_APP: Rückmeldung "Zustimmung liegt nicht vor"<br>inkl. Übergabe Pseudonym
    deactivate LER_BE
    LER_APP->>LER_APP: Hinweis anzeigen: "Zustimmung zum eLB-Verfahren fehlt und<br>muss in der Krankenkassen-FdV erteilt werden" - Wechsel dorthin folgt
    LER_APP->>KK_APP: Krankenkassen-FdV per<br>"eLB consent"-Deep-Link inkl.<br>Pseudonym aufrufen
    activate KK_APP
    deactivate LER_APP
    KK_APP->>KK_APP: Krankenkassen-FdV kommt in den Vordergrund
    activate V
    opt Versicherter nicht angemeldet
        V->>KK_APP: Anmeldung
    end
    V->>KK_APP: Versicherter bestätigt Teilnahme am eLB-Verfahren
    deactivate V
    KK_APP->>BS: Teilnahmebestätigung inkl. eKVNR + Pseudonym übermitteln
    activate BS
    BS->>BS: Teilnahmestatus speichern
    BS->>BS: Suche per eKVNR + Pseudonym ob ein<br>Callback-Deep-Link gespeichert wurde
    BS-->>KK_APP: Sofern Eintrag gefunden, Rückmeldung des<br>Callback-Deep-Link für Leistungserbringer-FdV
    BS->>LER_BE: Sofern Eintrag gefunden,<br>Leistungserbringer-Webhook-URL inkl.<br>eKVNR und Teilnahmestatus aufrufen
    activate LER_BE
    LER_BE->>LER_BE: Teilnahmestatus speichern
    deactivate LER_BE
    deactivate BS

    Note over V,BS: Phase 4 – Rückwechsel zu Leistungserbringer-FdV
    KK_APP->>LER_APP: Leistungserbringer-FdV mittels Callback-Deep-Link aufrufen
    activate LER_APP
    deactivate KK_APP
    LER_APP->>LER_APP: Leistungserbringer-FdV kommt in den Vordergrund
    LER_APP->>LER_BE: Teilnahmestatus prüfen
    activate LER_BE
    LER_BE-->>LER_APP: Teilnahmestatus
    deactivate LER_BE
    LER_APP->>LER_APP: Info, dass das Onboarding ins<br>eLB-Verfahren abgeschlossen ist
    activate V
    V->>LER_APP: Weiter im Leistungs-Workflow
    deactivate V
    deactivate LER_APP
```

**Offene Punkte**

Inhaltlich sind alle acht Punkte unverändert. Angepasst sind nur die Schrittnummern sowie bei Punkt 1 und 2 der Bezug auf den neuen Schritt 28.

1. **Race Condition zwischen Webhook und Statusabfrage (Schritte 21–28)**
Die Rückmeldung des Callback-Deep-Links an die Krankenkassen-FdV (21) und der Webhook-Aufruf (22) laufen faktisch parallel. Fragt die Leistungserbringer-FdV den Teilnahmestatus ab (26), bevor das Leistungserbringer-Backend den Webhook verarbeitet hat, erhält sie einen veralteten Status, und die Info in Schritt 28 wäre falsch. Lösungsoptionen: Das Leistungserbringer-Backend fragt bei noch fehlendem Webhook direkt beim Bestätigungssystem nach, oder das Bestätigungssystem ruft den Webhook auf, bevor es der Krankenkassen-FdV antwortet.

2. **Ablehnung der Zustimmung nicht modelliert (17, 28, 29)**
Abgebildet ist nur die Bestätigung der Teilnahme. Offen ist, wie eine Ablehnung an das Bestätigungssystem übermittelt wird und wie die Leistungserbringer-FdV darauf reagiert. Die Schritte 28 und 29 folgen derzeit unabhängig vom Teilnahmestatus.

3. **Kein Eintrag im Bestätigungssystem gefunden (21, 22, 24)**
Die Schritte 21 und 22 setzen voraus, dass ein Eintrag gefunden wird. Der Fall ohne Eintrag ist nicht beschrieben. Er tritt z. B. bei abgelaufenem Pseudonym auf oder wenn in der Krankenkassen-FdV ein anderer Versicherter angemeldet ist. Ohne Callback-Deep-Link entfällt Schritt 24. Offen ist, welche Rückmeldung die Krankenkassen-FdV erhält und wie der Versicherte zur Leistungserbringer-FdV zurückfindet, etwa über einen Hinweis zum manuellen Wechsel.

4. **Keine Registry-Prüfung der Webhook-URL (6)**
Geprüft wird nur der Callback-Deep-Link. Ohne Prüfung der Webhook-URL könnte das Bestätigungssystem Teilnahmestatus und eKVNR an beliebige Endpunkte senden.

5. **Fehlermeldung erreicht die Leistungserbringer-FdV nicht (7)**
Bei negativer Registry-Prüfung endet der Ablauf im Leistungserbringer-Backend. Offen ist, was dem Versicherten in der Leistungserbringer-FdV angezeigt wird.

6. **Ermittlung des Consent-Deep-Links der zuständigen Krankenkasse (14)**
Offen ist, woher die Leistungserbringer-FdV den „eLB consent“-Deep-Link der Krankenkassen-FdV des jeweiligen Versicherten kennt. Optionen: Das Bestätigungssystem gibt ihn zusammen mit dem Teilnahmestatus zurück (9), oder er wird aus den Genehmigungsinformationen abgeleitet.

7. **Krankenkassen-FdV nicht installiert (14)**
Ist die Krankenkassen-FdV auf dem Gerät nicht installiert, öffnet das Betriebssystem den Universal Link/App Link im Browser. Offen ist, welches Verhalten in diesem Fall vorgesehen ist, z. B. eine Webseite der Krankenkasse oder ein Hinweis zur Installation.

8. **Übermittlung der eKVNR im Webhook (22)**
Zur Zuordnung würde das Pseudonym genügen, da das Leistungserbringer-Backend die Zuordnung zur eKVNR in Schritt 4 gespeichert hat. Das entspräche besser der Datenminimierung. Der Punkt entfällt, falls die eKVNR bewusst übertragen wird.

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
    LER_BE->>LER_BE: Leistung als erbracht markieren
    LER_BE->>LER_BE: Bestätigungsanfrage inkl. ID erzeugen
    LER_BE->>BS: Bestätigungsanfrage anlegen<br>(Anfrage-ID, Leistungsdaten, KVNR,<br>Leistungserbringer-Webhook-URL und<br>Deep Link inkl. Anfrage-ID für die Leistungserbringer-App)
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
    LER_APP->>V: Aufforderung, die Leistungs in der<br>Krankenkassen-App zu bestätigen
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
    KK_APP->>V: Leistungsdaten anzeigen
    V->>KK_APP: Bestätigt die Leistung
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
