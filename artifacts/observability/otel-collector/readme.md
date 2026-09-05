# OTel Collector

Chart: https://artifacthub.io/packages/helm/opentelemetry-helm/opentelemetry-collector

Gateway único (`mode: deployment`, 1 réplica) que recebe traces via OTLP das
aplicações próprias do cluster, enriquece com metadados do Kubernetes
(`k8sattributes` processor) e encaminha para o Tempo.

## Gotchas do chart (confirmados renderizando)

1. O chart falha (`[ERROR] 'image.repository' must be set`) sem escolher uma
   distribuição de imagem explicitamente — usamos a distribuição
   `opentelemetry-collector-k8s` (mais leve que a `-contrib` completa, mas já
   inclui o processor `k8sattributes` de que precisamos).
2. Sem `fullnameOverride: otel-collector`, o Service nasceria com o nome
   `otel-collector-opentelemetry-collector` em vez do nome limpo usado nos
   exemplos abaixo.

O preset `presets.kubernetesAttributes.enabled: true` já injeta o processor
`k8s_attributes` no pipeline de traces **e** cria o
`ClusterRole`/`ClusterRoleBinding` necessário automaticamente — sem RBAC
manual.

## Renderizar o template

```sh
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

helm template otel-collector open-telemetry/opentelemetry-collector \
  --version 0.172.1 \
  -n observability \
  -f helm-values/values.yaml \
  > manifests/apply.yaml
```

Repita este comando a cada upgrade de versão do chart.

## Endpoint para instrumentar suas aplicações

Qualquer aplicação no cluster (em qualquer namespace — não há
`NetworkPolicy` restringindo isso) pode enviar traces OTLP para:

- **gRPC** (preferencial): `otel-collector.observability.svc.cluster.local:4317`
- **HTTP**: `otel-collector.observability.svc.cluster.local:4318`

Sem TLS (tráfego interno em texto plano — limitação deliberada, mesmo
espírito do `auth_enabled: false` do Loki) e sem autenticação.

### Node.js / TypeScript

```sh
npm install --save @opentelemetry/api @opentelemetry/auto-instrumentations-node
```

```sh
OTEL_SERVICE_NAME=my-service \
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.observability.svc.cluster.local:4317 \
OTEL_EXPORTER_OTLP_PROTOCOL=grpc \
NODE_OPTIONS="--require @opentelemetry/auto-instrumentations-node/register" \
node dist/server.js
```

Alternativa (mais controle, para adicionar processors/exporters customizados):
um arquivo `tracing.js` chamado antes do app, usando `NodeSDK` de
`@opentelemetry/sdk-node` e `.start()`.

### Java

```sh
curl -L -o opentelemetry-javaagent.jar \
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar
```

```sh
java -javaagent:./opentelemetry-javaagent.jar \
  -Dotel.service.name=my-service \
  -Dotel.exporter.otlp.endpoint=http://otel-collector.observability.svc.cluster.local:4317 \
  -Dotel.traces.exporter=otlp \
  -jar app.jar
```

Ou via variáveis de ambiente (equivalente, mais comum em Deployments):

```yaml
env:
  - name: JAVA_TOOL_OPTIONS
    value: "-javaagent:/otel/opentelemetry-javaagent.jar"
  - name: OTEL_SERVICE_NAME
    value: "my-service"
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://otel-collector.observability.svc.cluster.local:4317"
  - name: OTEL_TRACES_EXPORTER
    value: "otlp"
```

(monte o agent jar via initContainer/emptyDir na imagem, se ela não o
incluir de fábrica.)

### C# / .NET

**Opção 1 — instrumentação manual via SDK** (`Program.cs`):

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("my-service"))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter(otlp =>
        {
            otlp.Endpoint = new Uri("http://otel-collector.observability.svc.cluster.local:4317");
            otlp.Protocol = OpenTelemetry.Exporter.OtlpExportProtocol.Grpc;
        }));
```

Pacotes NuGet: `OpenTelemetry.Extensions.Hosting`,
`OpenTelemetry.Instrumentation.AspNetCore`, `OpenTelemetry.Instrumentation.Http`,
`OpenTelemetry.Exporter.OpenTelemetryProtocol`.

**Opção 2 — auto-instrumentação sem mudar código** (distribuição oficial
"OpenTelemetry .NET Automatic Instrumentation" — útil para apps que você não
quer/pode recompilar):

```sh
OTEL_DOTNET_AUTO_HOME=/otel-dotnet-auto \
OTEL_SERVICE_NAME=my-service \
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.observability.svc.cluster.local:4317 \
OTEL_DOTNET_AUTO_TRACES_ENABLED=true \
dotnet myapp.dll
```

A distribuição inclui um launcher (`instrument.sh`) que configura as demais
variáveis `OTEL_DOTNET_AUTO_*`/`CORECLR_*` necessárias para o profiler do
CLR — prefira usá-lo em vez de montar essas variáveis manualmente.

## O que não foi configurado / requer ação manual

- **Sem TLS** entre apps e o collector — tráfego interno em texto plano,
  simplificação deliberada para um cluster home-lab.
- **Sem política de sampling** — os SDKs acima usam 100% head sampling por
  padrão. Revisar se o volume de traces crescer.
- **Sem autenticação no OTLP** — qualquer pod no cluster pode enviar traces.
