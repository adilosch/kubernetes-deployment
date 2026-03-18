# Einordnung von Argo-CD-Impersonation fuer einen zentralen Shared-Cluster-Betrieb

## Ausgangslage

In unserem Betriebsmodell laeuft Argo CD zentral auf einem Infra-Cluster. Von dort aus werden mehrere angebundene Workload-Cluster bzw. viele Team-Namespaces in Shared-Clustern verwaltet.

Die Plattform stellt die benoetigten Argo-CD-Ressourcen standardisiert und automatisiert bereit:

- `AppProject`
- `Application`
- Git-Repository
- Ziel-Namespace
- technisches Standard-Setup fuer Deployments

Es gibt dabei keinen klassischen Self-Service, bei dem Teams eigenstaendig abweichende Sonderrechte oder individuelle Argo-CD-Konfigurationen erhalten. Die Provisionierung erfolgt zentral und einheitlich durch das Plattformteam.

## Fragestellung

Die zentrale Frage war, welchen konkreten Nutzen Argo-CD-Impersonation in diesem Modell bringt, wenn ohnehin alles standardisiert durch die Plattform erzeugt wird und in sehr vielen Namespaces identische technische Muster verwendet werden.

Insbesondere stand zur Diskussion:

- pro Namespace einen zusaetzlichen dedizierten `ServiceAccount` anzulegen
- dazu jeweils `Role` und `RoleBinding` bereitzustellen
- den Argo-CD-`application-controller` nicht direkt mit seiner zentralen Identitaet deployen zu lassen
- stattdessen Deployments per Impersonation ueber den jeweiligen Namespace-`ServiceAccount` auszufuehren

## Was Impersonation technisch bedeutet

Ohne Impersonation fuehrt Argo CD die eigentlichen Aenderungen im Zielcluster mit seiner eigenen technischen Identitaet aus. In einem Shared-Cluster-Modell bedeutet das in der Praxis oft, dass diese zentrale Argo-CD-Identitaet in vielen Team-Namespaces direkt schreiben kann.

Mit Impersonation bleibt Argo CD zwar die zentrale Steuerungsinstanz, die effektive Aenderung im Ziel-Namespace erfolgt aber mit einem dort definierten `ServiceAccount`.

Vereinfacht:

- ohne Impersonation: ein zentraler Generalschluessel
- mit Impersonation: viele kleine, pro Namespace begrenzte Schluessel

## Diskussion fuer unseren konkreten Fall

Zunaechst wirkte der Zusatznutzen gering, weil:

- die Plattform ohnehin alles selbst und einheitlich anlegt
- keine individuelle Delegation an Teams erfolgt
- fuer hunderte Namespaces ein zusaetzlicher `ServiceAccount` samt `Role` und `RoleBinding` verwaltet werden muesste

Aus reiner Betriebs- und Automatisierungssicht ist diese Einschaetzung nachvollziehbar. Impersonation schafft in einem stark standardisierten Modell keinen direkten Mehrwert fuer Self-Service oder Flexibilitaet.

Der entscheidende Punkt ist jedoch die Sicherheits- und Compliance-Perspektive:

Wenn die zentrale Argo-CD-Identitaet in einem Shared Cluster ueber viele Team-Namespaces schreiben kann, dann buendelt sie umfangreiche Rechte ueber viele Mandantenbereiche. Genau darin liegt das relevante Risiko.

Gleichzeitig ist fuer unseren konkreten Fall wichtig, dass bereits starke Governance-Kontrollen existieren:

- `AppProjects` regeln genau, in welche Namespaces deployt werden darf
- Projekte werden zentral durch die Plattform erzeugt und nicht durch Teams veraendert
- Aenderungen erfolgen ueber GitOps-Prozesse
- direkte manuelle Cluster-Aenderungen werden zurueckgesetzt

Dadurch ist das Risiko ungeordneter oder eigenmaechtiger Ausweitung von Deployment-Zielen bereits deutlich reduziert.

