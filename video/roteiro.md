# Roteiro do Vídeo de Demonstração (máx. 15 minutos)

Roteiro com **fala sugerida** (não decore, use como guia) e comandos exatos
pra cada bloco. Grave nessa ordem. Onde diz "diga:", é o texto pra narrar
enquanto a tela mostra o que está descrito acima.

**Antes de gravar**: rode
[`conexao-solidaria-infra/scripts/seed-demo-data.sh`](https://github.com/marcarinivinicius/conexao-solidaria-infra/blob/main/scripts/seed-demo-data.sh)
por alguns minutos (`DURATION_SECONDS=180` ou mais) — gera campanhas,
doadores e um fluxo real de doações via API, deixando os painéis do
Grafana e o RabbitMQ com dados de verdade em vez de vazios/planos. Deixe
rodando em segundo plano durante a gravação dos blocos 1-3.

Também abra **antes** de começar: Swagger da `campaign-api`, RabbitMQ
Management UI, Grafana, um terminal com `kubectl` configurado apontando pro
Minikube, e os 5 terminais/abas do VS Code abertos nos 5 repos — evita
alternar contexto gravando.

---

## 1. Abertura + diagrama de arquitetura (1-2 min)

Abrir [`architecture/diagrama.md`](../architecture/diagrama.md).

**Diga:**
> "Esse é o Conexão Solidária, um MVP de plataforma de doações pra ONGs,
> feito com arquitetura de microsserviços. São 5 repositórios: uma
> biblioteca compartilhada de mensageria, a API principal de campanhas e
> doações, um worker assíncrono, a infraestrutura como código e a
> documentação. O fluxo principal é: o doador registra uma doação na API,
> a API **não** atualiza o valor arrecadado direto no banco — ela publica
> um evento no RabbitMQ. Quem processa esse evento e soma o valor é um
> segundo serviço, o donation-worker, rodando de forma totalmente
> desacoplada. Isso é o requisito de comunicação assíncrona entre
> microsserviços atendido de verdade, não simulado."

Apontar rapidamente pro Postgres, RabbitMQ e Zabbix/Grafana no diagrama.

---

## 2. Pipeline de CI/CD (2 min)

Abrir a aba **Actions** do repo `conexao-solidaria-campaign-api` no
GitHub, mostrando uma execução recente e verde.

**Diga:**
> "O pipeline de CI roda em três níveis diferentes de gatilho. Em todo
> Pull Request, ele builda e roda os testes automatizados, sem gerar
> imagem — é só validação antes do merge. Em todo push na `main`, além de
> build e teste, ele já builda e publica uma imagem Docker no GitHub
> Container Registry, taggeada com o hash do commit — isso satisfaz o
> requisito de gerar imagem a cada push na main, e dá rastreabilidade
> total: qualquer commit da main tem uma imagem correspondente publicada.
> Só quando eu crio uma tag de versão, tipo `v1.0.0`, é que o pipeline
> dispara o terceiro estágio: builda a imagem com a tag de versão — nunca
> `latest` — e abre um Pull Request automático no repositório de infra,
> trocando a tag da imagem no manifesto do Kubernetes."

Clicar num job `publish-image` recente pra mostrar a imagem sendo
publicada no GHCR. Se tiver uma tag de release recente, mostrar também o
PR aberto automaticamente no `conexao-solidaria-infra` (mergeado à mão,
deploy manual e deliberado — não sobe sozinho no cluster).

**Diga (fechando o bloco):**
> "Esse desenho separa claramente 'todo commit gera evidência' de 'só
> release consciente promove pro cluster' — é o padrão GitOps: o Git é a
> fonte da verdade de o que está rodando, nunca um `kubectl apply` manual."

---

## 3. Testes automatizados (3-4 min) — bloco central pedido

Esse é o bloco que mais precisa de contexto falado, porque os testes por
si só não contam a história. Cada projeto de teste protege uma decisão de
arquitetura específica — explique **qual risco** cada um elimina, não só
"tem teste".

### 3.1 `campaign-api` — Domain.Tests

Terminal, dentro de `conexao-solidaria-campaign-api`:
```bash
dotnet test tests/ConexaoSolidaria.CampaignApi.Domain.Tests
```

**Diga (antes de rodar, explicando o que vem):**
> "Começando pelos testes de domínio, que validam as regras de negócio
> puras, sem banco, sem HTTP. Aqui garantimos duas regras que vêm direto
> do enunciado do hackathon: uma campanha não pode ser criada com meta
> financeira menor ou igual a zero, e não pode ter data de término no
> passado. E do lado do doador, validamos o CPF com o algoritmo de
> checksum real, não só formato — um CPF com dígito verificador errado é
> rejeitado no domínio, antes de qualquer persistência."

Rodar o comando, deixar a saída verde aparecer na tela.

### 3.2 `campaign-api` — Application.Tests (o mais rico)

```bash
dotnet test tests/ConexaoSolidaria.CampaignApi.Application.Tests
```

**Diga (antes de rodar):**
> "Esse aqui é o projeto que testa os casos de uso de autenticação, com
> fakes em memória no lugar do banco real — sem biblioteca de mock, só
> implementações simples das interfaces, o suficiente pro que
> precisamos. Ele cobre o mecanismo de refresh token com rotação, que é
> o ponto de segurança mais sensível da API. Os casos cobertos são:
> primeiro, um refresh válido gera um par novo de tokens e marca o token
> antigo como revogado, apontando pro substituto. Segundo, um token
> expirado é rejeitado. Terceiro — e esse é o mais importante — se
> alguém tentar reusar um refresh token que já foi trocado por um novo,
> a API detecta isso como sinal de roubo ou replay de sessão e revoga
> **todos** os tokens ativos daquele usuário na hora, não só o que foi
> reusado. Isso segue a recomendação da RFC 6819 de detecção de reuso em
> refresh token rotation. Também testamos o logout, que precisa ser
> idempotente — chamar duas vezes não pode dar erro — e o caso do
> SuperAdmin, que não tem uma linha de usuário tradicional no banco, só
> existe via claim, e mesmo assim o fluxo de refresh funciona igual."

Rodar o comando, mostrar a lista de testes passando (a saída nomeia cada
cenário — deixe aparecer pelo menos 2-3 segundos parado na tela antes de
cortar).

### 3.3 `donation-worker` — testes do consumer

No repo `conexao-solidaria-donation-worker`:
```bash
dotnet test
```

**Diga (antes de rodar):**
> "Do lado do worker, os testes cobrem o consumidor da fila de doações.
> O cenário mais importante aqui é idempotência contra reentrega do
> RabbitMQ: o broker garante entrega **pelo menos uma vez**, o que
> significa que a mesma mensagem pode chegar duplicada em cenários de
> falha de rede ou reinício do worker. Testamos que, se a mesma doação
> chegar duas vezes, o worker detecta que ela já foi processada e não
> soma o valor de novo — sem isso, uma simples reconexão do RabbitMQ
> poderia inflar artificialmente o total arrecadado de uma campanha, o
> que numa plataforma de doação de dinheiro real seria um bug crítico.
> Também cobrimos: campanha que não existe mais não trava a fila,
> mensagem malformada é descartada, e valor zero ou negativo é
> ignorado."

Rodar o comando, mostrar verde.

**Diga (fechando o bloco de testes):**
> "Resumindo os três: domínio garante que a regra de negócio nunca é
> violada mesmo sem chegar no banco; a aplicação garante que um ataque
> de replay de sessão é detectado e contido automaticamente; e o worker
> garante que a mensageria assíncrona é segura contra reentrega, que é
> uma característica normal — não um bug — do RabbitMQ."

---

## 4. Kubernetes + observabilidade rodando (2 min)

Terminal:
```bash
kubectl get pods -n conexao-solidaria
kubectl argo rollouts get rollout campaign-api -n conexao-solidaria
```

**Diga:**
> "No cluster, os dois serviços sobem como `Rollout` do Argo Rollouts em
> vez de `Deployment` puro — isso dá deploy canário automático: toda
> imagem nova sobe primeiro pra 20% das réplicas, pausa, sobe pra 60%,
> pausa de novo, e só então vai pra 100%. O ArgoCD sincroniza o cluster a
> partir do repositório de infra automaticamente, sem `kubectl apply`
> manual."

Trocar pro Grafana, mostrar os painéis com dados reais do
`seed-demo-data.sh` rodando.

**Diga:**
> "A observabilidade usa Zabbix coletando as métricas em formato
> Prometheus expostas em `/metrics` de cada serviço, mais os dados da API
> de management do RabbitMQ, e o Grafana como camada de visualização.
> Aqui dá pra ver requisições HTTP por endpoint, o total de doações
> processadas e o tamanho da fila em tempo real."

Se der, rodar `kubectl top pods -n conexao-solidaria` como evidência
adicional de consumo de recursos (CPU/memória não vêm do Zabbix nesse
setup, é limitação documentada no README do infra).

---

## 5. Funcionamento ponta a ponta via Swagger (4-5 min)

Usar o Swagger (`/swagger`) ou a collection do Postman — o que ficar mais
fluido pra narrar ao vivo.

### 5.1 Cadastro e autenticação

`POST /api/v1/auth/register` com um doador novo.

**Diga:**
> "Cadastro de doador é público e já retorna o token de acesso e o
> refresh token direto — não precisa fazer uma segunda chamada de login.
> O token de acesso é um JWT de vida curta, 15 minutos, o refresh token é
> um segredo opaco de 7 dias que fica guardado com hash no banco, nunca
> em texto puro."

Copiar o token, autorizar no Swagger.

### 5.2 Rate limiting (rápido, opcional se sobrar tempo)

**Diga (se for mostrar):**
> "Login, registro e refresh têm limite de 5 requisições por minuto por
> IP — na sexta tentativa seguida, a API responde 429 em vez de deixar
> tentar força bruta de senha indefinidamente."

### 5.3 Criar campanha (GestorONG)

`POST /api/v1/campanhas` autenticado como `GestorONG`.

**Diga:**
> "Só quem tem a role GestorONG pode criar campanha. O corpo é validado
> em duas camadas: primeiro o formato dos campos, via FluentValidation,
> antes mesmo de chegar na lógica de negócio; depois a regra de negócio
> em si, no domínio — meta financeira maior que zero e data de término
> no futuro."

### 5.4 Registrar doação com Idempotency-Key

`POST /api/v1/doacoes` autenticado como `Doador`, com o header
`Idempotency-Key: <um GUID>`.

**Diga:**
> "Ao registrar a doação eu mando um header `Idempotency-Key` gerado pelo
> cliente. Se essa mesma chamada falhar por timeout de rede e o cliente
> reenviar automaticamente com a mesma chave, a API detecta que já
> processou essa doação e não publica o evento duas vezes — isso é
> idempotência do lado do produtor, complementando a idempotência que já
> vimos do lado do consumidor no worker. Doação nunca é contada em
> dobro, nem por retry do cliente, nem por reentrega do broker."

### 5.5 RabbitMQ Management UI

Abrir `http://localhost:15672` (ou NodePort no Minikube), mostrar a
mensagem passando pela fila
`conexao-solidaria.doacoes.donation-worker`.

**Diga:**
> "Aqui dá pra ver o evento `DoacaoRecebidaEvent` passando pela fila em
> tempo real, sendo consumido pelo worker."

### 5.6 Confirmar atualização assíncrona

`GET /api/v1/campanhas` (painel público, sem autenticação).

**Diga:**
> "E aqui, no painel público — que qualquer visitante pode acessar sem
> login — o `valorTotalArrecadado` já está atualizado. Repare que
> ninguém chamou nenhum endpoint pra 'atualizar o total': foi o worker,
> processando a fila de forma assíncrona, que fez essa soma."

---

## 6. Fechamento (1 min)

**Diga:**
> "Resumindo os requisitos técnicos atendidos: autenticação JWT com
> controle de acesso por role, dois microsserviços se comunicando de
> forma assíncrona via RabbitMQ, deploy em Kubernetes com rollout
> canário via ArgoCD e Argo Rollouts, observabilidade com Zabbix e
> Grafana mostrando métricas reais, e um pipeline de CI que gera imagem
> Docker publicada a cada push na main. Além do mínimo pedido, o projeto
> tem refresh token com rotação e detecção de reuso, validação de
> entrada em duas camadas, rate limiting, e proteção de idempotência
> tanto no produtor quanto no consumidor do evento de doação. Obrigado."

---

## Onde gravar cada coisa

| Item | Onde |
|---|---|
| Diagrama | [`architecture/diagrama.md`](../architecture/diagrama.md) (renderiza no GitHub) |
| Guia de subida completa | [`GUIA-MINIKUBE.md`](../GUIA-MINIKUBE.md) |
| Testes | terminal, dentro de cada repo (`dotnet test`) |
| Swagger da API | `http://localhost:<porta>/swagger` |
| RabbitMQ Management | `http://localhost:15672` (compose) ou NodePort 30672 (minikube) |
| Grafana | `http://localhost:3000` (compose) ou NodePort 30300 (minikube) |
| Actions (CI) | `https://github.com/marcarinivinicius/conexao-solidaria-campaign-api/actions` |

## Erros comuns a evitar na gravação

- Não deixar o `seed-demo-data.sh` rodando **antes** de mostrar o Grafana
  — painel vazio não convence ninguém.
- Não pular a explicação do **porquê** de cada teste (o "diga" de cada
  bloco) — rodar `dotnet test` mudo não mostra que a arquitetura foi
  pensada, só que existe uma suíte.
- Testar o fluxo completo (5.1 a 5.6) **uma vez sem gravar** antes da
  gravação de verdade, pra garantir que nenhum token expirou e a fila
  não está com mensagem presa de um teste anterior.
