# Tutorial completo: cinebridge (microsserviço de recomendação de equipes)

Este tutorial tem duas partes que se encaixam:

- **Parte A (seções 1 a 4):** como instalar e como funciona o que **já existia** (Atividade 1: Strategy, Template Method, Observer, Visitor).
- **Parte B (seção 5):** o que foi **adicionado agora** (Atividade 2: Prototype, Factory Method e Abstract Factory) e como ele se liga ao que já existia.

As seções 6 em diante são de uso (API, eventos, testes, extensão, problemas comuns).

---

## 1. Pré-requisitos

| Item | Versão | Observação |
|---|---|---|
| Node.js | 20 LTS ou superior | `node -v` |
| PostgreSQL | 16 | via Docker (recomendado) ou instalado na máquina |
| Docker Desktop (Windows) / Docker Engine (Linux) | qualquer recente | só para subir o Postgres |

**Compatibilidade exigida pela atividade:** Windows 10 ou superior, Ubuntu 24.04.03 ou superior e derivados do Ubuntu. Os scripts do `package.json` usam só `npm` (nada de bash ou caminhos com `/` fixos), então os mesmos comandos funcionam nos três. O arquivo `.gitattributes` normaliza as quebras de linha entre Windows e Linux.

## 2. Instalação e execução

**Ubuntu / derivados**
```bash
cd AV1-TP-II-main
npm install
cp .envExemplo .env
docker compose up -d        # sobe o Postgres com os dados do .env
npm run migration:run       # cria as tabelas (inclui o tipo PUBLICITARIO, novo)
npm start                   # API em http://localhost:3000
```

**Windows 10+ (PowerShell)**
```powershell
cd AV1-TP-II-main
npm install
copy .envExemplo .env
docker compose up -d
npm run migration:run
npm start
```

Na primeira execução a API popula o banco com 32 profissionais, 10 projetos (agora alternando os 4 tipos) e 160 avaliações. Se o banco já tiver dados, o seed é ignorado.

Sem Docker: instale o PostgreSQL, crie banco e usuário e ajuste `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASS` e `DB_NAME` no `.env`. Modo desenvolvimento: `npm run dev`. Outra porta: `PORT` no `.env`.

Teste rápido: `curl http://localhost:3000/saude` deve responder `{"status":"ok"}`.

> **Atenção, banco já existente:** se você já tinha rodado a versão anterior, rode `npm run migration:run` de novo. A nova migration `AddTipoPublicitario` acrescenta o valor `PUBLICITARIO` ao enum `projetos_tipo_enum` do Postgres.

## 3. Mapa do projeto

```
src/
├── app.ts                    # monta tudo (registro de fábricas, sistema, service), seed e start
├── server.ts                 # Fastify (recebe service e registro por injeção)
├── controller/               # schemas de validação e handlers HTTP
├── servico/                  # ServiceRecomendacao: cache, retry, fallback
├── SistemaRecomendacao/      # fachada: recomendar, substituir membro, responder convite
├── Equipe/ e Projetos/       # modelos de domínio (Equipe, Profissional, Projeto + subclasses)
├── Strategy/                 # 3 estratégias + StrategyFactory           (Atividade 1)
├── Observer/                 # e-mail, interno, auditoria, barramento     (Atividade 1)
├── Visitor/                  # validador, compatibilidade, relatório      (Atividade 1)
├── Fabricas/                 # NOVO: FabricaProjeto, 4 fábricas, RegistroFabricas,
│   ├── Validadores/          #       validadores, recomendadores e notificadores por tipo
│   ├── Recomendadores/
│   └── Notificadores/
├── Entidades/ Repositorio/   # persistência TypeORM
├── migration/                # esquema do banco (+ AddTipoPublicitario, novo)
└── __tests__/                # 115 testes (83 antigos + 32 novos em fabricas/)
```

## 4. Como funciona o que já existia (Atividade 1)

### 4.1 O caminho de uma requisição

