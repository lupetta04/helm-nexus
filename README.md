# Nexus Helm Chart

Helm chart per l'installazione di Sonatype Nexus Repository Manager su Kubernetes.

## Descrizione

Questo repository contiene un Helm chart per il deployment di Nexus Repository Manager su un singolo pod in un cluster Kubernetes, insieme a una pipeline Azure DevOps per automatizzare il deployment su AKS (Azure Kubernetes Service).

## Struttura del Progetto

```
.
├── nexus-chart/              # Helm chart per Nexus
│   ├── Chart.yaml           # Metadati del chart
│   ├── values.yaml          # Valori di configurazione di default
│   ├── templates/           # Template Kubernetes
│   │   ├── _helpers.tpl     # Helper template
│   │   ├── deployment.yaml  # Deployment (single pod)
│   │   ├── service.yaml     # Service
│   │   ├── pvc.yaml         # PersistentVolumeClaim
│   │   ├── serviceaccount.yaml
│   │   └── ingress.yaml     # Ingress (opzionale)
│   └── .helmignore
└── azure-pipelines.yml      # Pipeline Azure DevOps
```

## Prerequisiti

- Kubernetes cluster (AKS o altro)
- Helm 3.x installato
- kubectl configurato per il cluster di destinazione
- Azure DevOps con service connection configurata (per la pipeline)

## Installazione Manuale

### 1. Installazione di base

```bash
helm install nexus ./nexus-chart
```

### 2. Installazione con namespace personalizzato

```bash
kubectl create namespace nexus-ns
helm install nexus ./nexus-chart --namespace nexus-ns
```

### 3. Installazione con valori personalizzati

```bash
helm install nexus ./nexus-chart \
  --set persistence.size=20Gi \
  --set resources.requests.memory=4Gi \
  --set service.type=LoadBalancer
```

### 4. Upgrade del release

```bash
helm upgrade nexus ./nexus-chart
```

### 5. Disinstallazione

```bash
helm uninstall nexus
```

## Configurazione

Il file `values.yaml` contiene tutte le opzioni di configurazione. Le principali sono:

### Immagine Docker

```yaml
image:
  repository: sonatype/nexus3
  tag: "3.65.0"
  pullPolicy: IfNotPresent
```

### Risorse

```yaml
resources:
  limits:
    cpu: 2000m
    memory: 4Gi
  requests:
    cpu: 500m
    memory: 2Gi
```

### Persistenza

```yaml
persistence:
  enabled: true
  storageClass: ""  # Usa la storage class di default del cluster
  size: 8Gi
  accessMode: ReadWriteOnce
```

### Service

```yaml
service:
  type: ClusterIP
  port: 8081
```

Per esporre Nexus esternamente, puoi usare:
- `type: LoadBalancer` - per creare un LoadBalancer esterno
- `type: NodePort` - per esporre su una porta del nodo
- Abilitare l'Ingress - per routing HTTP/HTTPS

### Ingress

```yaml
ingress:
  enabled: true
  className: "nginx"  # o altro ingress controller
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: nexus.example.com
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls:
    - secretName: nexus-tls
      hosts:
        - nexus.example.com
```

## Deployment con Azure DevOps Pipeline

La pipeline Azure DevOps automatizza il deployment del chart Nexus su un cluster AKS esistente.

### Configurazione della Pipeline

1. **Crea una Service Connection in Azure DevOps**:
   - Vai su Project Settings → Service connections
   - Crea una nuova Kubernetes service connection
   - Seleziona il tuo cluster AKS
   - Assegna un nome (es. "aks-nexus-connection")

2. **Configura la Pipeline**:
   - In Azure DevOps, vai su Pipelines → Create Pipeline
   - Seleziona il repository GitHub
   - Scegli "Existing Azure Pipelines YAML file"
   - Seleziona `azure-pipelines.yml`

3. **Esegui la Pipeline con Parametri**:

La pipeline accetta i seguenti parametri:

| Parametro | Descrizione | Default |
|-----------|-------------|---------|
| `kubernetesServiceConnection` | Nome della service connection verso AKS | (obbligatorio) |
| `namespace` | Namespace Kubernetes dove installare Nexus | `default` |
| `releaseName` | Nome del release Helm | `nexus` |
| `chartPath` | Path del chart Helm nel repository | `nexus-chart` |

### Esecuzione della Pipeline

Quando esegui la pipeline, ti verrà chiesto di fornire:

1. **Kubernetes Service Connection**: Il nome della tua service connection configurata
2. **Namespace**: Il namespace dove vuoi installare Nexus (verrà creato se non esiste)

Esempio di configurazione:
```yaml
kubernetesServiceConnection: 'aks-nexus-connection'
namespace: 'nexus-prod'
releaseName: 'nexus'
```

### Steps della Pipeline

1. **Install Helm**: Installa l'ultima versione di Helm
2. **Helm upgrade/install**: Esegue `helm upgrade --install` per deployare o aggiornare Nexus
3. **Get deployment status**: Verifica lo stato dei pod deployati

## Accesso a Nexus

Dopo l'installazione, per accedere a Nexus:

### 1. Port Forward (per test locale)

```bash
kubectl port-forward svc/nexus 8081:8081
```

Poi apri il browser su: http://localhost:8081

### 2. LoadBalancer (se configurato)

```bash
kubectl get svc nexus
```

Usa l'EXTERNAL-IP mostrato per accedere.

### 3. Ingress (se configurato)

Accedi tramite l'hostname configurato nell'Ingress (es. https://nexus.example.com)

### Credenziali di Default

- **Username**: `admin`
- **Password**: La password iniziale è salvata nel file `/nexus-data/admin.password` nel pod

Per recuperarla:

```bash
kubectl exec -it $(kubectl get pod -l app.kubernetes.io/name=nexus -o jsonpath='{.items[0].metadata.name}') -- cat /nexus-data/admin.password
```

**Importante**: Cambia la password dopo il primo accesso!

## Verifica del Deployment

```bash
# Verifica i pod
kubectl get pods -l app.kubernetes.io/name=nexus

# Verifica il service
kubectl get svc nexus

# Verifica il PVC
kubectl get pvc

# Visualizza i logs
kubectl logs -l app.kubernetes.io/name=nexus -f
```

## Troubleshooting

### Il pod non si avvia

```bash
# Controlla gli eventi
kubectl describe pod -l app.kubernetes.io/name=nexus

# Controlla i logs
kubectl logs -l app.kubernetes.io/name=nexus
```

### Problemi di persistenza

```bash
# Verifica il PVC
kubectl get pvc
kubectl describe pvc nexus-data

# Verifica che ci sia una storage class disponibile
kubectl get storageclass
```

### Nexus è lento all'avvio

Nexus può richiedere 2-3 minuti per avviarsi completamente. Le probe sono configurate per:
- `initialDelaySeconds: 120` per liveness
- `initialDelaySeconds: 60` per readiness

## Personalizzazioni Avanzate

### File values-custom.yaml

Crea un file `values-custom.yaml` con le tue personalizzazioni:

```yaml
persistence:
  size: 50Gi
  storageClass: "premium-ssd"

resources:
  limits:
    cpu: 4000m
    memory: 8Gi
  requests:
    cpu: 1000m
    memory: 4Gi

ingress:
  enabled: true
  className: "nginx"
  hosts:
    - host: nexus.mycompany.com
      paths:
        - path: /
          pathType: ImplementationSpecific
```

Poi installa con:

```bash
helm install nexus ./nexus-chart -f values-custom.yaml
```

## License

MIT

## Maintainer

lupetta04
