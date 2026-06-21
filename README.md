# Network Observability Operator — OpenShift

Equivalente comunitario a Kubeshark para analisis de trafico de red en OpenShift.
Instalacion basada en **community-operators** — no requiere suscripcion ni licencia Red Hat.

Referencia: https://docs.openshift.com/container-platform/latest/network_observability/installing-operators.html

## Comparacion con Kubeshark

| Caracteristica          | Kubeshark             | Network Observability Operator      |
|-------------------------|-----------------------|-------------------------------------|
| Captura de trafico      | eBPF + AF_PACKET      | eBPF (kernel hooks)                 |
| Soporte OpenShift       | Comunidad             | Comunidad (sin licencia requerida)  |
| Almacenamiento          | En memoria            | Loki (persistente)                  |
| Visualizacion           | UI propia             | OpenShift Console plugin            |
| Metricas Prometheus     | Limitado              | Nativas (netobserv_*)               |
| RBAC                    | Basico                | Integrado con OpenShift RBAC        |
| PacketDrop / DNS / RTT  | Parcial               | Si (eBPF features)                  |
| Licencia Red Hat        | No requerida          | No requerida (community-operators)  |

## Requisitos

- OpenShift 4.12 o superior
- Acceso como cluster-admin
- StorageClass disponible (para Loki)
- 3 worker nodes recomendados (minimo 1)
- Recursos minimos: 4 CPU / 8 GB RAM adicionales en el cluster
- **Sin suscripcion Red Hat** — usa `community-operators` del catalogo de OperatorHub

## Arquitectura

```
[Worker Nodes]
  └── eBPF Agent (DaemonSet) — captura flows en kernel space
        |
        | gRPC
        v
[flowlogs-pipeline] — enriquece con metadatos K8s
        |
        | HTTP/LogQL
        v
[LokiStack] — almacena flows en MinIO / S3
        |
        | LogQL queries
        v
[OpenShift Console Plugin] — visualizacion en la consola web
```

Ver `arch-diagram.svg` para el diagrama completo.

## Instalacion paso a paso

### Paso 1: Crear namespace base

```bash
oc apply -f manifests/01-namespace.yaml
```

Verificar:
```bash
oc get namespace netobserv
```

---

### Paso 2: Verificar OperatorGroup y instalar Loki Operator

El archivo `02-loki-operatorgroup.yaml` es solo documentativo — no hay que aplicarlo.
El namespace `openshift-operators` ya existe en OpenShift con un OperatorGroup global.

Confirmar que el OperatorGroup existe:
```bash
oc get operatorgroup -n openshift-operators
# Debe aparecer: global-operators
```

Verificar el canal disponible en tu version de OCP:
```bash
oc get packagemanifest loki-operator -n openshift-marketplace \
  -o jsonpath='{.status.channels[*].name}'
```

Actualizar el campo `channel` en `manifests/03-loki-subscription.yaml` si es necesario, luego aplicar:

```bash
oc apply -f manifests/03-loki-subscription.yaml
```

Esperar a que el operator este listo:
```bash
oc get csv -n openshift-operators | grep loki
# Esperar status: Succeeded
```

---

### Paso 3: Desplegar MinIO (object storage local)

> Solo para laboratorio. En produccion usar AWS S3, ODF, Azure Blob o GCS.

```bash
oc apply -f manifests/04-minio.yaml

# Esperar que el pod este Running
oc wait --for=condition=Available deployment/minio \
  -n netobserv --timeout=120s
```

---

### Paso 4: Crear secret de credenciales para Loki

```bash
oc apply -f manifests/05-loki-secret.yaml
```

---

### Paso 5: Crear el LokiStack

Verificar el StorageClass disponible en tu cluster:
```bash
oc get storageclass
```

Editar `manifests/06-lokistack.yaml` y actualizar `storageClassName` con el valor correcto, luego:

```bash
oc apply -f manifests/06-lokistack.yaml

# Monitorear el estado (puede tardar 3-5 minutos)
oc get lokistack -n netobserv loki -w
# Esperar: CONDITIONS Ready=True
```

---

### Paso 6: Instalar Network Observability Operator

```bash
# Verificar canal disponible (community-operators)
oc get packagemanifest netobserv-operator -n openshift-marketplace \
  -o jsonpath='{.status.channels[*].name}'
# Canales tipicos: community, stable

oc apply -f manifests/07-netobserv-subscription.yaml

# Esperar que el operator este listo
oc get csv -n openshift-operators | grep netobserv
# Esperar status: Succeeded
```

---

### Paso 7: Crear el FlowCollector

```bash
oc apply -f manifests/08-flowcollector.yaml

# Verificar que el agente eBPF este en todos los nodos
oc get daemonset -n netobserv
# netobserv-ebpf-agent debe tener DESIRED == READY

# Verificar el pipeline
oc get deployment -n netobserv
oc get pod -n netobserv
```

