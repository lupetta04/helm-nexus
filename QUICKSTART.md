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

## Come Raggiungere il Servizio Nexus / How to Access Nexus Service

Dopo l'installazione su Kubernetes/AKS, esistono diversi metodi per raggiungere il servizio Nexus dall'esterno del cluster:

### Metodo 1: LoadBalancer (Consigliato per Azure AKS)

Il modo più semplice per esporre Nexus su Azure AKS è utilizzare un **LoadBalancer**, che creerà automaticamente un IP pubblico in Azure.

#### Installazione con LoadBalancer:
```bash
# Durante l'installazione
helm install nexus ./nexus-chart --set service.type=LoadBalancer

# Oppure upgrade di un'installazione esistente
helm upgrade nexus ./nexus-chart --set service.type=LoadBalancer
```

#### Recuperare l'IP pubblico:
```bash
# Attendi che l'EXTERNAL-IP sia assegnato (potrebbero volerci 1-2 minuti)
kubectl get svc nexus -w

# Output esempio:
# NAME    TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)          AGE
# nexus   LoadBalancer   10.0.123.45    20.123.45.67     8081:30123/TCP   2m
```

#### Accesso dal browser:
Una volta ottenuto l'EXTERNAL-IP (es. `20.123.45.67`), puoi accedere a Nexus:
```
http://20.123.45.67:8081
```