## Was in unserem Modell bereits gut abgesichert ist

Durch die zentrale Provisionierung und die GitOps-basierte Steuerung besteht bereits eine starke logische Kontrollschicht:

- Deployments sind projektbezogen auf definierte Ziele beschraenkt
- Teams koennen Projektgrenzen nicht eigenstaendig aufweiten
- Konfigurationsaenderungen unterliegen einem kontrollierten Aenderungsprozess
- manuelle Abweichungen im Cluster werden wieder eingehegt

Diese Punkte sind wesentlich, weil sie den Governance-Nutzen von Impersonation in unserem Modell kleiner machen als in offenen Self-Service-Umgebungen.

## Was Impersonation zusaetzlich absichern wuerde

Impersonation ersetzt diese Governance-Schicht nicht, sondern legt eine technische Begrenzung darunter.

Die Unterscheidung ist:

- `AppProject` definiert, was Argo CD logisch tun soll
- Impersonation begrenzt, was die technische Identitaet im Cluster tatsaechlich tun kann

Solange alle Konfigurationen korrekt sind, fuehren beide Modelle oft zum selben Ergebnis. Der Zusatznutzen von Impersonation zeigt sich erst dann, wenn die logische Steuerung versagt, fehlerhaft umgesetzt wird oder in Ausnahmefaellen nicht mehr ausreichend greift.

Relevante Rest-Risiken bleiben zum Beispiel:

- Fehler in der zentralen Plattform-Automation
- Fehlkonfigurationen in generierten `Applications` oder `AppProjects`
- Sicherheitsvorfaelle in Argo CD selbst oder bei seiner technischen Identitaet
- spaetere Sonderfaelle, in denen zusaetzliche Rechte oder Ressourcenarten eingefuehrt werden

In diesen Situationen hilft Impersonation als zusaetzliche Begrenzung des effektiven Berechtigungsscope.

## Relevanz fuer DORA / BAIT / BAFIN / Least Privilege

Da unser Umfeld regulatorisch gepraegt ist und Least Privilege beachtet werden muss, ist nicht nur die Automatisierungseffizienz entscheidend, sondern vor allem die Frage, wie Berechtigungen technisch zugeschnitten und begrenzt werden.

Impersonation bringt in diesem Kontext folgenden Nutzen:

- die zentrale Argo-CD-Instanz orchestriert Deployments, besitzt aber nicht mehr die eigentlichen breiten Schreibrechte in allen Ziel-Namespaces
- die effektiven Deploy-Rechte liegen bei dedizierten, namespacebezogenen technischen Konten
- der Berechtigungsscope pro Deployment wird klar abgegrenzt
- der mandantenuebergreifende Schadensradius sinkt
- das Modell ist gegenueber Pruefern besser begruendbar als eine zentrale Sammelidentitaet mit Flaechenrechten

Wichtig ist dabei:

Impersonation ersetzt nicht gutes RBAC, sondern macht gutes RBAC pro Zielbereich technisch nutzbar. Der Sicherheitsgewinn entsteht nur dann, wenn die impersonierten `ServiceAccounts` tatsaechlich minimal berechtigt sind.

## Vorteile in einfacher Sprache

Heute ist Argo CD in einem Shared Cluster ohne Impersonation vergleichbar mit einem Generalschluessel, der in sehr vielen Team-Bereichen funktioniert.

Mit Impersonation nutzt Argo CD fuer jeden Namespace einen kleineren, separaten Schluessel. Dadurch kann ein Fehler, eine Fehlkonfiguration oder ein Missbrauch nicht mehr automatisch denselben Effekt ueber alle Team-Bereiche hinweg entfalten.

Das macht den Betrieb zwar etwas aufwendiger, aber aus Sicherheits- und Regulierungssicht auch deutlich sauberer.

## Einschraenkungen und Aufwand

Impersonation ist kein kostenloser Gewinn. Das Plattformteam muss zusaetzlich standardisiert verwalten:

- einen dedizierten `ServiceAccount` pro Namespace oder Zielbereich
- passende `Role`- bzw. `RoleBinding`-Objekte
- die Zuordnung im `AppProject` oder in der Argo-CD-Projektkonfiguration

Bei hunderten Namespaces erhoeht das die Zahl der verwalteten Objekte und die Komplexitaet der Provisionierung. Der Nutzen entsteht deshalb vor allem dort, wo das Risiko einer zu maechtigen Zentralidentitaet schwerer wiegt als dieser Zusatzaufwand.

## Sollbild fuer Berechtigungen

Wenn Impersonation als echtes Least-Privilege-Modell genutzt werden soll, dann muss die Rechteverteilung bewusst getrennt werden.

### Rechte, die beim `application-controller` bleiben

Der zentrale Argo-CD-`application-controller` behaelt nur die Rechte, die fuer Steuerung, Beobachtung und technische Pruefung erforderlich sind:

- Argo-CD-interne Rechte im Namespace `argocd`
- Lese- und Watch-Rechte fuer den Clusterabgleich
- erforderliche Discovery- und Vergleichsrechte
- `impersonate` auf die vorgesehenen Ziel-`ServiceAccounts`
- gegebenenfalls Rechte auf Pruefressourcen wie `selfsubjectaccessreviews`

Wesentlich ist dabei, dass der Controller nicht zusaetzlich weiterhin die eigentlichen fachlichen Schreibrechte in allen Team-Namespaces direkt besitzt.

### Rechte, die in die Ziel-`ServiceAccounts` gehoeren

Die dedizierten `ServiceAccounts` in den Ziel-Namespaces tragen die eigentlichen Deploy-Rechte:

- `create`, `update`, `patch`, `delete` auf die benoetigten Namespace-Ressourcen
- nur innerhalb des jeweils vorgesehenen Namespace
- moeglichst ohne Cluster-Rechte
- moeglichst ohne unnoetig breite Wildcards

Das Prinzip lautet:

- der `application-controller` steuert den Deployment-Vorgang
- der namespacebezogene `ServiceAccount` fuehrt die Aenderung mit lokal begrenzten Rechten aus

### Bedeutung fuer Shared Cluster

Gerade in Shared-Clustern mit vielen Team-Namespaces ist diese Trennung wesentlich. Ohne sie bleibt Argo CD faktisch eine zentrale Sammelidentitaet mit breiten Aenderungsrechten ueber viele Mandantenbereiche hinweg.

Mit Impersonation und verschlanktem Controller-RBAC wird diese Rechtebuendelung reduziert:

- die zentrale Steuerungsinstanz bleibt erhalten
- die effektiven Schreibrechte werden auf lokale technische Identitaeten verlagert
- der Schadensradius einer Fehlkonfiguration oder Kompromittierung wird begrenzt

### Umgang mit Cluster-Ressourcen

Cluster-weite Ressourcen sind ein gesonderter Fall. Namespacebezogene Deploy-`ServiceAccounts` reichen fuer solche Objekte nicht aus.

Fuer diesen Fall sollte es kein allgemeines Standardmodell geben, sondern einen bewusst kontrollierten Ausnahmeweg, zum Beispiel:

- separate privilegierte Projekte
- gesonderte technische Identitaeten
- eigene Freigabe- und Review-Prozesse

Damit bleibt der Regelfall fuer normale Team-Anwendungen namespacebezogen und least-privileged, waehrend clusterweite Rechte nur gezielt und nachvollziehbar vergeben werden.

## Trennung von Provisionierung und Deployment

In unserem Modell ist zusaetzlich zu beachten, dass die zentrale technische Identitaet fuer die Zielcluster in bestimmten Faellen weiterhin clusterweite Rechte benoetigen kann, zum Beispiel zum Anlegen von `Namespaces`.

Das ist fachlich ein anderer Vorgang als das eigentliche Deployment einer Anwendung.

Eine sinnvolle Trennung ist daher:

