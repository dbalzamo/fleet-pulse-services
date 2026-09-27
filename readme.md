# Fleet Pulse — Kubernetes (k8s/)

> **Linguaggio**: italiano (semplice, pensato per essere compreso anche da un junior).

---

## Sommario

1. [Cos'è la cartella k8s/](#1-cosè-la-cartella-k8s)
2. [Prerequisiti](#2-prerequisiti)
3. [Concetti base di Kubernetes](#3-concetti-base-di-kubernetes)
   - [3.1 Cos'è Kubernetes](#31-cosè-kubernetes)
   - [3.2 Pod](#32-pod)
   - [3.3 Deployment](#33-deployment)
   - [3.4 Service](#34-service)
   - [3.5 ConfigMap](#35-configmap)
   - [3.6 Secret](#36-secret)
   - [3.7 Namespace](#37-namespace)
   - [3.8 PersistentVolumeClaim (PVC)](#38-persistentvolumeclaim-pvc)
   - [3.9 InitContainer](#39-initcontainer)
   - [3.10 Probe (Health Check)](#310-probe-health-check)
   - [3.11 Resource (Limiti CPU/Memoria)](#311-resource-limiti-cpumemoria)
   - [3.12 Kustomize](#312-kustomize)
4. [Struttura della cartella k8s/](#4-struttura-della-cartella-k8s)
5. [Template YAML spiegati riga per riga](#5-template-yaml-spiegati-riga-per-riga)
   - [5.1 Namespace](#51-namespace)
   - [5.2 ConfigMap](#52-configmap)
   - [5.3 Secret](#53-secret)
   - [5.4 Deployment](#54-deployment)
   - [5.5 Service](#55-service)
   - [5.6 PersistentVolumeClaim (PVC)](#56-persistentvolumeclaim-pvc)
   - [5.7 Kustomization](#57-kustomization)
6. [Guida pratica: deployare un microservizio](#6-guida-pratica-deployare-un-microservizio)
   - [Step 1 — Struttura delle cartelle](#61-step-1--struttura-delle-cartelle)
   - [Step 2 — Dockerfile e build dell'immagine](#62-step-2--dockerfile-e-build-dellimmagine)
   - [Step 3 — ConfigMap con le variabili d'ambiente](#63-step-3--configmap-con-le-variabili-dambiente)
   - [Step 4 — Secret con i dati sensibili](#64-step-4--secret-con-i-dati-sensibili)
   - [Step 5 — Deployment](#65-step-5--deployment)
   - [Step 6 — Service](#66-step-6--service)
   - [Step 7 — Aggiungere al kustomization.yaml](#67-step-7--aggiungere-al-kustomizationyaml)
   - [Step 8 — Deployare su Kubernetes](#68-step-8--deployare-su-kubernetes)
   - [Step 9 — Verificare che tutto funzioni](#69-step-9--verificare-che-tutto-funzioni)
7. [Ambiente locale con Minikube](#7-ambiente-locale-con-minikube)
8. [Errori comuni e soluzioni](#8-errori-comuni-e-soluzioni)
9. [Riferimenti](#9-riferimenti)

---

## 1. Cos'è la cartella k8s/

La cartella `k8s/` contiene tutti i file di configurazione per deployare i microservizi del progetto su **Kubernetes** (abbreviato **k8s**).

Qui dentro trovi:
- I template YAML di ogni componente (Deployment, Service, ConfigMap, Secret...)
- La configurazione per **Kustomize** (lo strumento che usiamo per gestire ambienti diversi)
- Le risorse condivise come PostgreSQL

Ogni microservizio ha la sua sottocartella con i propri file:
```
k8s/
  local/              ← ambiente di sviluppo (Minikube)
    namespace.yaml
    kustomization.yaml
    postgres/         ← database condiviso
    auth-service/     ← servizio di autenticazione
    api-gateway/      ← gateway API
    fleet-service/    ← servizio fleet (da creare)
  staging/            ← futuro ambiente di staging
  production/         ← futuro ambiente di produzione
```

---

## 2. Prerequisiti

Per seguire questa guida e usare i file in `k8s/`, hai bisogno di:

| Strumento | Versione | Perché |
|---|---|---|
| **Docker Desktop** | Ultima | Per costruire le immagini dei microservizi e avere il runtime Docker |
| **Minikube** | v1.30+ | Cluster Kubernetes locale per sviluppo |
| **kubectl** | v1.28+ | Client a riga di comando per interagire con Kubernetes |
| **Skaffold** (opzionale) | v2.x | Per sviluppo continuo (build + deploy automatico) |
| **Helm** (opzionale) | v3.x | Gestione pacchetti Kubernetes (non lo usiamo qui, ma e' diffuso) |

> **Cos'è Minikube?** E' un cluster Kubernetes per sviluppo che gira in un container Docker sulla tua macchina. Simula un vero cluster ma con un solo nodo.
>
> **Cos'è kubectl?** E' il comando che usi per dire a Kubernetes cosa fare: "crea questo pod", "mostrami i servizi", "cancella quel deployment".

---

## 3. Concetti base di Kubernetes

### 3.1 Cos'è Kubernetes

Kubernetes (k8s) e' un sistema che **orchestra container**. Immagina di avere 10 microservizi, ognuno con 3 repliche (30 container totali). Kubernetes:

- Decide **dove** far partire ogni container (su quale macchina)
- **Riavvia** i container che crashano
- **Bilancia** il traffico tra le varie repliche
- **Aggiorna** le versioni senza downtime (rolling update)
- **Scala** automaticamente in base al carico

Tutto questo lo fai scrivendo file YAML che descrivono **lo stato desiderato** del sistema, e Kubernetes fa il resto.

### 3.2 Pod

Il **Pod** e' l'unita' minima in Kubernetes. Un pod contiene uno o piu' container che condividono lo stesso IP e la stessa memoria.

**Nella pratica**: non crei mai Pod direttamente. Crei un **Deployment** che gestisce i pod per te.

### 3.3 Deployment

Il **Deployment** dice a Kubernetes:
- Quale immagine Docker usare (`image:`)
- Quante copie (repliche) far partire (`replicas:`)
- Come controllare se il container e' vivo (`livenessProbe`)
- Quanta CPU/RAM allocare (`resources`)
- Come aggiornare senza downtime

Se un pod crasha, il Deployment lo ricrea. Se vuoi 3 repliche e ne muore 1, ne crea un'altra.

### 3.4 Service

Il **Service** e' l'indirizzo stabile per parlare con un gruppo di pod. I pod hanno IP dinamici (cambiano quando vengono ricreati), il Service ha un IP fisso e un nome DNS.

**Esempio**: il Service `auth-service` risponde all'indirizzo `http://auth-service:8081` (Dentro il cluster Kubernetes, i nomi dei Service sono risolvibili come DNS).

Tipi di Service:
| Tipo | Descrizione | Quando usarlo |
|---|---|---|
| `ClusterIP` | IP visibile solo dentro il cluster | Comunicazione tra microservizi |
| `NodePort` | Espone su una porta del nodo (es. :30080) | Sviluppo e test |
| `LoadBalancer` | Crea un load balancer esterno (es. AWS ELB) | Produzione |

### 3.5 ConfigMap

La **ConfigMap** contiene **variabili d'ambiente NON sensibili**: URL del database, porta del server, percorsi, ecc.

E' separata dal Deployment cosi' puoi cambiare la configurazione senza ricostruire l'immagine Docker.

### 3.6 Secret

Il **Secret** e' come ConfigMap ma per dati **sensibili**: password, chiavi JWT, token API.

I dati in un Secret sono codificati in Base64 (non cifrati, solo offuscati — per vera cifratura serve tool come SOPS o SealedSecrets).

### 3.7 Namespace

Il **Namespace** e' una "cartella" virtuale dentro il cluster per separare ambienti o team:
- `fleet-pulse` — namespace della nostra applicazione
- `kube-system` — namespace dei componenti interni di Kubernetes

### 3.8 PersistentVolumeClaim (PVC)

La **PVC** chiede spazio disco persistente a Kubernetes. Serve per database come PostgreSQL che devono mantenere i dati anche quando il pod viene ricreato.

### 3.9 InitContainer

L'**InitContainer** e' un container che parte PRIMA del container principale. Serve per:
- Aspettare che un servizio sia pronto (es. PostgreSQL)
- Preparare file di configurazione
- Migrare il database

Nel nostro progetto, l'auth-service ha un InitContainer che aspetta PostgreSQL:
```yaml
initContainers:
  - name: wait-for-postgres
    command: ["until pg_isready -h postgres; do sleep 2; done"]
```

### 3.10 Probe (Health Check)

Kubernetes controlla la salute del container con tre tipi di probe:

| Probe | Cosa controlla | Cosa fa se fallisce |
|---|---|---|
| `startupProbe` | Il container e' partito? | Non invia traffico finche' non passa |
| `readinessProbe` | Il container e' pronto per ricevere richieste? | Toglie il pod dal Service (non riceve traffico) |
| `livenessProbe` | Il container e' ancora vivo? | Riavvia il pod |

Le probe possono essere TCP (prova a connettersi a una porta), HTTP (chiama un endpoint) o exec (esegue un comando).

### 3.11 Resource (Limiti CPU/Memoria)

I **limiti di risorse** impediscono a un container di consumare tutta la CPU o RAM del nodo:

```yaml
resources:
  requests:    # Quanto garantisci al container (minimo)
    memory: 512Mi
    cpu: 250m    # 0.25 CPU core
  limits:      # Quanto al massimo puo' usare
    memory: 768Mi
    cpu: 500m    # 0.5 CPU core
```

- **requests**: Kubernetes usa questo valore per decidere su quale nodo far partire il pod (deve avere almeno tanta memoria libera)
- **limits**: se il container supera questo limite, viene fermato (OOMKilled) o throttlato

### 3.12 Kustomize

**Kustomize** e' uno strumento integrato in kubectl per gestire file YAML senza usare template. Permette di:

- Definire le risorse base in una cartella
- Overridare configurazioni per ambiente (locale, staging, produzione)
- Aggiungere prefissi/suffissi ai nomi

Il file `kustomization.yaml` elenca tutte le risorse da applicare:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: fleet-pulse
resources:
  - namespace.yaml
  - postgres/deployment.yaml
  - auth-service/deployment.yaml
  - api-gateway/deployment.yaml
```

---

## 4. Struttura della cartella k8s/

```
k8s/
├── local/                                    ← ambiente: sviluppo locale (Minikube)
│   ├── namespace.yaml                        ← namespace dell'applicazione
│   ├── kustomization.yaml                    ← elenco di tutte le risorse da applicare
│   │
│   ├── postgres/                             ← database PostgreSQL
│   │   ├── secret.yaml                       │   password e username
│   │   ├── pvc.yaml                          │   spazio disco persistente
│   │   ├── configmap.yaml (init)             │   script SQL iniziale
│   │   ├── deployment.yaml                   │   come far partire PostgreSQL
│   │   └── service.yaml                      │   indirizzo: postgres:5432
│   │
│   ├── auth-service/                         ← microservizio di autenticazione
│   │   ├── configmap.yaml                    │   URL database, porta
│   │   ├── secret.yaml                       │   password DB, chiave JWT
│   │   ├── deployment.yaml                   │   come far partire auth-service
│   │   └── service.yaml                      │   indirizzo: auth-service:8081
│   │
│   ├── api-gateway/                          ← gateway API
│   │   ├── configmap.yaml                    │   porta, URL auth-service
│   │   ├── secret.yaml                       │   chiave JWT (identica a auth-service)
│   │   ├── deployment.yaml                   │   come far partire gateway
│   │   └── service.yaml                      │   indirizzo: api-gateway:8080
│   │
│   └── fleet-service/                        ← (da creare)
│       └── ...
│
├── staging/                                  ← (futuro) ambiente di staging
└── production/                               ← (futuro) ambiente di produzione
```

**Perche' locale/ invece di mettere tutto in k8s/?** Perche' cosi' puoi avere configurazioni diverse per ambienti diversi (es. produzione con 3 repliche, locale con 1).

---

## 5. Template YAML spiegati riga per riga

### 5.1 Namespace

```yaml
apiVersion: v1              # Versione dell'API Kubernetes
kind: Namespace             # Tipo di risorsa
metadata:
  name: fleet-pulse         # Nome del namespace
```

Applicazione: `kubectl apply -f local/namespace.yaml`

### 5.2 ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-service-config          # Nome del ConfigMap (riferito dal Deployment)
  namespace: fleet-pulse
data:
  SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/robofleet_db
  SERVER_PORT: "8081"
```

- `data:` contiene coppie chiave-valore
- I valori diventano **variabili d'ambiente** nel container (grazie a `envFrom.configMapRef` nel Deployment)
- Ogni microservizio ha la sua ConfigMap

### 5.3 Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: auth-service-secret
  namespace: fleet-pulse
type: Opaque                           # Secret generico (non TLS)
stringData:                            # Valori in chiaro (k8s li codifica in Base64 da solo)
  DB_USERNAME: admin
  DB_PASSWORD: admin
  JWT_SECRET: ZmxlZXQtcHVsc2UtbG9jYWwtand0LXNlY3JldC1rZXkhIQ==
```

- `stringData:` accetta valori in chiaro (comodo per sviluppo)
- In produzione: mai mettere password in chiaro nei file YAML! Usa tool come **SealedSecrets** o **SOPS**
- Il Deployment carica i secret con `envFrom.secretRef`

### 5.4 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service                              # Nome del deployment
  namespace: fleet-pulse
  labels:
    app: auth-service                             # Etichetta per identificare il deployment
spec:
  replicas: 1                                     # Quante copie (pod) far partire
  selector:
    matchLabels:
      app: auth-service                           # Seleziona i pod con questa etichetta
  template:
    metadata:
      labels:
        app: auth-service                         # Etichetta sui pod (match con selector)
    spec:
      initContainers:                             # Container che parte PRIMA del principale
        - name: wait-for-postgres
          image: postgres:16-alpine
          command: ["sh", "-c", "until pg_isready -h postgres; do sleep 2; done"]
      containers:
        - name: auth-service                      # Nome del container (visibile nei log)
          image: auth-service:local               # Immagine Docker (nome:tag)
          imagePullPolicy: IfNotPresent           # Scarica solo se non presente in locale
          ports:
            - containerPort: 8081                 # Porta che il container espone
          envFrom:                                # Carica variabili d'ambiente DA:
            - configMapRef:
                name: auth-service-config         # ← ConfigMap
            - secretRef:
                name: auth-service-secret         # ← Secret
          startupProbe:                           # Il container e' partito?
            tcpSocket:
              port: 8081
            failureThreshold: 30
            periodSeconds: 5
          readinessProbe:                         # Il container e' pronto?
            tcpSocket:
              port: 8081
            periodSeconds: 10
          livenessProbe:                          # Il container e' vivo?
            tcpSocket:
              port: 8081
            periodSeconds: 30
          resources:
            requests:                             # Minimo garantito
              memory: 512Mi
              cpu: 250m
            limits:                               # Massimo consentito
              memory: 768Mi
              cpu: 500m
```

**Campi obbligatori** che vanno cambiati per ogni microservizio:
| Campo | Cosa mettere |
|---|---|
| `metadata.name` | Nome del servizio (es. `fleet-service`) |
| `spec.selector.matchLabels.app` | Etichetta che identifica i pod (match con `template.metadata.labels.app`) |
| `spec.template.metadata.labels.app` | Stessa etichetta sopra |
| `spec.template.spec.containers[0].image` | `nome-immagine:tag` (es. `fleet-service:local`) |
| `spec.template.spec.containers[0].ports.containerPort` | Porta HTTP del servizio (es. `8082`) |
| `spec.template.spec.containers[0].envFrom[*].name` | Nome della ConfigMap e del Secret |
| `spec.template.spec.containers[0].resources` | CPU/RAM in base ai requisiti del servizio |

### 5.5 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-service               # Nome DNS: http://auth-service:8081
  namespace: fleet-pulse
  labels:
    app: auth-service
spec:
  type: ClusterIP                  # Visibile solo dentro il cluster
  selector:
    app: auth-service              # Inoltra il traffico ai pod con questa etichetta
  ports:
    - name: http
      port: 8081                   # Porta del Service (cosa vedono gli altri servizi)
      targetPort: 8081             # Porta del container (deve matchare con containerPort)
```

**Campi obbligatori**:
| Campo | Cosa mettere |
|---|---|
| `metadata.name` | Nome DNS del servizio (es. `fleet-service`) |
| `spec.selector.app` | Deve matchare con `template.metadata.labels.app` del Deployment |
| `spec.ports[0].port` | Porta del Service (cosa usano gli altri servizi per chiamarti) |
| `spec.ports[0].targetPort` | Porta del container (deve matchare con `containerPort` del Deployment) |

**Attenzione**: `port` e `targetPort` possono essere diversi (es. Service su `80`, container su `8080`). Nel nostro progetto sono uguali per semplicita'.

### 5.6 PersistentVolumeClaim (PVC)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: fleet-pulse
spec:
  accessModes:
    - ReadWriteOnce              # Un solo pod puo' scrivere
  resources:
    requests:
      storage: 1Gi               # Quanto spazio richiedere
```

La PVC serve solo per servizi che hanno stato (database). Per microservizi stateless (auth-service, gateway) NON serve.

### 5.7 Kustomization

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: fleet-pulse                           # Namespace di default per tutte le risorse

resources:
  - namespace.yaml
  - postgres/secret.yaml
  - postgres/statefulset.yaml
  - postgres/deployment.yaml
  - postgres/service.yaml
  - auth-service/configmap.yaml
  - auth-service/secret.yaml
  - auth-service/deployment.yaml
  - auth-service/service.yaml
  - api-gateway/configmap.yaml
  - api-gateway/secret.yaml
  - api-gateway/deployment.yaml
  - api-gateway/service.yaml
```

L'ordine delle risorse non conta — Kustomize le applica tutte. Pero' e' buona prassi mettere prima i servizi "base" (PostgreSQL) e poi quelli che dipendono da essi.

---

## 6. Guida pratica: deployare un microservizio

Questa guida ti spiega come aggiungere un nuovo microservizio (es. `fleet-service`) al cluster.

### 6.1 Step 1 — Struttura delle cartelle

Crea una cartella per il tuo microservizio:

```bash
mkdir -p k8s/local/fleet-service
```

All'interno creerai 4 file:
```
fleet-service/
    configmap.yaml     # Variabili d'ambiente non sensibili
    secret.yaml        # Password e chiavi
    deployment.yaml    # Come far partire il container
    service.yaml       # Indirizzo stabile
```

### 6.2 Step 2 — Dockerfile e build dell'immagine

Prima di deployare su Kubernetes, devi avere un'immagine Docker del tuo microservizio.

Nel progetto, ogni microservizio ha un `Dockerfile`. Per costruire l'immagine:

```bash
# Dalla cartella del microservizio
docker build -t fleet-service:local .

# Verifica che l'immagine esista
docker images | grep fleet-service
```

**Regola importante**: il nome dell'immagine nel `Dockerfile` buildato deve matchare con `image:` nel `deployment.yaml`. Per sviluppo locale usiamo `fleet-service:local` (con `imagePullPolicy: IfNotPresent`).

### 6.3 Step 3 — ConfigMap con le variabili d'ambiente

Crea `configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fleet-service-config
  namespace: fleet-pulse
data:
  SERVER_PORT: "8082"
  SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/robofleet_db
```

Metti qui TUTTO cio' che non e' segreto: URL, porte, configurazioni.

### 6.4 Step 4 — Secret con i dati sensibili

Crea `secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: fleet-service-secret
  namespace: fleet-pulse
type: Opaque
stringData:
  DB_USERNAME: admin
  DB_PASSWORD: admin
```

> **Per sviluppo locale**: va bene mettere le credenziali in chiaro. **MAI** in produzione — usa SealedSecrets, Vault, o le variabili d'ambiente della CI/CD.

### 6.5 Step 5 — Deployment

Crea `deployment.yaml` copiando da un servizio esistente (es. `auth-service/deployment.yaml`) e cambia:

| Cosa cambia | auth-service | fleet-service |
|---|---|---|
| `metadata.name` | `auth-service` | `fleet-service` |
| `selector.matchLabels.app` | `auth-service` | `fleet-service` |
| `template.metadata.labels.app` | `auth-service` | `fleet-service` |
| `containers[0].image` | `auth-service:local` | `fleet-service:local` |
| `containers[0].ports[0].containerPort` | `8081` | `8082` |
| `containers[0].envFrom[*].name` | `auth-service-config` / `auth-service-secret` | `fleet-service-config` / `fleet-service-secret` |
| `startupProbe.port` | `8081` | `8082` |
| `readinessProbe.port` | `8081` | `8082` |
| `livenessProbe.port` | `8081` | `8082` |

**Rimozione InitContainer**: se il nuovo servizio non dipende da PostgreSQL (o ne dipende ma non serve aspettarlo esplicitamente), togli la sezione `initContainers`.

### 6.6 Step 6 — Service

Crea `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleet-service
  namespace: fleet-pulse
  labels:
    app: fleet-service
spec:
  type: ClusterIP
  selector:
    app: fleet-service
  ports:
    - name: http
      port: 8082
      targetPort: 8082
```

**Cosa controllare**:
- `metadata.name` → sara' il nome DNS: `http://fleet-service:8082`
- `spec.selector.app` → deve matchare ESATTAMENTE con `template.metadata.labels.app` del deployment
- `spec.ports[0].targetPort` → deve matchare ESATTAMENTE con `containerPort` del deployment

### 6.7 Step 7 — Aggiungere al kustomization.yaml

Modifica `k8s/local/kustomization.yaml` e aggiungi le nuove risorse:

```yaml
resources:
  - namespace.yaml
  - postgres/...
  - auth-service/...
  - api-gateway/...
  - fleet-service/configmap.yaml     # ← NUOVO
  - fleet-service/secret.yaml        # ← NUOVO
  - fleet-service/deployment.yaml    # ← NUOVO
  - fleet-service/service.yaml       # ← NUOVO
```

### 6.8 Step 8 — Deployare su Kubernetes

Prima di deployare, assicurati che Minikube sia in esecuzione:

```bash
# Avvia Minikube (se non e' gia' in esecuzione)
minikube start --driver=docker

# Verifica che kubectl parli con Minikube
kubectl cluster-info
```

Poi applica tutte le risorse con Kustomize:

```bash
# Da k8s/local/
kubectl apply -k .
```

Questo comando:
1. Legge `kustomization.yaml`
2. Carica tutti i file YAML elencati
3. Applica il namespace a tutte le risorse
4. Invia tutto a Kubernetes

### 6.9 Step 9 — Verificare che tutto funzioni

```bash
# Stato dei pod
kubectl get pods -n fleet-pulse

# Log di un pod specifico
kubectl logs -n fleet-pulse deployment/fleet-service

# Stato dei servizi
kubectl get svc -n fleet-pulse

# Stato dei deployment
kubectl get deployment -n fleet-pulse

# Descrizione dettagliata (se un pod non parte)
kubectl describe pod -n fleet-pulse -l app=fleet-service
```

**Comandi utili**:
```bash
# Seguire i log in tempo reale
kubectl logs -n fleet-pulse -f deployment/fleet-service

# Entrare dentro un pod (debug)
kubectl exec -n fleet-pulse -it deployment/fleet-service -- sh

# Port forwarding (accedere a un servizio dal browser locale)
kubectl port-forward -n fleet-pulse svc/fleet-service 8082:8082
```

---

## 7. Ambiente locale con Minikube

Setup rapido per chi parte da zero:

```bash
# 1. Avvia Minikube
minikube start --driver=docker

# 2. Configura il terminale per usare il Docker di Minikube
eval $(minikube docker-env)

# 3. Build delle immagini dei microservizi
docker build -t auth-service:local auth-service/
docker build -t api-gateway:local fleet-gateway/

# 4. Deploya tutto
kubectl apply -k k8s/local/

# 5. Esponi il gateway (se devi testare dal browser)
kubectl port-forward -n fleet-pulse svc/api-gateway 8080:8080

# 6. Apri http://localhost:8080 nel browser
```

**Importante**: `eval $(minikube docker-env)` fa in modo che `docker build` costruisca le immagini DENTRO Minikube, non nel Docker Desktop locale. Se salti questo passaggio, Minikube non trova le immagini.

---

## 8. Errori comuni e soluzioni

### Pod in stato `ImagePullBackOff` o `ErrImageNeverPull`

**Causa**: Kubernetes non trova l'immagine Docker.

**Soluzione**:
```bash
# 1. Verifica di essere nell'environment Docker giusto
eval $(minikube docker-env)

# 2. Ricostruisci l'immagine
docker build -t auth-service:local auth-service/

# 3. Oppure cambia imagePullPolicy in "Always" e usa un registry
```

### Pod in stato `CrashLoopBackOff`

**Causa**: Il container parte e crasha subito.

**Soluzione**: Controlla i log
```bash
kubectl logs -n fleet-pulse deployment/auth-service --previous
```

Cause tipiche:
- Variabili d'ambiente mancanti (ConfigMap/Secret non trovati)
- Database non raggiungibile (InitContainer fallito)
- Porta sbagliata nel `containerPort`

### InitContainer non passa mai

**Causa**: Il servizio da cui dipende non e' pronto.

**Soluzione**:
```bash
# Controlla se PostgreSQL e' partito
kubectl get pods -n fleet-pulse -l app=postgres

# Controlla i log dell'init container
kubectl logs -n fleet-pulse deployment/auth-service -c wait-for-postgres
```

### `kubectl apply -k .` non trova i file

**Causa**: Sei nella cartella sbagliata.

**Soluzione**: Assicurati di essere in `k8s/local/` e che `kustomization.yaml` esista:
```bash
cd k8s/local/
kubectl apply -k .
```

### Service non risponde

**Causa**: `selector.app` del Service non matcha con `labels.app` del Deployment.

**Soluzione**: Controlla che i due valori siano IDENTICI:
```bash
kubectl describe svc -n fleet-pulse fleet-service   # Mostra il selector
kubectl get pods -n fleet-pulse --show-labels        # Mostra le labels dei pod
```

### Port forwarding non funziona

**Causa**: Il pod non e' in stato `Running`.

**Soluzione**:
```bash
kubectl get pods -n fleet-pulse
# Se non e' Running, usa kubectl describe per capire perche'
```

---

## 9. Riferimenti

| File | Percorso |
|---|---|
| Namespace | `k8s/local/namespace.yaml` |
| Kustomization | `k8s/local/kustomization.yaml` |
| PostgreSQL deployment | `k8s/local/postgres/deployment.yaml` |
| PostgreSQL service | `k8s/local/postgres/service.yaml` |
| Auth-service deployment | `k8s/local/auth-service/deployment.yaml` |
| Auth-service service | `k8s/local/auth-service/service.yaml` |
| Auth-service ConfigMap | `k8s/local/auth-service/configmap.yaml` |
| Auth-service Secret | `k8s/local/auth-service/secret.yaml` |
| API Gateway deployment | `k8s/local/api-gateway/deployment.yaml` |
| API Gateway service | `k8s/local/api-gateway/service.yaml` |
| API Gateway ConfigMap | `k8s/local/api-gateway/configmap.yaml` |
| API Gateway Secret | `k8s/local/api-gateway/secret.yaml` |

**Documentazione esterna**:
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Kustomize](https://kustomize.io/)
- [Minikube](https://minikube.sigs.k8s.io/docs/)
