# github-actions-cicd-rollback-tests

Pipeline de CI/CD com GitHub Actions para uma API em Go no Amazon ECS, com duas proteções a mais: rollback automático quando o deploy não responde e teste de carga em um ambiente de homologação criado sob demanda.

Projeto do curso **Integração Contínua: Rollback e teste de carga**, da Alura. Continuação de [github-actions-cicd-ecs](https://github.com/ssdvd/github-actions-cicd-ecs).

## O pipeline

```
push / pull request
        │
      test ──► build ──► docker ──┬──► deploy_ecs  (deploy + rollback)
                                  └──► loadtest    (teste de carga)
```

| Job | Workflow | O que faz |
| --- | --- | --- |
| `test` | [`go.yml`](.github/workflows/go.yml) | Matriz com Go `1.19`, `1.20` e `>=1.20`; sobe o PostgreSQL com `docker-compose` |
| `build` | [`go.yml`](.github/workflows/go.yml) | Publica o binário `main` como artefato (`app_go`) |
| `docker` | [`docker.yml`](.github/workflows/docker.yml) | Builda e envia a imagem `ssdvd/go_ci:<número da execução>` |
| `deploy_ecs` | [`ecs.yml`](.github/workflows/ecs.yml) | Publica a nova task definition e desfaz o deploy se a aplicação não responder |
| `loadtest` | [`loadtest.yml`](.github/workflows/loadtest.yml) | Cria o ambiente de homologação, roda o Locust e destrói o ambiente |

### Rollback

1. Antes do deploy, a task definition em uso é salva em `task-definition.json.old`.
2. A nova revisão é publicada no serviço `service-api-go`.
3. Depois de 30 segundos, o pipeline faz uma requisição ao load balancer.
4. Se a requisição falhar, a task definition antiga é publicada de novo.

### Teste de carga

1. O Terraform cria o ambiente de homologação a partir do repositório de infraestrutura do curso.
2. Um `locustfile.py` é gerado no próprio workflow.
3. O [Locust](https://locust.io/) roda sem interface por 60 segundos, com 10 usuários e taxa de 5 novos usuários por segundo, contra o load balancer criado.
4. O ambiente é destruído ao final.

> Os passos `go test` e `go build` estão comentados no `go.yml`; o artefato publicado é o binário `main` versionado no repositório.

### Secrets necessários

| Secret | Uso |
| --- | --- |
| `USER_DOCKERHUB`, `PW_DOCKERHUB` | Login no Docker Hub |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | Credenciais da AWS (região `us-east-2`) |
| `DBHOST`, `DBPORT`, `DBUSER`, `DBPASSWORD`, `DBNAME` | Conexão com o PostgreSQL |

### Infraestrutura esperada

- Cluster ECS `api-go`, serviço `service-api-go` e task definition `task-api-go` com um container chamado `go`.
- O endereço do load balancer usado na checagem do rollback está fixo no `ecs.yml` e precisa ser trocado pelo do seu ambiente.

## A aplicação

API REST de cadastro de alunos escrita em Go com [Gin](https://gin-gonic.com/) e [GORM](https://gorm.io/), usando PostgreSQL. É a aplicação de exemplo dos cursos da Alura (`guilhermeonrails/api-go-gin`); o foco deste repositório é o pipeline, não a API.

| Método | Rota | Descrição |
| --- | --- | --- |
| `GET` | `/:nome` | Saudação em JSON |
| `GET` | `/alunos` | Lista todos os alunos |
| `GET` | `/alunos/:id` | Busca um aluno pelo ID |
| `GET` | `/alunos/cpf/:cpf` | Busca um aluno pelo CPF |
| `POST` | `/alunos` | Cria um aluno (`nome`, `cpf` com 11 dígitos, `rg` com 9 dígitos) |
| `PATCH` | `/alunos/:id` | Edita um aluno |
| `DELETE` | `/alunos/:id` | Remove um aluno |
| `GET` | `/index` | Página HTML com a lista de alunos |

## Rodando localmente

Pré-requisitos: Go 1.19 ou superior, Docker e Docker Compose.

```bash
# sobe o PostgreSQL e o pgAdmin
docker-compose up -d

# variáveis lidas pela aplicação para conectar no banco
export HOST=localhost USER=root PASSWORD=root DBNAME=root DBPORT=5432

go run main.go              # API em http://localhost:8080
go test -v main_test.go     # testes de integração (precisam do banco no ar)
```

O pgAdmin fica em <http://localhost:54321>.

## Série de CI/CD com GitHub Actions

Este repositório faz parte de uma sequência em que o mesmo pipeline vai ganhando etapas:

| # | Repositório | O que acrescenta |
| --- | --- | --- |
| 1 | [github-actions-ci](https://github.com/ssdvd/github-actions-ci) | Testes automatizados e matriz de versões do Go |
| 2 | [github-actions-ci-docker](https://github.com/ssdvd/github-actions-ci-docker) | Build da imagem e push para o Docker Hub |
| 3 | [github-actions-cicd-ec2](https://github.com/ssdvd/github-actions-cicd-ec2) | Deploy contínuo em uma instância EC2 via SSH |
| 4 | [github-actions-cicd-ecs](https://github.com/ssdvd/github-actions-cicd-ecs) | Deploy contínuo no Amazon ECS |
| 5 | [github-actions-cicd-rollback-tests](https://github.com/ssdvd/github-actions-cicd-rollback-tests) | Rollback automático e teste de carga |
| 6 | [github-actions-cicd-kubernetes](https://github.com/ssdvd/github-actions-cicd-kubernetes) | Deploy contínuo no Kubernetes (EKS) |

As anotações de cada aula estão na pasta [`notes/`](notes).