```
POST /  ──► Controller ──► ServiceRecomendacao ──► SistemaRecomendacao ──► OrquestradorEquipe
            (valida o       (busca profissionais,    (fachada: escolhe a      (Template Method:
             corpo, monta    com cache e retry)       fábrica, orquestra,       validar → filtrar →
             o Projeto)                               notifica)                 recomendar → normalizar
                                                                                → pós-processar)
```
O domínio não conhece o banco: o service depende da interface `ProfissionalProvider`, implementada por `ProfissionalProviderTypeorm` (inversão de dependência).

### 4.2 Os quatro padrões da Atividade 1

| Padrão | Onde | O que faz |
|---|---|---|
| **Strategy** | `src/Strategy/*` | Três algoritmos de recomendação trocáveis pelo nome (`similiaridadecosseno`, `filtragemcolaborativa`, `regrasdeorcamento`), criados por `StrategyFactory` com parâmetros. |
| **Template Method** | `Equipe/Modelo/OrquestradorEquipe.ts` | `orquestrar()` fixa o esqueleto `validar → filtrar disponíveis → recomendar → normalizar → pós-processar`. O construtor bloqueia sobrescrever `orquestrar()`. `OrquestradorPadrao` preenche os passos variáveis. |
| **Observer** | `src/Observer/*` | `NotificadorEmail`, `NotificadorInterno`, `AuditoriaRecomendacao` e `PublicadorEventos` (barramento em memória) recebem cada `EventoRecomendacao`. |
| **Visitor** | `src/Visitor/*` | `ValidadorConscistencia`, `CalculadorCompatibilidade` e `GeradorRelatorio` percorrem `Projeto` e `Profissional` via `aceitar(visitante)`. |
| **Facade** | `SistemaRecomendacao` | Ponto único: recomendar, substituir membro, responder convite. |

### 4.3 Por que a Atividade 2 era necessária

Antes só existia **um** modelo de `Projeto`, com atributos fixos. O campo `tipo` era só um texto: documentário, animação e ficção eram tratados igual. Os pontos de dor que a atividade pede para identificar:

| Sintoma | Onde aparecia no código antigo |
|---|---|
| **Rigidez** | Para um tipo novo com regras próprias, teria de mexer em `SistemaRecomendacao`, no controller e nos validadores, com `if (tipo === ...)` espalhados. |
| **Imobilidade** | `SistemaRecomendacao.substituirMembro` montava o subprojeto com `new Projeto(...)`, copiando campo a campo. Um projeto com atributos extras perderia esses atributos. |
| **Fragilidade** | O `ValidadorConscistencia` existia, mas o orquestrador nunca o usava: nada garantia que a equipe respeitasse as regras de cada tipo. |

## 5. O que foi adicionado (Atividade 2)

### 5.1 Visão geral: o que o PDF pede e onde está

| Pedido da atividade | Onde foi feito |
|---|---|
| Prototype: cada subclasse de `Projeto` tem `clonar()` com cópia profunda (papéis, restrições) | `Projetos/Modelo/projetos.ts` (`clonar`, `copiarEstadoComum`), `ProjetoDocumentario.ts`, `ProjetoAnimacao.ts`, `ProjetoPublicitario.ts` |
| Protótipos registrados e clonados com ajustes finos | `Projetos/Modelo/RegistroPrototipos.ts` + `aplicarAjustes()` |
| Interface `FabricaProjeto` e 3 fábricas concretas (documentário, animação, publicitário), cada uma criando projeto, validador, recomendador e notificador | `Fabricas/FabricaProjeto.ts`, `FabricaDocumentario.ts`, `FabricaAnimacao.ts`, `FabricaPublicitario.ts` |
| `OrquestradorEquipe` recebe a `FabricaProjeto` no construtor e a usa, sem condicionais por tipo | `Equipe/Modelo/OrquestradorEquipe.ts` |
| `SistemaRecomendacao` identifica o tipo, escolhe a fábrica e cria todo o ecossistema | `SistemaRecomendacao/SistemaRecomendacao.ts` + `Fabricas/RegistroFabricas.ts` |
| Testes de integração alternando fábricas, sem alterar o orquestrador | `src/__tests__/fabricas/*.test.ts` |
| Compatível com Windows 10+ e Ubuntu 24.04+ | Seção 1; só `npm` nos scripts; `.gitattributes` |

