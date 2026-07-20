# Guia — Subindo o Conexão Solidária no Minikube

Guia passo a passo pra rodar toda a plataforma localmente num cluster
Kubernetes (Minikube), amarrando os 4 repositórios do projeto. Se você só
quer testar rápido sem Kubernetes, veja a [alternativa via Docker Compose](#alternativa-via-docker-compose)
no final.

## 1. Pré-requisitos

- [Docker](https://docs.docker.com/get-docker/)
- [Minikube](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- .NET SDK 9 (só se for buildar as imagens localmente em vez de baixar do registry)
- Um Personal Access Token do GitHub com escopo `read:packages`, pra
  autenticar no feed de pacotes NuGet privado do `conexao-solidaria-shared`
  (necessário só na hora de *buildar* as imagens, não para rodá-las)

## 2. Clonar os repositórios

```bash
mkdir conexao-solidaria && cd conexao-solidaria
git clone https://github.com/marcarinivinicius/conexao-solidaria-infra.git
git clone https://github.com/marcarinivinicius/conexao-solidaria-campaign-api.git
git clone https://github.com/marcarinivinicius/conexao-solidaria-donation-worker.git
```

O `conexao-solidaria-shared` **não** entra no cluster — ele só publica o
pacote NuGet consumido no build das imagens acima, via GitHub Packages.

## 3. Subir o Minikube

```bash
minikube start --cpus=4 --memory=6144 --driver=docker
minikube addons enable metrics-server
```

## 4. Buildar e carregar as imagens no cluster

O Minikube não enxerga imagens só porque existem no seu Docker local — é
preciso carregá-las explicitamente com `minikube image load` (evita
depender de um registry externo pra rodar localmente).

```bash
export GITHUB_PACKAGES_TOKEN=ghp_xxx  # seu token com read:packages

cd conexao-solidaria-campaign-api
docker build --secret id=nuget_token,env=GITHUB_PACKAGES_TOKEN -t conexao-solidaria-campaign-api:latest .
minikube image load conexao-solidaria-campaign-api:latest
cd ..

cd conexao-solidaria-donation-worker
docker build --secret id=nuget_token,env=GITHUB_PACKAGES_TOKEN -t conexao-solidaria-donation-worker:latest .
minikube image load conexao-solidaria-donation-worker:latest
cd ..
```

## 5. Aplicar os manifests, nessa ordem

A infra (Postgres, RabbitMQ, Zabbix, Grafana) precisa estar de pé e pronta
**antes** dos serviços, porque eles esperam encontrar `postgres` e
`rabbitmq` resolvíveis por nome dentro do namespace.

```bash
kubectl apply -k conexao-solidaria-infra/k8s
kubectl get pods -n conexao-solidaria -w
# espere todos os pods (postgres, rabbitmq, zabbix-*, grafana) ficarem Running antes de continuar (Ctrl+C pra sair do -w)

kubectl apply -k conexao-solidaria-campaign-api/k8s
kubectl apply -k conexao-solidaria-donation-worker/k8s

kubectl get pods -n conexao-solidaria
```

## 6. Acessando cada serviço

Via `minikube service` (abre o navegador direto) ou `kubectl port-forward`:

```bash
minikube service conexao-solidaria-campaign-api -n conexao-solidaria   # Swagger da API
minikube service rabbitmq -n conexao-solidaria                          # RabbitMQ Management UI
minikube service zabbix-web -n conexao-solidaria                        # Zabbix web
minikube service grafana -n conexao-solidaria                           # Grafana (login admin/admin)
```

Depois que o Zabbix web estiver acessível, rode o script de setup do
`conexao-solidaria-infra` pra criar os hosts/items que o dashboard do
Grafana espera:

```bash
cd conexao-solidaria-infra
ZABBIX_URL=$(minikube service zabbix-web -n conexao-solidaria --url) \
CAMPAIGN_API_METRICS_URL="http://conexao-solidaria-campaign-api.conexao-solidaria.svc.cluster.local:8080/metrics" \
./zabbix/setup.sh
```

## 7. Verificação e troubleshooting

```bash
kubectl get pods -n conexao-solidaria       # tudo deve estar Running
kubectl logs -n conexao-solidaria deploy/conexao-solidaria-campaign-api
kubectl logs -n conexao-solidaria deploy/conexao-solidaria-donation-worker
```

Problemas comuns:

| Sintoma | Causa provável | Solução |
|---|---|---|
| Pod em `Pending` | Recursos insuficientes no Minikube | Aumentar `--cpus`/`--memory` no `minikube start` |
| Pod em `ImagePullBackOff` | Esqueceu o `minikube image load` | Rodar o passo 4 de novo |
| `campaign-api`/`donation-worker` reiniciando | Postgres/RabbitMQ ainda não prontos | Confirmar que os pods da infra estão `Running` antes de aplicar os serviços |
| Painel "Containers Docker" vazio no Grafana | Node do Minikube não expõe `/var/run/docker.sock` (depende do driver) | Ver painel de HTTP requests (não depende disso) e usar `kubectl top pods` como evidência alternativa no vídeo |

## Alternativa via Docker Compose

Mais rápido pro dia a dia, sem Kubernetes:

```bash
cd conexao-solidaria-infra && docker compose up -d && cd ..
cd conexao-solidaria-campaign-api && docker compose build --secret id=nuget_token,env=GITHUB_PACKAGES_TOKEN && docker compose up -d && cd ..
cd conexao-solidaria-donation-worker && docker compose build --secret id=nuget_token,env=GITHUB_PACKAGES_TOKEN && docker compose up -d && cd ..
```

Acessos: API em `http://localhost:8081/swagger`, RabbitMQ UI em
`http://localhost:15672`, Grafana em `http://localhost:3000`, Zabbix em
`http://localhost:8080`.