#### Trovare l'IP nel portale Azure:
1. Accedi al [Portale Azure](https://portal.azure.com)
2. Vai a **Tutti i servizi** → **Gruppi di risorse**
3. Seleziona il gruppo di risorse del cluster AKS (di solito `MC_<resource-group>_<cluster-name>_<region>`)
4. Troverai una risorsa di tipo **Indirizzo IP pubblico** creata automaticamente
5. L'indirizzo IP pubblico sarà visibile nella panoramica della risorsa

**Nota**: Con LoadBalancer, Azure creerà automaticamente un IP pubblico e lo assocerà al servizio. Questo ha un costo aggiuntivo in Azure.

### Metodo 2: Ingress (Consigliato per ambienti di produzione)

L'Ingress permette di esporre Nexus tramite un nome di dominio con supporto HTTPS/TLS.

#### Prerequisiti:
- Un Ingress Controller installato nel cluster (es. nginx-ingress)
- Un nome di dominio (es. `nexus.tuodominio.com`)
- Opzionale: cert-manager per certificati SSL automatici

#### Installazione con Ingress:
```bash
helm install nexus ./nexus-chart \
  --set ingress.enabled=true \
  --set ingress.className=nginx \
  --set ingress.hosts[0].host=nexus.tuodominio.com \
  --set ingress.hosts[0].paths[0].path=/ \
  --set ingress.hosts[0].paths[0].pathType=ImplementationSpecific
```

#### Con certificato SSL (usando cert-manager):
```bash
helm install nexus ./nexus-chart \
  --set ingress.enabled=true \
  --set ingress.className=nginx \
  --set ingress.annotations."cert-manager\.io/cluster-issuer"=letsencrypt-prod \
  --set ingress.hosts[0].host=nexus.tuodominio.com \
  --set ingress.hosts[0].paths[0].path=/ \
  --set ingress.hosts[0].paths[0].pathType=ImplementationSpecific \
  --set ingress.tls[0].secretName=nexus-tls \
  --set ingress.tls[0].hosts[0]=nexus.tuodominio.com
```

#### Trovare l'IP dell'Ingress Controller:
```bash
# Trova l'IP pubblico dell'Ingress Controller
kubectl get svc -n ingress-nginx ingress-nginx-controller

# Output esempio:
# NAME                       TYPE           EXTERNAL-IP      PORT(S)
# ingress-nginx-controller   LoadBalancer   20.234.56.78     80:31234/TCP,443:31456/TCP
```

#### Configurazione DNS:
Crea un record DNS A che punta `nexus.tuodominio.com` all'EXTERNAL-IP dell'Ingress Controller.

#### Accesso:
```
https://nexus.tuodominio.com
```

### Metodo 3: NodePort (Per ambienti di test)

NodePort espone il servizio su una porta specifica di ogni nodo del cluster.

#### Installazione con NodePort:
```bash
helm install nexus ./nexus-chart --set service.type=NodePort
```

#### Recuperare la porta assegnata:
```bash
kubectl get svc nexus

# Output esempio:
# NAME    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
# nexus   NodePort   10.0.123.45    <none>        8081:31234/TCP   1m
```

#### Trovare l'IP pubblico dei nodi in Azure:
```bash
# Ottieni l'IP pubblico di un nodo
kubectl get nodes -o wide

# Oppure, in Azure AKS, trova l'IP del Load Balancer del cluster
```

#### Accesso:
```
http://<NODE-IP>:31234
```

**Nota**: Con AKS, i nodi non hanno sempre IP pubblici diretti. Potrebbe essere necessario configurare il Network Security Group in Azure.

### Metodo 4: Port Forward (Solo per test locali)

Metodo più semplice per testare localmente, ma non utilizzabile per accesso remoto.

```bash
# Avvia il port forward
kubectl port-forward svc/nexus 8081:8081

# Accedi da browser locale
# http://localhost:8081
```

**Nota**: Il port forward termina quando chiudi il terminale ed è accessibile solo dalla tua macchina locale.

### Confronto dei Metodi / Methods Comparison

| Metodo | Pros | Cons | Costo Azure | Uso Consigliato |
|--------|------|------|-------------|-----------------|
| **LoadBalancer** | ✅ Semplice<br>✅ IP pubblico automatico<br>✅ Setup rapido | ❌ Costo IP pubblico<br>❌ No HTTPS nativo<br>❌ Un IP per servizio | 💰 Medio | Sviluppo, demo, piccoli team |
| **Ingress** | ✅ Un IP per più servizi<br>✅ HTTPS/TLS<br>✅ Routing per dominio<br>✅ Produzione-ready | ❌ Setup più complesso<br>❌ Richiede Ingress Controller<br>❌ Richiede dominio | 💰 Basso | **Produzione** |
| **NodePort** | ✅ Nessun costo IP pubblico<br>✅ Semplice | ❌ Porta ad alto numero<br>❌ Configurazione NSG necessaria<br>❌ Meno sicuro | 💰 Nessuno | Test, staging |
| **Port Forward** | ✅ Nessun costo<br>✅ Immediato | ❌ Solo accesso locale<br>❌ Non persistente | 💰 Nessuno | Debug, sviluppo locale |

### Raccomandazioni per Azure AKS / Recommendations for Azure AKS

**Per ambienti di sviluppo/test:**
```bash
# Usa LoadBalancer per semplicità
helm install nexus ./nexus-chart --set service.type=LoadBalancer
```

**Per ambienti di produzione:**
```bash
# Usa Ingress con SSL
# 1. Installa nginx-ingress controller
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace

# 2. Installa cert-manager (opzionale, per SSL automatico)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# 3. Installa Nexus con Ingress
helm install nexus ./nexus-chart \
  --set ingress.enabled=true \
  --set ingress.className=nginx \
  --set ingress.hosts[0].host=nexus.tuodominio.com \
  --set ingress.hosts[0].paths[0].path=/ \
  --set ingress.hosts[0].paths[0].pathType=ImplementationSpecific
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

## Sicurezza e Best Practices / Security and Best Practices

### Protezione dell'accesso esterno
Quando esponi Nexus all'esterno del cluster, considera le seguenti misure di sicurezza:

1. **Usa sempre HTTPS in produzione**
   ```bash
   # Configura Ingress con TLS
   --set ingress.tls[0].secretName=nexus-tls \
   --set ingress.tls[0].hosts[0]=nexus.tuodominio.com
   ```

2. **Limita l'accesso tramite Network Security Group (Azure)**
   - Nel portale Azure, configura le regole NSG per limitare l'accesso solo a IP/subnet autorizzati
   - Vai a: Risorsa IP Pubblico → Impostazioni → Gruppo di sicurezza di rete

3. **Abilita l'autenticazione forte**
   - Dopo il primo accesso, configura l'autenticazione LDAP/Active Directory in Nexus
   - Abilita la 2FA (Two-Factor Authentication) se disponibile

4. **Configura RBAC in Nexus**
   - Crea ruoli specifici per utenti e gruppi
   - Applica il principio del minimo privilegio

5. **Backup regolari**
   ```bash
   # Il PVC contiene tutti i dati di Nexus
   kubectl get pvc
   # Configura snapshot periodici del PVC in Azure
   ```

### Monitoraggio e Logging
```bash
# Monitora i logs di Nexus
kubectl logs -l app.kubernetes.io/name=nexus -f

# Configura Azure Monitor per il cluster AKS
# per raccogliere metriche e logs automaticamente
```

### Verifica dello stato del servizio
```bash
# Verifica che il servizio sia esposto correttamente
kubectl get svc nexus

# Verifica gli endpoint
kubectl get endpoints nexus

# Testa la connettività (da un pod nel cluster)
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- curl http://nexus:8081
```
