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

### 1. Pré-requisitos
* Cluster Kubernetes

### Prometheus

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

# Instala a stack completa (Prometheus + Grafana + Alertmanager)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

### Configuração do OTel (`otel-values.yaml`)
A configuração define o collector como DaemonSet para garantir um agente em cada node.

* **Segurança:** Habilitamos `privileged: true` e `runAsUser: 0`. Necessário para que o container "quebre" o isolamento e consiga ler arquivos protegidos do sistema (`/proc`, `/sys`) para métricas de hardware real.
* **Pipeline:** Configurado fluxo: `hostmetrics` (receiver) → `batch` (processor) → `prometheus` (exporter).

### Deploy via Helm

```bash
# Adicionar repositório
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts

# Criar namespace
kubectl create ns monitoring

# Instalar já usando o arquivo personalizado
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
