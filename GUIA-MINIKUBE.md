# Guia — Subindo o Conexão Solidária no Minikube

Guia passo a passo pra rodar toda a plataforma localmente num cluster
Kubernetes (Minikube) usando o fluxo **GitOps real**: ArgoCD sincroniza os
manifests do [`conexao-solidaria-infra`](https://github.com/marcarinivinicius/conexao-solidaria-infra)
e o Argo Rollouts promove cada deploy em **canary**. Se você só quer
testar rápido sem Kubernetes/ArgoCD, veja a
[alternativa via Docker Compose](#alternativa-via-docker-compose) no final.

## 1. Pré-requisitos

- [Docker](https://docs.docker.com/get-docker/)
- [Minikube](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/) (CLI standalone — `kubectl kustomize` não tem o subcomando `edit` que usamos pra trocar a tag da imagem)
- [Argo Rollouts kubectl plugin](https://argo-rollouts.readthedocs.io/en/stable/installation/#kubectl-plugin-installation) (opcional, mas facilita acompanhar o canary)
- .NET SDK 9 (só se for buildar as imagens localmente em vez de puxar do GHCR)
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
minikube addons enable ingress   # necessário pro trafficRouting (nginx) do canary da campaign-api
```

## 4. Subir a infra compartilhada

Postgres, RabbitMQ, Zabbix e Grafana **não** passam pelo ArgoCD (são infra
de base, não "aplicações" desse projeto) — sobem direto via kustomize:

```bash
cd conexao-solidaria-infra
kubectl apply -k infra/
kubectl get pods -n conexao-solidaria -w
# espere postgres, rabbitmq, zabbix-*, grafana ficarem Running (Ctrl+C pra sair do -w)
```

## 5. Instalar ArgoCD e Argo Rollouts

```bash
# ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd wait --for=condition=available --timeout=300s deployment/argocd-server

# Argo Rollouts
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
kubectl -n argo-rollouts wait --for=condition=available --timeout=300s deployment/argo-rollouts
```

Detalhes e troubleshooting de cada um em
[`conexao-solidaria-infra/infra/argocd/README.md`](https://github.com/marcarinivinicius/conexao-solidaria-infra/blob/main/infra/argocd/README.md)
e [`infra/argo-rollouts/README.md`](https://github.com/marcarinivinicius/conexao-solidaria-infra/blob/main/infra/argo-rollouts/README.md).

## 6. Registrar os serviços no ArgoCD

Isso aplica só os manifests `Application` — quem materializa o `Rollout`,
`Service` e `Ingress` de cada serviço é o próprio ArgoCD, puxando do repo:

```bash
kubectl apply -f conexao-solidaria-infra/cluster/apps/services/campaign-api-app.yaml
kubectl apply -f conexao-solidaria-infra/cluster/apps/services/donation-worker-app.yaml

kubectl get applications -n argocd
kubectl get pods -n conexao-solidaria -w
```

A primeira sincronização vai tentar puxar
`ghcr.io/marcarinivinicius/conexao-solidaria-campaign-api:latest` (e o
worker equivalente). Se o CI de cada repo já rodou pelo menos uma vez com
push na `main` (precisa do secret `INFRA_REPO_TOKEN` configurado — ver
README de cada serviço), essa tag já existe e os pods sobem normalmente.
Se não, use o atalho do passo 7.

## 7. Testando localmente sem esperar o CI (atalho)

Pra iterar rápido sem depender do pipeline completo (build → push no GHCR
→ PR → merge), builda a imagem local, carrega no Minikube e edita a tag
manualmente:

```bash
export GITHUB_PACKAGES_TOKEN=ghp_xxx  # seu token com read:packages

cd conexao-solidaria-campaign-api
docker build --secret id=nuget_token,env=GITHUB_PACKAGES_TOKEN \
  -t ghcr.io/marcarinivinicius/conexao-solidaria-campaign-api:dev .
minikube image load ghcr.io/marcarinivinicius/conexao-solidaria-campaign-api:dev
cd ..

cd conexao-solidaria-infra/cluster/apps/services/campaign-api
kustomize edit set image ghcr.io/marcarinivinicius/conexao-solidaria-campaign-api=ghcr.io/marcarinivinicius/conexao-solidaria-campaign-api:dev
git commit -am "deploy(campaign-api): dev local" && git push origin main
```

O ArgoCD (com `syncPolicy.automated`) detecta o push na `main` e
sincroniza sozinho — não precisa `kubectl apply` manual depois disso.
Repita o mesmo pro `donation-worker`.

## 8. Acompanhando o rollout canary

```bash
kubectl argo rollouts get rollout conexao-solidaria-campaign-api -n conexao-solidaria --watch
```

Você vai ver os degraus `setWeight: 20` → pausa de 30s → `60` → pausa →
`100`. Pra promover manualmente (pular a pausa) ou abortar:

```bash
kubectl argo rollouts promote conexao-solidaria-campaign-api -n conexao-solidaria
kubectl argo rollouts abort conexao-solidaria-campaign-api -n conexao-solidaria
```

O `donation-worker` segue o mesmo padrão, mas sem `trafficRouting` (canary
por proporção de réplicas, já que é só um consumidor de fila, sem
Ingress).

## 9. Acessando cada serviço

Cada serviço tem um `Ingress` próprio (`nginx`, mesmo controller que já
faz o roteamento canary da `campaign-api`) com um hostname fixo — é o
caminho recomendado, porque com **um único processo** (`minikube tunnel`)
você acessa tudo, em vez de um `port-forward`/`minikube service` por
serviço. E como o Ingress aponta pro Service (não pro pod), sobrevive a
qualquer canary/restart de pod por baixo.

1. Adicione os hostnames no hosts file (uma vez só) apontando pra
   `127.0.0.1` — o `minikube tunnel` faz o roteamento local:

   ```
   # C:\Windows\System32\drivers\etc\hosts (como Administrador) ou /etc/hosts no Linux/Mac
   127.0.0.1  campaign-api.conexao-solidaria.local
   127.0.0.1  grafana.conexao-solidaria.local
   127.0.0.1  zabbix.conexao-solidaria.local
   127.0.0.1  rabbitmq.conexao-solidaria.local
   127.0.0.1  argocd.conexao-solidaria.local
   ```

2. Deixe **um** terminal aberto com o tunnel (pede senha de admin/sudo —
   é ele que faz a ponte do host pro container do node no driver `docker`):

   ```bash
   minikube tunnel
   ```

3. Acesse direto, sem porta:

   ```
   http://campaign-api.conexao-solidaria.local/swagger
   http://grafana.conexao-solidaria.local           (login admin/admin)
   http://zabbix.conexao-solidaria.local
   http://rabbitmq.conexao-solidaria.local
   https://argocd.conexao-solidaria.local
   ```

Se `minikube tunnel` não for uma opção (ex.: sem permissão de
admin/sudo), cada Service continua exposto como `NodePort` também
(`kubectl get svc -n conexao-solidaria` pra ver as portas) — `minikube
service <nome> -n <namespace> --url` ou `kubectl port-forward` funcionam
como alternativa, só que aí volta a precisar de um processo por serviço.

Depois que o Zabbix web estiver acessível, rode o script de setup do
`conexao-solidaria-infra` pra criar os hosts/items que o dashboard do
Grafana espera:

```bash
cd conexao-solidaria-infra
ZABBIX_URL=$(minikube service zabbix-web -n conexao-solidaria --url) \
CAMPAIGN_API_METRICS_URL="http://conexao-solidaria-campaign-api-svc-stable.conexao-solidaria.svc.cluster.local:8080/metrics" \
./zabbix/setup.sh
```

## 10. Verificação e troubleshooting

```bash
kubectl get pods -n conexao-solidaria                       # tudo deve estar Running
kubectl get applications -n argocd                           # Synced / Healthy
kubectl argo rollouts get rollout conexao-solidaria-campaign-api -n conexao-solidaria
kubectl logs -n conexao-solidaria deploy/conexao-solidaria-campaign-api
```

Problemas comuns:

| Sintoma | Causa provável | Solução |
|---|---|---|
| Pod em `Pending` | Recursos insuficientes no Minikube | Aumentar `--cpus`/`--memory` no `minikube start` |
| Pod em `ImagePullBackOff` | Tag da imagem não existe no GHCR ainda, ou esqueceu o `minikube image load` no atalho local | Ver passo 7, ou conferir se o CI do serviço já rodou com push na `main` |
| Application do ArgoCD em `OutOfSync`/`Unknown` | ArgoCD ainda não conseguiu ler o repo, ou path errado | `kubectl describe application <nome> -n argocd` pra ver o erro exato |
| `Rollout` travado num `setWeight` | Está esperando promoção manual (comportamento esperado — não tem `AnalysisTemplate` automatizada) | `kubectl argo rollouts promote <app> -n conexao-solidaria` |
| `campaign-api`/`donation-worker` reiniciando | Postgres/RabbitMQ ainda não prontos | Confirmar que os pods da infra estão `Running` antes de registrar os `Application` no ArgoCD |
| Painel "Containers Docker" vazio no Grafana | Node do Minikube não expõe `/var/run/docker.sock` (depende do driver) | Ver painel de HTTP requests (não depende disso) e usar `kubectl top pods` como evidência alternativa no vídeo |
| Swagger/Grafana/RabbitMQ/ArgoCD não carregam | Se estiver usando `kubectl port-forward` em vez de `minikube service` (passo 9), ele fica preso ao processo do pod específico — qualquer substituição de pod (canary, `minikube stop`/`start`, restart) mata o forward em silêncio | Usar `minikube service <nome> -n <namespace> --url` (NodePort fixo, sobrevive à troca de pod). Se insistir em `port-forward`, rodar de novo o comando |

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