### 5.2 Prototype: `Projeto.clonar()`

```
            Projeto  ◄── clonar(): Projeto
              ▲
   ┌──────────┼───────────────┐
ProjetoDocumentario   ProjetoAnimacao   ProjetoPublicitario
(tema, entrevistados) (tecnica, cenas)  (cliente, veiculos)
```

- `clonar()` devolve **o mesmo tipo concreto**. Quem clona só conhece `Projeto`.
- A cópia é profunda: o `Map` de competências e cada `Competencias`, o `Map` de pesos, a `Equipe` e seus `MembroEquipe`, o `prazo` (`Date`) e as listas específicas (`entrevistados`, `veiculos`). Mudar a cópia não toca o original (há testes para isso).
- Os `Profissional` dentro da equipe são **compartilhados por referência**: são entidades independentes do projeto (o mesmo profissional pode estar em vários projetos), então não fazem parte do agregado copiado.
- `aplicarAjustes({ id, orcamento, prazo, detalhes... })` muda o clone depois de criado. Cada subclasse trata os seus `detalhes`.

**RegistroPrototipos** guarda modelos pelo nome e entrega clones:

```ts
const registro = new RegistroPrototipos()
  .registrar("doc-curto", new ProjetoDocumentario("m", Genero.DRAMA, 52, 80000, prazo, competencias, undefined, undefined, undefined, "Modelo", []))

const meuProjeto = registro.criar("doc-curto", { id: "42", orcamento: 95000, detalhes: { tema: "Mangues", entrevistados: ["Ana"] } })
```
O modelo registrado nunca é alterado e o registro guarda uma cópia do que recebeu.

**Onde o clone já é usado de verdade:**
1. `SistemaRecomendacao.substituirMembro` cria o subprojeto com `projeto.clonar().aplicarAjustes(...)`: o tipo e os atributos do projeto são preservados.
2. Os recomendadores por tipo trabalham sobre um clone, para reforçar competências sem alterar o projeto do cliente.
3. A validação de consistência valida um clone com a equipe candidata.

### 5.3 Factory Method e Abstract Factory: `FabricaProjeto`

`FabricaProjeto` é uma classe abstrata com quatro **métodos fábrica**. Cada fábrica concreta decide qual classe instanciar. O conjunto das quatro é a **família** de um tipo (Abstract Factory):

| Método | Documentário | Animação | Publicitário | Ficção (genérica) |
|---|---|---|---|---|
| `criarProjeto(dados)` | `ProjetoDocumentario` | `ProjetoAnimacao` | `ProjetoPublicitario` | `Projeto` |
| `criarValidador()` | `ValidadorDocumentario` | `ValidadorAnimacao` | `ValidadorPublicitario` | `ValidadorConscistencia` |
| `criarRecomendador(base?)` | `RecomendadorDocumentario` | `RecomendadorAnimacao` | `RecomendadorPublicitario` | a própria estratégia do cliente |
| `criarNotificador()` | `NotificadorDocumentario` | `NotificadorAnimacao` | `NotificadorPublicitario` | `NotificadorFiccao` |

A quarta coluna (`FabricaFiccao`) é um acréscimo: o enum `TipoProjeto` já tinha `FICÇÃO`, e essa fábrica mantém o comportamento original, o que garante que nada do que existia quebrou.

**O que cada tipo faz de diferente**

| Tipo | Papéis essenciais (validador) | Competências priorizadas (recomendador) | Notificador avisa |
|---|---|---|---|
| Documentário | `DIRETOR`, `EDITOR` | Roteirista: Roteiro 4; Editor: Edição 4 | produção, para agendar entrevistas |
| Animação | `DIRETOR`, `EFEITOS_VISUAIS` | Efeitos visuais: Animação 3D 4 e Efeitos Visuais 4; Diretor: Animação 3D 3 | estúdio, para reservar renderização |
| Publicitário | `DIRETOR`, `DIRETOR_FOTOGRAFIA` | Fotografia: Fotografia 4; Diretor: Produção 3 | agência, para aprovação do cliente |
| Ficção | (só as regras gerais) | nenhuma | pré-produção |

