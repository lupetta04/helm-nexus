# Quick Start Guide - Guida Rapida

## Installazione Manuale / Manual Installation

### Installazione base
```bash
# Clone del repository
git clone https://github.com/lupetta04/helm-nexus.git
cd helm-nexus

# Installazione con Helm
helm install nexus ./nexus-chart

# Verifica lo stato
kubectl get pods -l app.kubernetes.io/name=nexus
```

### Accesso a Nexus
```bash
# Port forward per accesso locale
kubectl port-forward svc/nexus 8081:8081

# Apri il browser su: http://localhost:8081

# Recupera la password iniziale di admin
kubectl exec -it $(kubectl get pod -l app.kubernetes.io/name=nexus -o jsonpath='{.items[0].metadata.name}') -- cat /nexus-data/admin.password
```

## Pipeline Azure DevOps

### Setup Iniziale

1. **Crea Service Connection in Azure DevOps**
   - Project Settings → Service connections
   - New service connection → Kubernetes
   - Seleziona il tuo cluster AKS
   - Nome: `aks-nexus-connection`

2. **Configura Pipeline**
   - Pipelines → New Pipeline
   - Seleziona GitHub repository
   - Use existing Azure Pipelines YAML file
   - Seleziona: `azure-pipelines.yml`

3. **Esegui Pipeline con Parametri**
   - `kubernetesServiceConnection`: `aks-nexus-connection`
   - `namespace`: `nexus-prod` (o altro namespace)
   - `releaseName`: `nexus`

### Esempio di Esecuzione

```yaml
# Parametri della pipeline
kubernetesServiceConnection: 'aks-nexus-connection'
namespace: 'nexus-prod'
releaseName: 'nexus'
chartPath: 'nexus-chart'
```

## Configurazioni Comuni / Common Configurations

### Aumentare la memoria
```bash
helm install nexus ./nexus-chart \
  --set resources.requests.memory=4Gi \
  --set resources.limits.memory=8Gi
```

### Aumentare lo storage
```bash
helm install nexus ./nexus-chart \
  --set persistence.size=50Gi
```

### Esporre con LoadBalancer
```bash
helm install nexus ./nexus-chart \
  --set service.type=LoadBalancer
```

### Abilitare Ingress
```bash
helm install nexus ./nexus-chart \
  --set ingress.enabled=true \
  --set ingress.hosts[0].host=nexus.example.com \
  --set ingress.hosts[0].paths[0].path=/ \
  --set ingress.hosts[0].paths[0].pathType=ImplementationSpecific
```

## Comandi Utili / Useful Commands

```bash
# Lista release Helm
helm list

# Status del release
helm status nexus

# Upgrade del release
helm upgrade nexus ./nexus-chart

# Rollback
helm rollback nexus

# Disinstallazione
helm uninstall nexus

# Visualizza logs
kubectl logs -l app.kubernetes.io/name=nexus -f

# Descrivi il pod
kubectl describe pod -l app.kubernetes.io/name=nexus

# Verifica PVC
kubectl get pvc

# Connetti al pod
kubectl exec -it $(kubectl get pod -l app.kubernetes.io/name=nexus -o jsonpath='{.items[0].metadata.name}') -- /bin/bash
```

## Troubleshooting

### Pod in CrashLoopBackOff
```bash
# Controlla i logs
kubectl logs -l app.kubernetes.io/name=nexus --previous

# Verifica gli eventi
kubectl get events --sort-by='.lastTimestamp'
```

### Problema di persistenza
```bash
# Verifica storage class disponibili
kubectl get storageclass

# Se necessario, specifica una storage class
helm upgrade nexus ./nexus-chart --set persistence.storageClass=standard
```

### Nexus non risponde
```bash
# Verifica che il pod sia Ready
kubectl get pods -l app.kubernetes.io/name=nexus

# Nexus richiede 2-3 minuti per l'avvio completo
# Controlla i logs per verificare l'avvio
kubectl logs -l app.kubernetes.io/name=nexus -f
```

## Credenziali Default

- **Username**: `admin`
- **Password**: Recuperala con:
  ```bash
  kubectl exec -it $(kubectl get pod -l app.kubernetes.io/name=nexus -o jsonpath='{.items[0].metadata.name}') -- cat /nexus-data/admin.password
  ```

**IMPORTANTE**: Cambia la password dopo il primo accesso!
