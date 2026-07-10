## Estrutura

- `apps/` — manifests Kubernetes dos microsserviços e da stack de monitoramento
  - `auth-service/`, `flag-service/`, `targeting-service/`, `evaluation-service/`, `analytics-service/` — Deployments, Services e ConfigMaps de cada microsserviço
  - `monitoring/` — Helm values, dashboards, datasources e configuração de alertas do Grafana
- `argocd/applications/` — manifests de Application do ArgoCD, um por componente sincronizado no cluster (5 microsserviços + kube-prometheus-stack + loki-stack + otel-collector + monitoring-extras)
- `.github/workflows/` — automações do repositório, incluindo o workflow de self-healing (`self-healing.yml`)

## Como o ArgoCD usa este repositório

Cada arquivo em `argocd/applications/` é uma `Application` do ArgoCD apontando
para uma pasta dentro de `apps/`. O ArgoCD observa este repositório continuamente
e aplica automaticamente qualquer mudança commitada na branch `main` (modelo
GitOps): não é necessário rodar `kubectl apply` manual para os componentes
gerenciados por ele.

Para sincronizar manualmente (ex.: após provisionar um cluster novo):

```bash
kubectl apply -f argocd/applications/
```

## Segredos

Nenhum valor sensível é commitado neste repositório em texto plano. Os arquivos
que precisam de credenciais reais existem como `.example`:

| Arquivo real (não commitado) | Exemplo versionado | Contém |
|---|---|---|
| `apps/monitoring/grafana-alerting-secret.yaml` | `grafana-alerting-secret.yaml.example` | Integration Key do PagerDuty e webhook URL do Slack |
| `apps/monitoring/otel-collector-secrets.yaml` | `otel-collector-secrets.yaml.example` | Chave de ingestão do New Relic |

Fluxo de uso: copiar o `.example`, preencher os valores reais localmente e aplicar
com `kubectl apply -f <arquivo> -n <namespace>` no dia da gravação/deploy. O
arquivo preenchido nunca deve ser adicionado ao Git.

## Self-Healing

O workflow `.github/workflows/self-healing.yml` é disparado manualmente
(`workflow_dispatch`) pelo GitHub Actions. Ele:

1. Autentica na AWS usando credenciais temporárias do AWS Academy Learner Lab (via GitHub Secrets)
2. Atualiza o kubeconfig do cluster EKS (`aws eks update-kubeconfig`)
3. Executa `kubectl rollout restart` no deployment informado
4. Verifica o status do rollout até a conclusão

**GitHub Secrets necessários** (Settings → Secrets and variables → Actions):

| Secret | Descrição |
|---|---|
| `AWS_ACCESS_KEY_ID` | Credencial temporária da sessão do AWS Academy |
| `AWS_SECRET_ACCESS_KEY` | Credencial temporária da sessão do AWS Academy |
| `AWS_SESSION_TOKEN` | Credencial temporária da sessão do AWS Academy |
| `AWS_REGION` | Região do cluster (ex.: `us-east-1`) |
| `EKS_CLUSTER_NAME` | Nome do cluster EKS criado pelo Terraform |

Como as credenciais da AWS Academy expiram a cada sessão do Lab, esses 3 primeiros
secrets precisam ser atualizados toda vez que uma nova sessão for iniciada.

## Monitoramento

- **Grafana** — dashboard customizado com uso de recursos do cluster, taxa de requisições dos microsserviços e painel de logs em tempo real
- **Prometheus** — armazenamento e consulta de métricas de infraestrutura
- **Loki** — centralização e indexação de logs dos contêineres
- **OpenTelemetry Collector** — peça central que recebe telemetria dos microsserviços e roteia métricas para o Prometheus, logs para o Loki e traces para o New Relic (APM)
- **Alertas** — regra de erro 5xx do `auth-service` (>5% por 2 min), notificando PagerDuty (incidente) e Slack (ChatOps)

## Repositórios relacionados

- [`togglemaster-infra`](https://github.com/leonardofmoraes/togglemaster-infra) — Terraform: VPC, EKS, RDS, Redis
- [`togglemaster-local-dev`](https://github.com/leonardofmoraes/togglemaster-local-dev) — ambiente de desenvolvimento local via docker-compose