- **Validador (Visitor):** além das regras gerais (todos os papéis pedidos preenchidos e custo dentro do orçamento), exige os papéis essenciais do tipo. Se a equipe passa nas regras gerais mas falha nas do tipo, o status vira `inconsistente`.
- **Recomendador:** *decora* a estratégia que o cliente escolheu (`estrategia` no JSON). Com cosseno, as competências priorizadas entram na exigência do papel. Com `filtragemcolaborativa` e `regrasdeorcamento`, que não comparam competências, quem tem as competências priorizadas passa à frente no desempate entre candidatos. A escolha da estratégia continua sendo do cliente.
- **Notificador (Observer):** é chamado além dos observadores gerais (e-mail, interno, auditoria).

### 5.4 Orquestrador e SistemaRecomendacao

**`OrquestradorEquipe`** agora recebe a fábrica no construtor e o esqueleto continua o mesmo, sem `if` por tipo:

```ts
orquestrar(projeto, estrategia, profissionais) {
  validarRestricoes(projeto)                                   // passo abstrato
  candidatos = filtrarDisponiveis(projeto, profissionais)
  recomendador = this.fabrica.criarRecomendador(estrategia)    // vem da fábrica
  recomendacoes = recomendador.recomendar(projeto, candidatos)
  equipe = normalizarDados(recomendacoes)                      // passo abstrato
  concluir(equipe, projeto)   // posProcessar (abstrato) + validador da fábrica
  return equipe
}
```
`OrquestradorPadrao` não mudou. Por isso um tipo novo não exige alterar o orquestrador.

**`SistemaRecomendacao`** ganhou um `RegistroFabricas` (padrão: os 4 tipos). Em cada operação ele faz `registro.obter(projeto.tipo)`, cria o orquestrador com essa fábrica e notifica com o notificador dela. O construtor continua aceitando só a lista de observadores, então o código antigo segue funcionando.

**`RegistroFabricas`** é uma tabela `tipo → fábrica`, não uma cadeia de `if`/`switch`. Tipo sem fábrica gera erro 400 com a lista dos tipos disponíveis.

**Controller:** `montarProjeto(body, registro)` usa `registro.obter(body.tipo).criarProjeto(...)`. O campo opcional `detalhes` carrega os atributos específicos.

### 5.5 Banco de dados

`TipoProjeto` ganhou `PUBLICITARIO`. Como o Postgres guarda o tipo como `ENUM`, a migration `1790000000000-AddTipoPublicitario.ts` executa `ALTER TYPE ... ADD VALUE IF NOT EXISTS 'PUBLICITARIO'`. O seed passou a alternar os 4 tipos.

### 5.6 Testando as novidades pela API

Há um exemplo pronto para cada tipo em `exemplos/`.

```bash
curl -X POST http://localhost:3000/ -H "content-type: application/json" -d @exemplos/documentario.json
curl -X POST http://localhost:3000/ -H "content-type: application/json" -d @exemplos/animacao.json
curl -X POST http://localhost:3000/ -H "content-type: application/json" -d @exemplos/publicitario.json
```
```powershell
Invoke-RestMethod -Uri http://localhost:3000/ -Method Post -ContentType 'application/json' -InFile .\exemplos\documentario.json | ConvertTo-Json -Depth 8
```

**`detalhes` por tipo** (todos opcionais; ignorados em `FICÇÃO`):

| Tipo | Campos |
|---|---|
| `DOCUMENTARIO` | `tema` (texto), `entrevistados` (lista de textos) |
| `ANIMAÇÃO` | `tecnica` (texto), `cenas` (número) |
| `PUBLICITARIO` | `cliente` (texto), `veiculos` (lista de textos) |

