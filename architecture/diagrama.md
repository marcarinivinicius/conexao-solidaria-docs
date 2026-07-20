# Diagrama de Arquitetura — Conexão Solidária

```mermaid
flowchart TB
    subgraph Cliente["Cliente (Postman/Swagger)"]
        Doador["Doador"]
        Gestor["GestorONG"]
    end

    subgraph K8s["Kubernetes (namespace conexao-solidaria)"]
        subgraph API["conexao-solidaria-campaign-api"]
            Auth["Auth JWT + RBAC"]
            CampCtrl["Campanhas / Doadores / Doacoes"]
            Metrics1["/health /metrics"]
        end

        subgraph Worker["conexao-solidaria-donation-worker"]
            Consumer["DoacaoRecebidaConsumer"]
            Metrics2["/health /metrics"]
        end

        Postgres[("Postgres\ncampanhas / doadores")]
        RabbitMQ{{"RabbitMQ\nexchange: conexao-solidaria.doacoes"}}

        subgraph Obs["Observabilidade"]
            ZabbixAgent["Zabbix agent2\n(plugin Prometheus + Docker)"]
            ZabbixServer["Zabbix server + web"]
            Grafana["Grafana\n(datasource Zabbix)"]
        end
    end

    Doador -->|"JWT"| CampCtrl
    Gestor -->|"JWT"| CampCtrl
    CampCtrl -->|"CRUD"| Postgres
    CampCtrl -->|"1. publica DoacaoRecebidaEvent"| RabbitMQ
    RabbitMQ -->|"2. consome"| Consumer
    Consumer -->|"3. UPDATE ValorTotalArrecadado"| Postgres

    Metrics1 -.->|"scrape /metrics"| ZabbixAgent
    Metrics2 -.->|"scrape /metrics"| ZabbixAgent
    ZabbixAgent --> ZabbixServer
    ZabbixServer --> Grafana
```

## Componentes

| Componente | Repositório | Responsabilidade |
|---|---|---|
| `campaign-api` | [conexao-solidaria-campaign-api](https://github.com/marcarinivinicius/conexao-solidaria-campaign-api) | Auth JWT/RBAC, CRUD de campanhas, cadastro de doadores, painel público, recebe doação e publica evento |
| `donation-worker` | [conexao-solidaria-donation-worker](https://github.com/marcarinivinicius/conexao-solidaria-donation-worker) | Consome `DoacaoRecebidaEvent`, soma o valor arrecadado na campanha |
| `shared` | [conexao-solidaria-shared](https://github.com/marcarinivinicius/conexao-solidaria-shared) | Biblioteca `ConexaoSolidaria.Shared.RabbitMq` (cliente RabbitMQ compartilhado) |
| `infra` | [conexao-solidaria-infra](https://github.com/marcarinivinicius/conexao-solidaria-infra) | Postgres, RabbitMQ, Zabbix, Grafana |

## Fluxo assíncrono da doação (o ponto central do desafio)

1. Doador autenticado chama `POST /api/v1/doacoes` na `campaign-api`.
2. A API valida a campanha (não pode estar `Concluida`/`Cancelada`) e **não** grava o valor arrecadado — só publica `DoacaoRecebidaEvent` no RabbitMQ (exchange `conexao-solidaria.doacoes`, routing key `doacao-recebida`).
3. O `donation-worker` consome a fila `conexao-solidaria.doacoes.donation-worker` e executa um `UPDATE` atômico somando o valor ao `ValorTotalArrecadado` da campanha.
4. O painel público (`GET /api/v1/campanhas`) passa a refletir o novo total.

Essa separação é o que cumpre o requisito de "comunicação assíncrona e mensageria" do desafio: a escrita do valor arrecadado nunca acontece no mesmo processo/transação que recebe a doação.
