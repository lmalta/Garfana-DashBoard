# Dashboard Grafana + Prometheus pour Kubernetes (Helm)

![description](images/grafana_dashboard_preview.png)

## 📘 Description
Ce projet déploie un système complet de surveillance pour un cluster Kubernetes, utilisant **Prometheus** pour la collecte des métriques et **Grafana** pour la visualisation.  
Le tableau de bord fournit :

- État des pods, nœuds et déploiements  
- Utilisation CPU / mémoire / disque  
- Métriques réseau  
- Alertes configurables sur les ressources critiques  

---

## 📦 Déploiement du stack Prometheus + Grafana

### 1️⃣ Pré-requis
- Helm 3.x installé  
- Cluster Kubernetes actif  

### 2️⃣ Création d'un namespace dédié

```bash
kubectl create namespace monitoring
```

3️⃣ Création du fichier values.yaml
```yaml
grafana:
  adminUser: admin
  adminPassword: changeme
  service:
    type: ClusterIP
    port: 80
  ingress:
    enabled: false

prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false
    evaluationInterval: "30s"
    scrapeInterval: "30s"

alertmanager:
  alertmanagerSpec:
    replicas: 1

kubeStateMetrics:
  enabled: true

nodeExporter:
  enabled: true
```

4️⃣ Installation via Helm
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring-stack prometheus-community/kube-prometheus-stack -n monitoring -f values.yaml
```

Vérification des pods :
```bash
kubectl --namespace monitoring get pods -l "release=monitoring-stack"
```

🌐 Accès aux interfaces
Grafana

Lancer le port-forward :
```bash
kubectl -n monitoring port-forward svc/monitoring-stack-grafana 3000:80
```

Ouvrir ensuite : http://localhost:3000

Récupération du mot de passe admin de Grafana

Méthode classique :
```bash
kubectl --namespace monitoring get secrets monitoring-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

Méthode alternative via label admin-secret :
```bash
kubectl get secret --namespace monitoring -l app.kubernetes.io/component=admin-secret -o jsonpath="{.items[0].data.admin-password}" | base64 --decode ; echo
```

Il est recommandé de changer le mot de passe à la première connexion.

Prometheus (optionnel)
```bash
kubectl -n monitoring port-forward svc/monitoring-stack-kube-prom-prometheus 9090:9090
```

Ouvrir : http://localhost:9090

📊 Dashboards Grafana pré-configurés

Le chart kube-prometheus-stack inclut automatiquement :
Kubernetes Cluster Monitoring
Node Exporter Full
Pods / Deployments CPU & Memory

Alerts Overview
Il est également possible d’importer ou d’exporter des dashboards depuis Grafana.com.

🔔 Alerting intégré

Alertes CPU/mémoire pour pods et nœuds
Alertes pour pods crashés ou hors ligne
Alertes configurables via l’interface Grafana ou Alertmanager

🧹 Suppression

Pour supprimer le stack :
```bash
helm uninstall monitoring-stack -n monitoring
kubectl delete namespace monitoring
```

🔑 Gestion du mot de passe admin Grafana
1️⃣ Changement temporaire via l’interface Grafana

Se connecter à Grafana avec le mot de passe actuel
Aller dans Configuration → Utilisateurs → admin → Modifier le mot de passe
Entrer le nouveau mot de passe

Limite : le mot de passe sera écrasé si le pod Grafana est recréé ou si Helm est relancé.

2️⃣ Changement persistant via Secret Kubernetes

Encoder le nouveau mot de passe en base64 :
```bash
echo -n 'NouveauMdp123' | base64
```

Mettre à jour le secret Grafana :
```bash
kubectl -n monitoring patch secret monitoring-stack-grafana \
  -p '{"data":{"admin-password":"Tm91dmVhdURkcDEyMw=="}}'
```

Redémarrer le pod Grafana :
```bash
kubectl -n monitoring delete pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=monitoring-stack"
```

3️⃣ Changement persistant via Helm values.yaml

Ajouter dans values.yaml :
```yaml
grafana:
  adminPassword: NouveauMdp123
```

Puis mettre à jour le chart :
```bash
helm upgrade monitoring-stack prometheus-community/kube-prometheus-stack -n monitoring -f values.yaml
```

⚠️ Problème connu sur macOS M1/M2

Le pod monitoring-stack-prometheus-node-exporter peut être en CrashLoopBackOff.

Cause : Node Exporter tente d’accéder à des chemins host non disponibles sur Docker Desktop.

Diagnostic : consulter les logs :

kubectl -n monitoring logs monitoring-stack-prometheus-node-exporter-<pod-id>

Solution rapide : désactiver Node Exporter dans values.yaml :
```yaml
nodeExporter:
  enabled: false
```

Puis mettre à jour le release :
```bash
helm upgrade monitoring-stack prometheus-community/kube-prometheus-stack -n monitoring -f values.yaml
```

Option avancée (hostPath + privilèges) : possible mais non garanti sur Docker Desktop.

Recommandation : sur un cluster local Docker Desktop, désactiver Node Exporter et se concentrer sur kube-state-metrics et Prometheus.