**Experimento para ver o Abstract Factory trabalhando:** pegue `exemplos/documentario.json`, remova o papel `EDITOR` de `competencias` e envie. A resposta vem com `"status": "inconsistente"`: a equipe está completa e no orçamento, mas o validador de documentário exige um editor. Troque `"tipo"` para `"FICÇÃO"` e reenvie: o mesmo pedido volta `"formada"`. Nada no orquestrador mudou entre as duas chamadas, só a fábrica.

Para ver o notificador do tipo, olhe o console da API: aparece uma linha JSON `"evento":"notificacao_tipo"` com `"observador":"NOTIFICADOR-DOCUMENTARIO"` (ou o do tipo usado).

### 5.7 Diferenças em relação ao diagrama da atividade

| No diagrama | No código | Motivo |
|---|---|---|
| `recomendar(...): Map<Papel, Profissional[]>` | `Map<Papel, Profissional>` | Mantém a assinatura que a Atividade 1 e os testes já usavam (um profissional por papel). |
| `criarRecomendador()` sem parâmetro | `criarRecomendador(base?)` | O parâmetro é opcional. Permite que o recomendador do tipo envolva a estratégia escolhida pelo cliente. |
| `orquestrar(projeto, estrategia)` | `orquestrar(projeto, estrategia, profissionais)` | O orquestrador não acessa banco; recebe os profissionais já carregados. |
| Só `ProjetoDocumentario` visível (imagem cortada) | `ProjetoAnimacao` e `ProjetoPublicitario` com atributos próprios | Atributos de animação e publicitário **inventados** (técnica/cenas; cliente/veículos). Ajuste se o diagrama completo for diferente. |
| 3 fábricas | 3 + `FabricaFiccao` | Mantém `FICÇÃO` funcionando como antes. |

## 6. A API HTTP (já existia, agora com tipos de projeto)

| Método e rota | Para quê |
|---|---|
| `GET /saude` | verificar se o serviço está de pé |
| `POST /` | recomendar uma equipe para um projeto |
| `POST /equipe/substituir` | trocar o profissional de um papel, mantendo os demais |
| `POST /convites/responder` | aceitar (confirma) ou recusar (substitui) uma sugestão |

### 6.1 Recomendar uma equipe

O arquivo `exemplos/projeto.json` já traz um corpo pronto.

```bash
curl -X POST http://localhost:3000/ -H "content-type: application/json" -d @exemplos/projeto.json
```
```powershell
Invoke-RestMethod -Uri http://localhost:3000/ -Method Post -ContentType 'application/json' -InFile .\exemplos\projeto.json | ConvertTo-Json -Depth 8
```

**Campos do corpo**

| Campo | Obrigatório | Descrição |
|---|---|---|
| `id` | sim | número do projeto |
| `genero` | sim | `COMEDIA`, `AÇÃO`, `TERROR` ou `DRAMA` |
| `tipo` | sim | `DOCUMENTARIO`, `FICÇÃO`, `ANIMAÇÃO` ou `PUBLICITARIO`. **Escolhe a fábrica** (seção 5) |
| `duracaoMin`, `orcamento` | sim | números maiores que 0 |
| `prazo` | sim | data ISO, ex.: `2026-03-10T00:00:00.000Z`. Só entram profissionais disponíveis nessa data |
| `competencias` | sim | papéis exigidos, cada um com a lista `{nome, nivel 1-5}` |
| `pesos` | não | importância de cada papel (padrão 1). Divide o orçamento por papel e pesa a nota em `regrasdeorcamento` |
| `estrategia` | sim | `similiaridadecosseno`, `filtragemcolaborativa` ou `regrasdeorcamento` |
| `parametros` | não | ajustes do algoritmo (tabela abaixo) |
| `detalhes` | não | atributos específicos do tipo (seção 5.6) |

**Papéis aceitos** (valor ou nome do enum): `DIRETOR`, `DIRETOR DE FOTOGRAFIA` (ou `DIRETOR_FOTOGRAFIA`), `SONOPLASTA`, `EDITOR`, `ROTEIRISTA`, `EFEITOS VISUAIS` (ou `EFEITOS_VISUAIS`).

**Estratégias e parâmetros**