### 1. Zentrale Provisionierung

Diese Phase umfasst plattformseitige Vorbereitungen wie:

- Anlegen des Namespace
- Anlegen von `ServiceAccount`
- Anlegen von `Role` und `RoleBinding`
- gegebenenfalls Labels, Quotas oder weitere Standardobjekte

Diese Schritte koennen weiterhin mit einer staerker privilegierten zentralen Plattform-Identitaet erfolgen, weil hier bewusst clusterweite oder plattformweite Aufgaben ausgefuehrt werden.

### 2. Laufendes Anwendungs-Deployment

Nach abgeschlossener Provisionierung sollte das eigentliche Deployment der Anwendung moeglichst nicht mehr mit dieser zentralen Identitaet erfolgen.

Stattdessen werden die fachlichen Ressourcen im Ziel-Namespace mit dem dort vorgesehenen Deploy-`ServiceAccount` ausgerollt, zum Beispiel:

- `Deployment`
- `Service`
- `ConfigMap`
- `Secret`

Dadurch bleibt die staerker privilegierte zentrale Identitaet auf Plattform-Aufgaben begrenzt, waehrend die laufenden App-Deployments mit lokal beschraenkten Rechten erfolgen.

## Bedeutung fuer die Bewertung von Impersonation

Dieser Punkt ist wichtig, weil Impersonation nicht bedeutet, dass die zentrale Argo-CD- oder Plattform-Identitaet ueberhaupt keine clusterweiten Rechte mehr haben darf.

Realistischer ist:

- zentrale Identitaet fuer Provisionierung und technische Steuerung
- namespacebezogene Identitaet fuer das eigentliche Anwendungs-Deployment

Der Sicherheitsgewinn entsteht also nicht dadurch, dass saemtliche zentralen Rechte verschwinden, sondern dadurch, dass normale App-Syncs nicht mehr mit einer breit berechtigten Zentralidentitaet ausgefuehrt werden.

## Zusammenfassung der Unterhaltung

Im Verlauf der Diskussion wurden folgende Punkte geklaert:

- Eine leere oder fehlende `namespaceResourceWhitelist` im `AppProject` fuehrte in der Argo-CD-Oberflaeche nicht zum gewuenschten Verhalten fuer Child-Resources wie `Pod` und `ReplicaSet`.
- Fuer die Demo wurde deshalb `namespaceResourceWhitelist` explizit auf alle Namespaced-Ressourcen gesetzt:

```yaml
namespaceResourceWhitelist:
  - group: "*"
    kind: "*"
```

- Der grundlegende Vorteil von Impersonation wurde anhand der Argo-CD-Proposal eingeordnet:
  Argo CD soll nicht mit seinen eigenen weitreichenden Controller-Rechten deployen, sondern mit den Rechten eines definierten Ziel-`ServiceAccount`.
- Fuer ein stark zentralisiertes Plattformmodell ohne Team-Self-Service ist der organisatorische Mehrwert geringer.
- Fuer ein reguliertes Shared-Cluster-Modell mit vielen Team-Namespaces bleibt der Sicherheits- und Compliance-Mehrwert jedoch relevant.
- Gleichzeitig wurde festgestellt, dass in unserem Modell bereits starke Projekt- und GitOps-Grenzen bestehen, wodurch der unmittelbare Governance-Mehrwert von Impersonation kleiner ausfaellt.
- Daraus ergibt sich fuer unseren Fall vor allem ein `defense in depth`-Nutzen:
  Impersonation waere weniger eine neue Steuerungslogik als eine zusaetzliche technische Härtung unterhalb der bereits existierenden Prozesse.
- Es wurde ausserdem herausgearbeitet, dass clusterweite Plattform-Aufgaben wie das Anlegen von `Namespaces` weiterhin eine staerker privilegierte zentrale Identitaet erfordern koennen.
- Daraus folgt eine sinnvolle Trennung zwischen zentraler Provisionierung und spaeterem Applikations-Deployment mit namespacebezogener Identitaet.

