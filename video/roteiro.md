# Roteiro do Vídeo de Demonstração (máx. 15 minutos)

Checklist direto do que o enunciado do hackathon exige mostrar — grave
nessa ordem, sem ler código linha a linha.

**Antes de gravar**: rode
[`conexao-solidaria-infra/scripts/seed-demo-data.sh`](https://github.com/marcarinivinicius/conexao-solidaria-infra/blob/main/scripts/seed-demo-data.sh)
por alguns minutos (`DURATION_SECONDS=180` ou mais) — gera campanhas,
doadores e um fluxo real de doações via API, deixando os painéis do
Grafana e o RabbitMQ com dados de verdade em vez de vazios/planos.

- [ ] **1. Diagrama de arquitetura** (2 min) — abrir
      [`architecture/diagrama.md`](../architecture/diagrama.md) e explicar
      os 4 repositórios, o fluxo assíncrono da doação e onde entra o
      Zabbix/Grafana.

- [ ] **2. Pipeline de CI** (2 min) — abrir a aba Actions de um dos repos
      (`campaign-api` ou `donation-worker`) rodando: build, testes e
      geração da imagem Docker com sucesso.

- [ ] **3. Kubernetes rodando** (2 min)
      - Terminal: `kubectl get pods -n conexao-solidaria` com tudo `Running`.
      - Dashboard do Grafana com dados reais (painel de HTTP requests da
        `campaign-api` e/ou CPU/memória dos containers via Zabbix).

- [ ] **4. Funcionamento ponta a ponta** (7-8 min)
      1. **Autenticação**: login como `GestorONG` via Swagger/Postman,
         copiar o token JWT.
      2. **Criar campanha**: `POST /api/v1/campanhas` autenticado como
         `GestorONG`.
      3. **Cadastro + login de doador**: `POST /api/v1/auth/register/doador`
         e depois `POST /api/v1/auth/login`.
      4. **Simular doação**: `POST /api/v1/doacoes` autenticado como
         `Doador` — mostrar o payload sendo enviado.
      5. **RabbitMQ Management UI**: abrir `http://localhost:15672` (ou a
         URL do NodePort no minikube) e mostrar a mensagem passando pela
         fila `conexao-solidaria.doacoes.donation-worker`.
      6. **Confirmar atualização**: `GET /api/v1/campanhas` (painel
         público) mostrando o `ValorTotalArrecadado` já somado pelo
         worker.

- [ ] **5. Fechamento** (1 min) — recapitular os requisitos técnicos
      atendidos: microsserviços, mensageria assíncrona, Kubernetes,
      observabilidade, CI/CD.

## Onde gravar cada coisa

| Item | Onde |
|---|---|
| Diagrama | [`architecture/diagrama.md`](../architecture/diagrama.md) (renderiza no GitHub) |
| Guia de subida completa | [`GUIA-MINIKUBE.md`](../GUIA-MINIKUBE.md) |
| Swagger da API | `http://localhost:<porta>/swagger` |
| RabbitMQ Management | `http://localhost:15672` (compose) ou NodePort 30672 (minikube) |
| Grafana | `http://localhost:3000` (compose) ou NodePort 30300 (minikube) |