---

### Paso 8: Habilitar el plugin en la consola

El FlowCollector registra el plugin automaticamente. Verificar:

```bash
oc get consolePlugin netobserv-plugin
# Esperar que aparezca

# Si no esta habilitado automaticamente:
oc patch console cluster --type=merge \
  -p '{"spec":{"plugins":["netobserv-plugin"]}}'
```

Recargar la consola de OpenShift. Ir a: **Observe → Network Traffic**

---

## Desplegar proyecto de prueba

```bash
oc apply -f test-project/01-test-namespace.yaml
oc apply -f test-project/02-test-apps.yaml

# Verificar pods
oc get pod -n demo-netobserv

# Ver la URL del frontend
oc get route frontend -n demo-netobserv
```

El `traffic-generator` genera requests HTTP continuos entre los servicios automaticamente.
En la consola de OpenShift ir a **Observe → Network Traffic** y filtrar por namespace `demo-netobserv`.

---

## Verificacion post-instalacion

```bash
# Estado general de todos los componentes
oc get all -n netobserv

# FlowCollector status
oc get flowcollector cluster -o yaml | grep -A 20 status

# Logs del agente eBPF (un pod por nodo)
oc logs -n netobserv \
  $(oc get pod -n netobserv -l app=netobserv-ebpf-agent -o name | head -1)

# Logs del pipeline
oc logs -n netobserv deployment/flowlogs-pipeline --tail=50

# Metricas Prometheus
oc exec -n netobserv deployment/flowlogs-pipeline -- \
  curl -s localhost:9102/metrics | grep netobserv_node_ingress
```

---

## Troubleshooting

### El eBPF agent falla con error de privilegios

```bash
# Verificar el SCC asignado
oc get pod -n netobserv -l app=netobserv-ebpf-agent -o yaml \
  | grep openshift.io/scc

# Si el nodo no soporta unprivileged eBPF, cambiar en FlowCollector:
oc patch flowcollector cluster --type=merge \
  -p '{"spec":{"agent":{"ebpf":{"privileged":true}}}}'
```

### Loki no recibe datos

```bash
# Verificar que el LokiStack este en estado Ready
oc get lokistack -n netobserv loki \
  -o jsonpath='{.status.conditions}' | python3 -m json.tool

# Verificar conectividad desde flowlogs-pipeline a Loki
oc exec -n netobserv deployment/flowlogs-pipeline -- \
  curl -s http://loki-gateway.netobserv.svc:8080/ready
```

### La consola no muestra el plugin

```bash
# Verificar que el plugin este registrado
oc get consoleplugin netobserv-plugin -o yaml

# Forzar habilitacion manual
oc patch console cluster --type=json \
  -p '[{"op":"add","path":"/spec/plugins/-","value":"netobserv-plugin"}]'
```

### Ver el StorageClass correcto

```bash
oc get storageclass
# Buscar el que tenga (default) o el provisionado por ODF
# Actualizar spec.storageClassName en 06-lokistack.yaml
```

---

## Desinstalacion

```bash
# 1. Eliminar FlowCollector
oc delete flowcollector cluster

# 2. Eliminar LokiStack
oc delete lokistack loki -n netobserv

# 3. Desinstalar operators desde la consola:
#    Operators → Installed Operators → Desinstalar

# 4. Eliminar namespace y recursos
oc delete namespace netobserv
oc delete namespace demo-netobserv

# 5. Eliminar CRDs
oc delete crd flowcollectors.flows.netobserv.io
oc delete crd lokistacks.loki.grafana.com
```

---

## Imagenes utilizadas (todas comunitarias / sin licencia)

| Componente           | Imagen                              | Fuente             |
|----------------------|-------------------------------------|--------------------|
| MinIO                | quay.io/minio/minio:latest          | Quay.io (publico)  |
| MinIO Client (mc)    | quay.io/minio/mc:latest             | Quay.io (publico)  |
| Redis                | redis:7-alpine                      | Docker Hub         |
| httpbin (backend)    | kennethreitz/httpbin:latest         | Docker Hub         |
| nginx (frontend)     | nginxinc/nginx-unprivileged:latest  | Docker Hub         |
| curl (generador)     | curlimages/curl:latest              | Docker Hub         |
| Loki Operator        | community-operators (OperatorHub)   | Sin licencia       |
| NetObserv Operator   | community-operators (OperatorHub)   | Sin licencia       |

## Referencias

- [Documentacion oficial - Network Observability](https://docs.openshift.com/container-platform/latest/network_observability/installing-operators.html)
- [FlowCollector API Reference](https://docs.openshift.com/container-platform/latest/network_observability/flowcollector-api.html)
- [Loki Operator community](https://github.com/grafana/loki/tree/main/operator)
- [netobserv-operator community](https://github.com/netobserv/network-observability-operator)
