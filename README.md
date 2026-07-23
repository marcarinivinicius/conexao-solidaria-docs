# conexao-solidaria-docs

Documentação do hackathon "Conexão Solidária" — MVP de plataforma de
gestão de doações para ONG, com arquitetura de microsserviços, mensageria
assíncrona, Kubernetes e observabilidade (Zabbix + Grafana).

## Índice

| Documento | Conteúdo |
|---|---|
| [`architecture/diagrama.md`](architecture/diagrama.md) | Diagrama de arquitetura (Mermaid) com os 4 repositórios e o fluxo assíncrono da doação |
| [`architecture/justificativa-tecnica.pdf`](architecture/justificativa-tecnica.pdf) | Justificativa da escolha de PostgreSQL e RabbitMQ (vs. Kafka) |
| [`GUIA-MINIKUBE.md`](GUIA-MINIKUBE.md) | Passo a passo completo pra subir tudo localmente (Minikube ou Docker Compose) |
| [`video/roteiro.md`](video/roteiro.md) | Checklist de gravação do vídeo de demonstração |
| [`relatorio-entrega/template.md`](relatorio-entrega/template.md) | Template do relatório final de entrega |

## Repositórios do projeto

| Repositório | Responsabilidade |
|---|---|
| [`conexao-solidaria-shared`](https://github.com/marcarinivinicius/conexao-solidaria-shared) | Biblioteca compartilhada (cliente RabbitMQ) |
| [`conexao-solidaria-campaign-api`](https://github.com/marcarinivinicius/conexao-solidaria-campaign-api) | API: auth JWT/RBAC, campanhas, doadores, painel público, doações |
| [`conexao-solidaria-donation-worker`](https://github.com/marcarinivinicius/conexao-solidaria-donation-worker) | Worker consumidor de doações (RabbitMQ → Postgres) |
| [`conexao-solidaria-infra`](https://github.com/marcarinivinicius/conexao-solidaria-infra) | Postgres, RabbitMQ, Zabbix, Grafana + repo GitOps (manifests, ArgoCD, Argo Rollouts) |
