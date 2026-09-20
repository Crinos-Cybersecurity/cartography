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

- `origin` → nosso fork (`Crinos-Cybersecurity/cartography`, onde fazemos push)
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

## Instalação na instância "cloud" — **a partir DESTE fork**

`python3.12 -m venv /opt/cartography-venv && /opt/cartography-venv/bin/pip
install -e /opt/cartography`, onde `/opt/cartography` é o código deste
submódulo copiado por `rsync`. **Nunca `pip install cartography` do
PyPI.** Passo a passo em `backend/infra/deploy/DEPLOY.md`, seção 8.

Mesma razão do fork do Prowler: a versão em produção passa a ser a
fixada no ponteiro de submódulo, e não a última que o PyPI serviu no dia
do deploy — um upgrade que renomeie ou remova um módulo de sync quebraria
a lista de `--aws-requested-syncs` do worker em silêncio.

**VENV DEDICADO, não compartilhado com o Prowler** — os dois têm
conflito duro de dependências (detalhe em `cloud-tools/prowler/CLAUDE.md`).

Neo4j continua efêmero via Docker (imagem oficial `neo4j`, subida e
derrubada pelo worker a cada scan — nunca um serviço permanente na
instância).

## Extensão sem tocar no fork

`--analysis-job-directory` roda Cypher arbitrário ao fim do sync. Serve
para **derivar** a partir do que já está no grafo; **não coleta nada da
AWS**. Ou seja, não resolve lacuna de COLETA — para essas, ou se corrige
o fork (com PR upstream), ou o worker complementa o grafo por conta
própria depois do sync, usando o boto3 e a credencial que ele já tem.

### Lacunas de coleta conhecidas (conferidas no upstream atual)

- **Cognito user pool nunca sincronizado.** Em `intel/aws/cognito.py`,
  `get_user_pools()` está dentro do `else` de `if not identity_pools`.
  Conta que usa Cognito só para login de aplicação (user pool, sem
  identity pool federado) fica com ZERO dado de Cognito no grafo,
  registrado apenas como `Skipping sync`.
- **Regra em event bus customizado invisível.**
  `get_eventbridge_rules()` chama `list_rules()` sem `EventBusName`, o
  que só devolve o barramento `default`.

**As duas já estão cobertas**, sem patch, por
`backend/infra/deploy/cloud/complemento_cartography.py`. Ele roda depois
do sync e IMPORTA as funções deste projeto em vez de reimplementá-las —
`get_user_pools()`, `transform_user_pools()`, `load_user_pools()` e
`load_eventbridge_rules()` são todas públicas e funcionam; o que falta
no upstream é apenas a CHAMADA. Consequência: os nós nascem com o schema
canônico por construção.

Três detalhes que o teste revelou e que vale conhecer antes de mexer
nele:

- **Herda o `lastupdated` do sync**, lido do nó `AWSAccount`. Usar um tag
  próprio faria os nós parecerem de outra execução, e a limpeza do
  Cartography apaga nó com tag antigo.
- **`list_rules` não devolve `EventBusName`** em cada regra; o
  complemento injeta. Sem isso a propriedade fica nula e nem a própria
  verificação de auto-aposentadoria reconheceria as regras.
- **Auto-aposentadoria**: cada complemento verifica antes se o upstream
  já preencheu e, em caso afirmativo, só registra em log. No dia em que
  o bug for corrigido lá, isto vira no-op sozinho — e o gate de
  regressão (`cloud/validacao/regressao.py`) avisa que a lacuna sumiu,
  para que o código morto seja removido em vez de carregado para sempre.