| Estratégia | Como escolhe | Parâmetros |
|---|---|---|
| `similiaridadecosseno` | similaridade entre as competências exigidas e as do profissional | `limiarMinimo` (0 a 1, padrão 0) |
| `filtragemcolaborativa` | maior média de avaliações recebidas no papel | `minAvaliacoes` (padrão 1) |
| `regrasdeorcamento` | maximiza a nota total respeitando o orçamento (otimização inteira) | `usarPesos` (padrão true), `topPorPapel` (padrão 50) |

Para ver o Strategy funcionando, envie o mesmo projeto trocando só `estrategia` e compare os profissionais devolvidos.

**Resposta**
```json
{
  "Recomendacao": {
    "dataFormacao": "2026-10-18T17:40:00.000Z",
    "status": "formada",
    "membrosEquipe": [
      { "papel": "DIRETOR", "confirmado": false, "profissional": { "id": "1", "nome": "Ana Lima", "precoMedio": 3000 } }
    ]
  },
  "degradado": false
}
```

| `status` | Significado |
|---|---|
| `formada` | todos os papéis preenchidos e dentro do orçamento |
| `parcial` | algum papel exigido ficou sem profissional elegível |
| `acima_do_orcamento` | o custo da equipe passou do orçamento |
| `confirmada` | todos os membros aceitaram o convite |
| `inconsistente` | **novo:** a equipe está completa e no orçamento, mas o validador do tipo reprovou (ex.: documentário sem editor) |

`degradado: true` indica que o banco estava fora do ar e a resposta usou os últimos dados em cache.

### 6.2 Substituir um membro

Envie o mesmo projeto, a equipe atual (papel → id do profissional) e o papel a trocar. O serviço não guarda estado: a equipe vai e volta no corpo.

```json
{
  "...": "todos os campos do projeto (id, genero, tipo, orcamento, prazo, competencias, estrategia)",
  "equipe": {
    "DIRETOR": { "profissionalId": "1" },
    "EDITOR": { "profissionalId": "4", "confirmado": true },
    "SONOPLASTA": { "profissionalId": "9" }
  },
  "papel": "DIRETOR",
  "rejeitados": ["7"]
}
```

Só o papel `DIRETOR` é recalculado. Os outros ficam fixos, o orçamento considerado é o que sobra depois deles, e quem já está na equipe ou em `rejeitados` não volta.

### 6.3 Responder a um convite

Mesmo corpo da substituição, mais `"aceito": true` ou `false`.

- `true`: o membro fica `confirmado`. Quando todos confirmam, o status vira `confirmada` e é emitido o evento `Equipe Confirmada`.
- `false`: o convite é marcado como recusado e o papel é substituído automaticamente.

### 6.4 Erros

| Código | Quando |
|---|---|
| 400 | corpo inválido, estratégia ou papel desconhecido |
| 422 | regra de negócio: ninguém elegível, sem substituto, orçamento restante esgotado |
| 503 | banco indisponível e sem cache |

Todos devolvem `{ "status": "error", "mensagem": "..." }`.

## 7. Notificações, auditoria e eventos

Cada ação gera uma linha JSON no console, por exemplo:
```json
{"ts":"...","nivel":"info","evento":"auditoria","observador":"LOG AUDITORIA","tipoEvento":"Convite Recusado","origem":"Sistema Recomendação","statusEquipe":"formada","membros":[...]}
```

Tipos de evento: `Recomendação de Equipe`, `Convite Aceito`, `Convite Recusado`, `Membro Substituído`, `Equipe Confirmada`.

Outros módulos podem assinar os eventos pelo barramento em memória (EventEmitter):
```ts
barramento.assinar("Equipe Confirmada", (evento) => { /* gerência de projetos, financeiro */ });
barramento.assinar("*", (evento) => { /* todos os eventos */ });
```
Para trocar por RabbitMQ, crie outra implementação de `BarramentoEventos` e passe-a ao `PublicadorEventos` no `app.ts`.


## 8. Testes

