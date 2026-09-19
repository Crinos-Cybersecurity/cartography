# cartography (fork interno) — documentação do IronBOT

**Nota**: o arquivo `CLAUDE.md` deste diretório já existe no upstream
(symlink pra `AGENTS.md`, guia de contribuição de módulos do próprio
Cartography) — não sobrescrito, de propósito, pra não destruir conteúdo
real deles. Este arquivo (`IRONBOT_FORK.md`) é o equivalente do
`strix/CLAUDE.md`/`cloud-tools/prowler/CLAUDE.md` pra este fork
específico — mesmo conteúdo/propósito, nome diferente só por causa
dessa colisão.

Este diretório é um FORK do Cartography open source. **Achado real ao
forkar (2026-09-16)**: o repositório MUDOU de organização — não está
mais em `lyft/cartography` (organização original), e sim em
`cartography-cncf/cartography` (projeto doado à CNCF) — confirmado via
`gh repo view` antes de forkar, não assumido pelo nome antigo. Licença
**Apache License 2.0** — confirmada lendo o arquivo `LICENSE` real do
repositório no momento do fork, não assumida por nome.

## Remotos configurados

- `origin` → nosso fork (`angelotieres/cartography`, onde fazemos push)
- `upstream` → repositório oficial atual, `cartography-cncf/cartography`
  — só pull/fetch, nunca push

## O que foi customizado em relação ao upstream

**Nenhuma customização de código até agora** — fork mantido em
paridade total com o upstream. É invocado como CLI (`cartography
--neo4j-uri ... --aws-requested-syncs ...`) via subprocess pelo worker
da instância "cloud", sincronizando dados reais da conta AWS do
cliente pra dentro de um Neo4j EFÊMERO (um container por scan, nunca
compartilhado entre execuções — ver justificativa de isolamento
multi-tenant em `backend/CLAUDE.md`).

## Estratégia de merge com upstream

Sem nenhuma customização hoje, sync com upstream deveria ser sempre
limpo. Se precisarmos de um sync novo (ex.: um recurso AWS que o
Cartography ainda não coleta), preferir contribuir/aguardar upstream em
vez de fork permanente divergente, dado que o projeto é mantido pela
CNCF (ciclo de release ativo) — mesma disciplina do `strix/CLAUDE.md`.

## Integração com o resto do projeto

- **Papel no pipeline de nuvem** (ver `backend/CLAUDE.md`, seção
  "Correlação de cadeia de ataque em nuvem (AWS) — MVP"): constrói o
  grafo real da conta AWS (IAM roles/users/policies, S3, EC2, VPC/SG,
  Lambda...) — a SEGUNDA fonte de dado da correlação (a primeira é o
  snapshot do Prowler). As queries Cypher de cadeia de ataque
  (`backend/infra/deploy/cloud/attack_paths.cypher`) rodam CONTRA o
  grafo que este fork constrói.
- **Neo4j efêmero, nunca compartilhado**: multi-tenant — se o grafo de
  duas contas AWS de clientes diferentes coexistisse no mesmo Neo4j
  (mesmo com tags de `organization_id`), um bug de filtro de query
  vira vazamento de dado de infraestrutura entre clientes. Inaceitável
  num produto de segurança. O worker sobe/derruba um container Neo4j
  por execução de scan (`docker run`/`docker stop`), sempre no
  `finally` do fluxo — dado nunca sobrevive entre execuções.
- **Credenciais**: mesmo mecanismo do Prowler — STS AssumeRole
  temporário, repassado só como env var pro processo filho.

## Instalação na instância "cloud"

`pip install cartography` + Neo4j efêmero via Docker (imagem oficial
`neo4j`, subida/derrubada pelo worker por scan — não um serviço Neo4j
permanente na instância).
