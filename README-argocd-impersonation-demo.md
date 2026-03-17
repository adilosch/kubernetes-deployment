# Argo CD Impersonation Demo

Diese Demo zeigt, wie Argo CD eine normale `Application` nicht mit seinen allgemeinen Controller-Rechten synchronisiert, sondern im Ziel-Namespace gezielt als ein definierter Kubernetes-`ServiceAccount`.

Die Demo ist fuer einen lokalen Cluster gedacht. Argo CD laeuft dabei im Namespace `argocd` und deployed in den Namespace `guestbook-impersonation`.

## Enthaltene Datei

- `argocd-impersonation-demo.yaml`

## Was die Demo konfiguriert

Die YAML-Datei legt diese Ressourcen an:

- `ConfigMap/argocd-cm`
- `ClusterRole/argocd-application-controller-impersonation`
- `ClusterRoleBinding/argocd-application-controller-impersonation`
- `Namespace/guestbook-impersonation`
- `ServiceAccount/guestbook-deployer`
- `Role/guestbook-deployer-role`
- `RoleBinding/guestbook-deployer-rolebinding`
- `AppProject/impersonation-demo`
- `Application/guestbook-impersonation-demo`

## Was genau passiert

### 1. Impersonation wird in Argo CD aktiviert

In `argocd-cm` wird diese Option gesetzt:

```yaml
application.sync.impersonation.enabled: "true"
```

Damit darf Argo CD beim Sync statt seiner normalen Identitaet einen Ziel-`ServiceAccount` verwenden.

### 2. Der Application Controller bekommt das Recht zur Impersonation

Der `argocd-application-controller` braucht dafuer auf Kubernetes-Seite zusaetzliche Rechte:

- `impersonate` auf `serviceaccounts`
- `create` auf `selfsubjectaccessreviews`

Ohne diese Rechte kann Argo CD nicht pruefen und nicht als Ziel-`ServiceAccount` arbeiten.

### 3. Im Ziel-Namespace wird ein eigener Deploy-Account angelegt

Im Namespace `guestbook-impersonation` wird dieser ServiceAccount erstellt:

- `guestbook-deployer`

Dieser Account repraesentiert die Identitaet, mit der Argo CD dort deployen soll.

### 4. Dieser Deploy-Account bekommt nur begrenzte Rechte

Per `Role` und `RoleBinding` bekommt `guestbook-deployer` nur die Rechte, die fuer die Demo noetig sind:

- `Deployment`
- `ReplicaSet`
- `Service`
- Leserechte auf `Pod`

Das ist der eigentliche Sicherheitsgewinn:

Argo CD deployed im Ziel-Namespace nicht mit globalen Controller-Rechten, sondern nur mit dem, was `guestbook-deployer` dort darf.

### 5. Das AppProject verbindet Ziel und ServiceAccount

Im `AppProject` steht:

- Zielcluster: `https://kubernetes.default.svc`
- Ziel-Namespace: `guestbook-impersonation`
- Ziel-ServiceAccount: `guestbook-deployer`

Das passiert ueber `destinationServiceAccounts`:

```yaml
destinationServiceAccounts:
  - server: https://kubernetes.default.svc
    namespace: guestbook-impersonation
    defaultServiceAccount: guestbook-deployer
```

Die Bedeutung ist:

Wenn eine Argo-CD-`Application` in diesem Projekt nach `guestbook-impersonation` deployed, dann soll der Sync als `guestbook-deployer` ausgefuehrt werden.

### 6. Die Application deployed eine Demo-App

Die `Application` `guestbook-impersonation-demo` zeigt auf das Beispiel-Repo:

- Repo: `https://github.com/argoproj/argocd-example-apps.git`
- Pfad: `guestbook`

Beim Sync passiert dann logisch folgendes:

1. Argo CD liest das Git-Repo.
2. Argo CD erkennt das Ziel `https://kubernetes.default.svc` und `guestbook-impersonation`.
3. Argo CD sucht im `AppProject` den passenden `destinationServiceAccount`.
4. Der `application-controller` fuehrt den Sync im Cluster als `guestbook-deployer` aus.
5. Kubernetes prueft nur die Rechte von `guestbook-deployer`.

## Warum das nuetzlich ist

Ohne Impersonation deployed Argo CD normalerweise mit den Rechten seines Controllers.

Mit Impersonation kannst du festlegen:

- welcher Namespace mit welchem ServiceAccount bespielt wird
- dass Teams oder Projekte nur die Rechte bekommen, die sie wirklich brauchen
- dass zentrale Argo-CD-Installationen kontrolliert auf Zielcluster deployen

Das ist besonders wichtig fuer:

- zentrale Argo-CD-Installationen
- Multi-Cluster-Setups
- Mandantentrennung
- Least-Privilege-RBAC

## Demo anwenden

```bash
kubectl apply -f /Users/opolat/Documents/Projekte/kubernetes-deployment/argocd-impersonation-demo.yaml
kubectl rollout restart deployment argocd-application-controller -n argocd
kubectl rollout status deployment argocd-application-controller -n argocd
```

Danach kannst du pruefen:

```bash
kubectl get appproject -n argocd impersonation-demo
kubectl get application -n argocd guestbook-impersonation-demo
kubectl get all -n guestbook-impersonation
```

## Woran du erkennst, dass es funktioniert

Wenn die Demo funktioniert:

- die `Application` wird erfolgreich synchronisiert
- im Namespace `guestbook-impersonation` erscheinen `Deployment`, `ReplicaSet`, `Pod` und `Service`
- Argo CD kann nur das anlegen, was der `guestbook-deployer` laut `Role` darf

Wenn du zum Beispiel in der `Role` absichtlich Rechte entfernst, wird der Sync fehlschlagen. Das ist ein guter Test dafuer, dass wirklich die Rechte des Ziel-`ServiceAccount` verwendet werden.

## Wichtiger Hinweis zum Controller-ServiceAccount

In der Demo ist dieser Subject eingetragen:

```yaml
subjects:
  - kind: ServiceAccount
    name: argocd-application-controller
    namespace: argocd
```

Das passt fuer viele Standard-Installationen.

Wenn dein Argo-CD-Release anders heisst, kann der echte ServiceAccount anders heissen, zum Beispiel:

- `argocd-argocd-application-controller`
- `<release-name>-application-controller`

Dann musst du den Namen im `ClusterRoleBinding` anpassen.

Pruefen kannst du das so:

```bash
kubectl get sa -n argocd
```

## Typische Fehler

### Die Application synchronisiert nicht

Moegliche Ursachen:

- `application.sync.impersonation.enabled` ist nicht gesetzt
- `argocd-application-controller` wurde nach der Aenderung nicht neu gestartet
- der falsche Controller-ServiceAccount ist im `ClusterRoleBinding` eingetragen
- `guestbook-deployer` hat im Ziel-Namespace nicht genug Rechte

### Es wird trotzdem mit zu vielen Rechten deployed

Dann wird das Feature oft noch nicht verwendet oder das `AppProject` passt nicht zur `Application`.

Wichtig ist:

- `spec.project` der `Application` muss `impersonation-demo` sein
- `destination.server` und `destination.namespace` muessen genau auf den Eintrag in `destinationServiceAccounts` passen

## Datei

- [argocd-impersonation-demo.yaml](/Users/opolat/Documents/Projekte/kubernetes-deployment/argocd-impersonation-demo.yaml)
