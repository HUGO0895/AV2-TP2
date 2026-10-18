# AV1-TP-II: cinebridge, recomendação de equipes de produção

Microsserviço (Node.js + TypeScript estrito + Fastify + TypeORM + PostgreSQL) que recomenda uma equipe de profissionais
para um projeto audiovisual usando os padrões Strategy, Template Method, Observer e Visitor (Atividade 1) e Prototype, Factory Method e Abstract Factory (Atividade 2).

**Como instalar, rodar e usar a API, e o que cada atividade acrescentou: veja [TUTORIAL.md](TUTORIAL.md).**

Resumo: `npm install`, copiar `.envExemplo` para `.env`, `docker compose up -d`, `npm run migration:run`, `npm start`. Para um banco antigo, rode também `npm run migration:run` (novo tipo `PUBLICITARIO`). Testes: `npm test` (115).

## Arquitetura

```
src/
├── app.ts                    # composição das dependências, seed e start
├── server.ts                 # montagem do Fastify (recebe o service por injeção)
├── controller/               # schemas de validação e handlers HTTP
├── servico/                  # ServiceRecomendacao (cache, retry, fallback) + porta ProfissionalProvider
├── SistemaRecomendacao/      # fachada: orquestra, substitui membros, responde convites, notifica
├── Equipe/ e Projetos/       # modelos de domínio puros (Equipe, MembroEquipe, Convite, Projeto...)
├── Fabricas/                 # Abstract Factory: FabricaProjeto + fábricas por tipo (documentário, animação, publicitário, ficção)
├── Strategy/                 # 3 estratégias + StrategyFactory
├── Observer/                 # notificações, auditoria e PublicadorEventos (EventEmitter)
├── Visitor/                  # validação, compatibilidade e relatório
├── Entidades/ Repositorio/   # persistência TypeORM + adaptador ProfissionalProviderTypeorm
└── migration/
```

O domínio não conhece o banco: `ServiceRecomendacao` depende da interface `ProfissionalProvider`,
implementada por `ProfissionalProviderTypeorm` (inversão de dependência).

## Padrões de projeto utilizados

| Padrão | Onde | Papel |
|---|---|---|
| **Strategy** | `src/Strategy/*` | cosseno, filtragem colaborativa e regras de orçamento; escolhidas pelo nome via `StrategyFactory`, com parâmetros |
| **Template Method** | `Equipe/Modelo/OrquestradorEquipe.ts` | esqueleto fixo `validar → filtrar disponíveis → recomendar → normalizar → pós-processar`; não pode ser sobrescrito |
| **Observer** | `src/Observer/*` | e-mail, mensagem interna, auditoria (JSON) e publicador para o barramento de eventos |
| **Visitor** | `src/Visitor/*` | `ValidadorConscistencia`, `CalculadorCompatibilidade`, `GeradorRelatorio` |
| **Prototype** | `Projetos/Modelo/*` | `Projeto.clonar()` (cópia profunda) em cada subclasse; `RegistroPrototipos` cria projetos a partir de modelos |
| **Factory Method** | `Fabricas/FabricaProjeto.ts` | `criarProjeto`, `criarValidador`, `criarRecomendador`, `criarNotificador`: cada fábrica decide a classe |
| **Abstract Factory** | `Fabricas/*` + `RegistroFabricas` | família completa por tipo; o orquestrador recebe a fábrica no construtor e não tem `if` por tipo |
| **Facade** | `SistemaRecomendacao` | ponto único: recomendar, substituir membro, responder convite |