```bash
npm test            # 115 testes (83 da Atividade 1 + 32 novos)
npm run test:cov    # com cobertura (mínimo exigido: 80%)
npm run typecheck   # TypeScript em modo estrito
```
Os testes não precisam de banco. Os de integração usam `fastify.inject` com um repositório falso.

**Testes novos (`src/__tests__/fabricas/`)**

| Arquivo | O que prova |
|---|---|
| `Prototype.test.ts` | clone mantém classe e tipo; é cópia profunda (alterar o clone não afeta o original); registro de protótipos com ajustes e erro para nome desconhecido |
| `AbstractFactory.test.ts` | cada fábrica cria a família certa; `detalhes` chegam ao projeto; registro padrão cobre todos os tipos do enum |
| `IntegracaoFabricas.test.ts` | **integração:** o mesmo pedido gera resultados diferentes conforme a fábrica (recomendação, validação, notificação); trocar a fábrica do mesmo `OrquestradorPadrao` troca o comportamento; um tipo novo (realidade virtual, definido dentro do teste) funciona só com `registrar(...)`; API HTTP com os 4 tipos |

Os testes antigos continuaram passando; a única mudança neles foi `new OrquestradorPadrao()` virar `new OrquestradorPadrao(new FabricaFiccao())`, já que o construtor agora exige a fábrica.

## 9. Como estender

| Quero... | Faça |
|---|---|
| **Novo tipo de projeto (ex.: realidade virtual)** | 1) acrescente o valor em `TipoProjeto` e crie uma migration `ALTER TYPE ... ADD VALUE`; 2) crie o projeto (opcional), validador, recomendador e notificador; 3) crie `FabricaRealidadeVirtual extends FabricaProjeto`; 4) `registro.registrar(new FabricaRealidadeVirtual())` no `app.ts`. **Nenhuma linha do orquestrador ou do SistemaRecomendacao muda.** |
| Modelo pré-configurado de projeto | `new RegistroPrototipos().registrar("nome", projetoModelo)` e depois `.criar("nome", ajustes)` |
| Nova estratégia | classe que implementa `RecomendacaoStrategy` + nome em `src/Strategy/StrategyFactory.ts` |
| Novo observador geral | implemente `Observador` e adicione à lista do `SistemaRecomendacao` no `app.ts` |
| Nova operação sobre projeto ou profissional | implemente `VisitanteProjeto` e chame `projeto.aceitar(visitante)` |
| Outra regra de orquestração | herde `OrquestradorEquipe` e implemente os 3 passos abstratos (`orquestrar` não pode ser sobrescrito) |

## 10. Problemas comuns

| Sintoma | Causa provável |
|---|---|
| `ECONNREFUSED 5432` | Postgres não está rodando (`docker compose ps`) ou o `.env` está errado |
| `relation "profissional" does not exist` | faltou `npm run migration:run` |
| `invalid input value for enum projetos_tipo_enum: "PUBLICITARIO"` | banco antigo sem a nova migration: rode `npm run migration:run` |
| 400 "Não há fábrica registrada para o tipo ..." | tipo sem fábrica no `RegistroFabricas` |
| Status `inconsistente` inesperado | o tipo exige papéis essenciais (tabela da seção 5.3) que o pedido não incluiu |
| 422 "Nenhum profissional elegível" | `prazo` fora da disponibilidade dos profissionais, ou orçamento baixo demais |
| Resposta `parcial` | falta profissional com a competência/avaliação do papel; aumente o orçamento ou relaxe `limiarMinimo` |
| Windows: `npm` bloqueado no PowerShell | rode uma vez `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |

## 11. Limitações conhecidas

- Localização geográfica de projetos e profissionais não foi implementada.
- A equipe final não é gravada no banco: o serviço devolve o resultado e emite eventos; o estado fica com o cliente.
- O barramento de eventos é só em memória (sem RabbitMQ).
- Os protótipos (`RegistroPrototipos`) existem no domínio e nos testes, mas ainda não há endpoint HTTP para listá-los ou cloná-los.
- Os papéis essenciais e as competências priorizadas de cada tipo são regras iniciais de exemplo; ajuste-as com o time de negócio.