## Resuemee

Fuer unser Plattformmodell ist Argo-CD-Impersonation kein Komfort- oder Self-Service-Feature, sondern in erster Linie ein Sicherheits- und Compliance-Mechanismus.

Wenn eine zentrale Argo-CD-Identitaet in einem Shared Cluster direkt in vielen Team-Namespaces schreiben kann, ist dies aus Least-Privilege-Sicht schwerer zu vertreten. Impersonation verlagert die effektiven Deploy-Rechte auf dedizierte, lokal begrenzte ServiceAccounts und reduziert damit die Rechtebuendelung einer zentralen Sammelidentitaet.

Der operative Aufwand steigt dadurch, insbesondere bei hunderten Namespaces. Da die Plattform die Provisionierung jedoch ohnehin automatisiert, ist dieser Aufwand beherrschbar. In einem DORA-/BAIT-/BAFIN-nahen Umfeld ist Impersonation deshalb gut begruendbar, wenn damit die technische Rechtebegrenzung pro Namespace oder Team nachweisbar verbessert wird.

Die pragmatische Bewertung lautet daher:

Impersonation ist fuer uns nicht wegen besserer Flexibilitaet wichtig, sondern wegen sauberer Rechte-Trennung, geringerer Blast Radius und besserer Argumentierbarkeit gegenueber Revision, Audit und Aufsicht.

Gleichzeitig ist festzuhalten, dass unser bestehendes Modell mit zentral verwalteten Projekten, kontrollierten GitOps-Prozessen und Ruecksetzung manueller Cluster-Aenderungen bereits eine starke Governance-Basis bildet. Der Zusatznutzen von Impersonation liegt fuer uns daher weniger in neuer funktionaler Kontrolle als in zusaetzlicher technischer Härtung gegen Fehlkonfigurationen, Produktfehler und Risiken einer zu maechtigen Zentralidentitaet.

Besonders sinnvoll wird diese Einordnung, wenn zwischen zentraler Provisionierung und laufendem Deployment unterschieden wird: Clusterweite Vorbereitungsaufgaben koennen weiterhin ueber eine privilegierte Plattform-Identitaet erfolgen, waehrend normale Anwendungs-Syncs im Ziel-Namespace ueber dedizierte, lokal begrenzte ServiceAccounts laufen.

## Empfehlung fuer unseren Fall

Fuer unseren Betrieb ist eine pragmatische Einfuehrung sinnvoller als eine vollstaendige Umstellung in einem Schritt.

Empfohlenes Zielbild:

- zentrale Plattform-Identitaet weiterhin fuer Provisionierung und clusterweite Vorbereitungsaufgaben
- namespacebezogene Deploy-`ServiceAccounts` fuer normale Anwendungs-Syncs
- klare Trennung von Standardfall und Ausnahmefall fuer clusterweite Ressourcen
- vollautomatisierte Bereitstellung der zusaetzlichen RBAC-Objekte durch die Plattform

Empfohlene Bewertung:

- als Governance-Mechanismus ist der Zusatznutzen in unserem Modell begrenzt, da bereits starke zentrale GitOps-Kontrollen bestehen
- als technische Härtung und Least-Privilege-Maßnahme ist Impersonation jedoch sinnvoll
- der groesste Mehrwert entsteht dort, wo die breite zentrale Schreibidentitaet in Shared-Clustern auf viele Team-Namespaces reduziert werden kann

Praktische Entscheidungsempfehlung:

Impersonation sollte bei uns nicht eingefuehrt werden, um bestehende Projekt- und GitOps-Steuerung zu ersetzen, sondern um die effektiven Deploy-Rechte im laufenden Betrieb pro Namespace besser zu begrenzen. Wenn der zusaetzliche Automatisierungsaufwand beherrschbar ist, ist dies aus Sicherheits- und Compliance-Sicht eine nachvollziehbare und gut begruendbare Weiterentwicklung des bestehenden Modells.
