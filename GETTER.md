# Getter Workflow SDK

Fork de [vercel/workflow](https://github.com/vercel/workflow) criado para desenvolver uma camada Agentic First sobre o motor durável upstream.

## Estado desta entrega

- Fork remoto: [SamirCarneiro/getter-workflow-sdk](https://github.com/SamirCarneiro/getter-workflow-sdk).
- Base conferida: `b583e4ae4a16903e2bc45281d3b00f9be8951f61`, de 8 de outubro de 2026; pacote `workflow` 5.2.0.
- Branch de trabalho: `getter/agentic-first`.
- `upstream` aponta para Vercel; `origin` aponta para o fork Getter.
- Nesta etapa foram preparados o fork e o desenho da migração. Nenhum pacote Getter foi portado, e o runtime/compiler upstream não recebeu alterações.
- O protótipo `D:\Gettet\getter-workflows` continua funcional e separado, com seu próprio runtime SQLite. Não está integrado ao fork ainda.

## Decisão

Usar o Workflow SDK como base de execução durável. Manter as extensões Getter em pacotes próprios para acompanhar correções e releases upstream com poucas divergências.

| Responsabilidade | Base | Extensão Getter planejada |
| --- | --- | --- |
| Replay, steps, retries, sleeps, hooks e streams | `workflow` e `@workflow/core` | Consumir as APIs públicas; evitar mudar o compiler no primeiro ciclo. |
| Autoria por Codex ou Claude Code | Adaptadores já implementados no protótipo | `@getter/workflow-authoring`: contexto explícito, catálogo, validação e artefato revisável. |
| Contratos Agentic First | Schemas e registry do protótipo | `@getter/workflow-contracts`: schemas, revisão, políticas de ferramentas e identidade de execução. |
| Operação por agentes | CLI/API upstream + MCP do protótipo | `@getter/workflow-mcp`: catálogo, validação, geração, start/status/cancel, eventos e aprovação. |
| Agente com loop de modelo e ferramentas | `WorkflowAgent` de `@ai-sdk/workflow` | Ferramentas autorizadas, limites de passos/custo e contexto do aplicativo. |
| Infraestrutura própria | Interface World e `@workflow/world-postgres` | `@getter/world-postgres`: derivação após validar autenticação, storage, fila, retenção e versões. |

Esses nomes são propostas de módulos, não pacotes existentes ou publicados. O `@workflow/ai` presente no monorepo e `@ai-sdk/workflow` citado no guia atual são pacotes distintos; verificar versões e peers na implementação.

## Dois usos diferentes de agentes

**Autoria:** Codex/Claude Code recebem um objetivo e capacidades conhecidas; geram uma definição para validação e revisão. Os logins desses CLIs não substituem credenciais para chamadas de modelo dentro do workflow.

**Execução:** `WorkflowAgent` usa um provedor do AI SDK durante a execução e persiste o loop de modelo/ferramentas. Streams podem retomar após reconexão. Configurar uma condição de parada explícita para limitar o loop. O provedor pode ser direto; Vercel AI Gateway é uma opção, não requisito universal. [Guia atual](https://workflow-sdk.dev/docs/ai), [WorkflowAgent](https://ai-sdk.dev/docs/agents/workflow-agent).

## Migração do protótipo

1. Portar catálogo, schemas e autoria. Os handlers existentes permanecem código confiável da aplicação; os executores Codex/Claude continuam separados do workflow determinístico.
2. Criar um host pequeno com integração oficial de build e um workflow compilado para proposta/aprovação. Começar com Local World para desenvolvimento.
3. Adaptar operações públicas de start, status, cancel e hooks ao MCP Getter. A API upstream exige workflow compilado; importar TypeScript em um processo MCP sem transformação não cria um workflow durável.
4. Migrar a representação JSON para um interpretador estático compilado, ou gerar TypeScript estático para revisão/build. A decisão deve preservar schema, dependências, deadlines e o catálogo permitido.
5. Validar a execução no Postgres World; adaptar a infraestrutura depois, mantendo a interface World compatível com o major/canal do runtime.

| Protótipo | Mapeamento a validar no adaptador |
| --- | --- |
| `action` | Step compilado com `"use step"`; chave estável para efeitos externos. |
| `sleep` | Primitiva `sleep` no workflow. |
| `wait` | Hook persistido e retomada pelas APIs públicas. |
| `approval` | Hook tipado; autenticação/autorização antes de retomar a decisão. |
| `dependsOn` / `when` | Orquestração determinística em workflow compilado, com paralelismo e condições. |
| `tick` / worker SQLite | Substituir pelo host/World upstream; não manter dois schedulers sobre a mesma execução. |
| `RunRecord` SQLite | Novo run upstream; migração de estado em andamento requer desenho explícito, não conversão automática. |

### Contratos que precisam de adaptação

`start()` não recebe `idempotencyKey`, `runId`, `tenantId` ou `executionContext` na API pública desta versão. Passar o contexto serializável nos argumentos do workflow e manter a associação tenant/run e a autorização no banco Getter. `attributes` serve para IDs pesquisáveis e é armazenado em texto simples. `getRun(id)` retorna um handle; o status é obtido com `await run.status`. O estado Getter `waiting` será uma projeção de hooks/timers pendentes, pois o enum upstream contém apenas `pending`, `running`, `completed`, `failed` e `cancelled`.

Preservar idempotência de start exige um registro durável de pedidos, chave/hash e recuperação de falhas; consultar antes de chamar `start()` não garante atomicidade. Um hook com token determinístico e verificação de conflito antes de efeitos sensíveis pode impedir trabalho duplicado, mas não substitui o registro que deve devolver o mesmo run ao cliente. A retenção/liberação do token precisa acompanhar a janela de deduplicação.

Para `wait`, implementar inbox durável com deduplicação por ID externo: `resumeHook()` exige hook já registrado e não aceita chave pública de idempotência para o evento. Eventos enviados a hooks registrados são bufferizados. Dois nós que consomem o mesmo evento precisam de roteamento para preservar consumo exclusivo; aguardar o mesmo hook em paralelo não equivale a uma fila de consumidores.

O protótipo usa uma tentativa por padrão; upstream usa três retries, isto é, quatro tentativas. Cada wrapper de ação deve configurar `maxRetries = (maxAttempts ?? 1) - 1`. Wrappers separados permitem retry por nó. Retries enfileirados têm resolução de segundos e mínimo de um segundo; delays antigos de zero ou subsegundo não têm equivalência exata. Preservar também outputs `null`, propagação de nós ignorados, limites de concorrência e rejeição de aprovação.

Essas diferenças foram conferidas no código da base indicada e deverão virar casos de teste do adaptador, não promessas de compatibilidade já entregue.

Aceitação da primeira integração: autoria gera artefato validado; start executa uma proposta; aprovação autenticada retoma; reiniciar o host conserva progresso; MCP lê status/stream; nenhuma ação fora do catálogo é executada; testes de idempotência, prazo, cancelamento e versionamento passam.

## Self-hosting e segurança

[Postgres World](https://workflow-sdk.dev/worlds/postgres) é uma implementação de referência completa e testada, sem autenticação automática nas rotas de workflow e sem otimização prometida para escala/segurança. Requer PostgreSQL, migrations e worker permanente. O fork não incorpora a infraestrutura gerenciada da Vercel.

Antes de disponibilizar um serviço próprio, projetar autenticação entre fila/host/CLI, autorização de aprovação e inspeção, escopo por tenant, criptografia/retenção e roteamento dos deployments usados por runs longos. `tenantId` em metadados sozinho não cria isolamento. Efeitos externos continuam exigindo idempotência no destino.

Não executar diretamente JavaScript produzido por modelo usando [Dynamic Workflows](https://workflow-sdk.dev/docs/advanced/dynamic-workflows). Essa API é experimental, desativada por padrão e executa source com privilégios do deployment. O mapa de steps e a VM determinística não isolam código não confiável. A rota inicial é artefato estático revisado ou JSON validado interpretado por código compilado.

## Manutenção

Preservar `LICENSE.md`, atribuições e avisos aplicáveis da licença Apache-2.0. Não republicar pacotes sob nomes upstream como se fossem releases oficiais. Alterações Getter usam branches próprias e passam por revisão; não fazer push direto para `main` ou `stable`, conforme `AGENTS.md`.

O checkout upstream exige Node 24 e pnpm 11.24.0. A preparação do fork não executou o build completo desse monorepo; os 52 testes anteriores pertencem ao protótipo Getter, não a esta cópia upstream.
