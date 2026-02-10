[PT-BR, clique aqui](#pt-br)

# Monitoring of Kubernetes infra with OpenTelemetry and Grafana

This project implements an observability pipeline to monitor a Kubernetes cluster nodes' health and resources (CPU, memory, disk, network).

Using the OpenTelemetry Collector as a DaemonSet, it collects metrics of infrastructure directly from the host (node) and exports to Prometheus. Visualization handled in Grafana.

---

## Snapshots

### 1. Prometheus
Prometheus automatically discovers the OpenTelemetry pod through ServiceMonitor, removing the necessity for static IPs.
<img width="1496" height="441" alt="otel1" src="https://github.com/user-attachments/assets/565476c4-c2bc-49eb-9a06-7cc22e731805" />

### 2. Grafana dashboard
Visualization is handled with PromQL to display the node's real memory usage, converting raw bytes to GiB.
<img width="790" height="568" alt="otel2" src="https://github.com/user-attachments/assets/1ec37c37-b19a-406e-bb1b-63fe7212f051" />

---

## Motivation for implementation

1. When adopting the OpenTelemetry pattern, data collection is decoupled from storage. If a company decides to migrate from Prometheus to Datadog, Dynatrace, or another, just updating an exporter configuration is enough. There'll be no need to update the application's code.
2. The monitoring runs on the host (node) level, not isolated in the container. It prevents an overloaded node from disturbing the performance of all critical pods in it.
3. Using batch processors in the OTel Collector reduces network overhead and the Prometheus load, grouping metrics before sending.

---

## Stack

* **Kubernetes:** Container orchestration.
* **OpenTelemetry Collector:** Collection agent and telemetry processing.
* **Prometheus Operator:** Storage of metrics and dynamic service discovery.
* **Grafana:** Visualization and dashboards.
* **Helm:** Package management and deploy.

---

## Reproducing it

```bash
# 1. Add repositories (Prometheus and OpenTelemetry)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

# 2. Install Prometheus stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

### OTel configuration (`otel-values.yaml`)
It defines the collector as a DaemonSet to ensure one agent on each node.

* **Security:** Enabled `privileged: true` and `runAsUser: 0`. So the container "breaks" its isolation and can read system's protected files (`/proc`, `/sys`) for real hardware metrics.
* **Pipeline:** Flow configured: `hostmetrics` (receiver) → `batch` (processor) → `prometheus` (exporter).

### OpenTelemetry deploy

```bash
# Install the Collector using a custom values file
helm install my-otel-collector open-telemetry/opentelemetry-collector \
  -f otel-values.yaml \
  -n monitoring
```

### Service discovery and exposure
Exposes the service and allows Prometheus Operator to find it:

```bash
# Creates the service
kubectl apply -f otel-service.yaml

# Creates ServiceMonitor (discovery rules for Prometheus)
kubectl apply -f otel-service-monitor.yaml
```

## Challenges and learning

* **Dynamic Service discovery:** Prometheus initially couldn't find the target. The solution was to implement a `ServiceMonitor` configured with the correct labels (`release: monitoring`), allowing the Prometheus Operator to find OTel pods automatically.

* **Security vs. observability:** It was necessary to configure `securityContext` as privileged. Although it is a risk in common applications, for host agents it is a mandatory exception, because isolated containers do not have read permissions for the physical machine's disk and kernel I/O statistics.

* **Grafana data treatment:** Metrics were coming in bytes and overlapping. I used PromQL (`sum by (state)`) and the unit formatting to transform raw data.

## Project's structure

* `otel-values.yaml`: Configuration of the DaemonSet, receivers, processors and exporters.

* `otel-service.yaml`: Defines the service of K8s to expose the metrics port.

* `otel-service-monitor.yaml`: "Teaches" Prometheus how to monitor the specific service.

---
<div id="pt-br"></div>

# Monitoramento de infra Kubernetes com OpenTelemetry e Grafana

Este projeto implementa um pipeline de observabilidade para monitorar a saúde e os recursos (CPU, memória, disco, rede) dos nodes de um cluster Kubernetes.

Utilizando o OpenTelemetry Collector como DaemonSet, coleta-se métricas de infraestrutura diretamente do host (node) e exporta-se para o Prometheus, com visualização final tratada no Grafana.

---

## Snapshots

### 1. Prometheus
O Prometheus descobre automaticamente os pods do OpenTelemetry através do ServiceMonitor, sem necessidade de IPs estáticos.
<img width="1496" height="441" alt="otel1" src="https://github.com/user-attachments/assets/565476c4-c2bc-49eb-9a06-7cc22e731805" />

### 2. Grafana dashboard
Visualização tratada com PromQL para exibir o uso real de memória do node, convertendo bytes brutos para GiB.
<img width="790" height="568" alt="otel2" src="https://github.com/user-attachments/assets/1ec37c37-b19a-406e-bb1b-63fe7212f051" />

---

## Motivações para implementar

1. Ao adotar o padrão OpenTelemetry, a coleta de dados é desacoplada do armazenamento. Se a empresa decidir migrar do Prometheus para o Datadog, Dynatrace, entre outros, basta alterar uma configuração no exporter, sem fazer alterações no código das aplicações.
2. O monitoramento roda no nível do host (node) e não apenas isolado no container. Isso previne que um node sobrecarregado degrade a performance de todos os pods críticos nele.
3. A utilização de processadores em batch no OTel Collector reduz o overhead de rede e a carga do no Prometheus, agrupando métricas antes do envio.

---

## Ferramentas

* **Kubernetes:** Orquestração de containers.
* **OpenTelemetry Collector :** Agente de coleta e processamento de telemetria.
* **Prometheus Operator:** Armazenamento de séries temporais e service discovery dinâmico.
* **Grafana:** Visualização e dashboards.
* **Helm:** Gerenciamento de pacotes e deploy.

---

## Como reproduzir

```bash
# 1. Adicionar repositórios (Prometheus e OpenTelemetry)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

# 2. Instalar a stack Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

### Configuração do OTel (`otel-values.yaml`)
A configuração define o collector como DaemonSet para garantir um agente em cada node.

* **Segurança:** Habilitamos `privileged: true` e `runAsUser: 0`. Necessário para que o container "quebre" o isolamento e consiga ler arquivos protegidos do sistema (`/proc`, `/sys`) para métricas de hardware real.
* **Pipeline:** Configurado fluxo: `hostmetrics` (receiver) → `batch` (processor) → `prometheus` (exporter).

### Deploy do OpenTelemetry

```bash
# Instalar o Collector usando o arquivo de valores personalizado
helm install my-otel-collector open-telemetry/opentelemetry-collector \
  -f otel-values.yaml \
  -n monitoring
```

### Service discovery e exposição
Expor o serviço e permitir que o Prometheus Operator o encontre:

```bash
# Cria o service
kubectl apply -f otel-service.yaml

# Cria o ServiceMonitor (regras de descoberta para o Prometheus)
kubectl apply -f otel-service-monitor.yaml
```

## Desafios e aprendizados

* **Service discovery dinâmico:** O Prometheus inicialmente não encontrava o alvo. A solução foi implementar um `ServiceMonitor` configurado com as labels corretas (`release: monitoring`), permitindo que o Prometheus Operator descobrisse os Pods do OTel automaticamente.

* **Segurança vs. observabilidade:** Foi necessário configurar o `securityContext` como privilegiado. Embora seja um risco em aplicações comuns, para agentes de host isso é uma exceção mandatória, pois containers isolados não têm permissão de leitura sobre estatísticas de I/O de disco e kernel da máquina física.

* **Tratamento de dados no Grafana:** As métricas chegavam cruas (em bytes) e sobrepostas. Utilizei PromQL (`sum by (state)`) e formatação de unidades para transformar dados brutos.

## Estrutura do projeto

* `otel-values.yaml`: Configuração principal do DaemonSet, receivers, processors e exporters.

* `otel-service.yaml`: Define o service do K8s para expor a porta de métricas.

* `otel-service-monitor.yaml`: "Ensina" o Prometheus a monitorar o serviço específico.
