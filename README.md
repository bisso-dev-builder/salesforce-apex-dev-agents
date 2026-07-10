# salesforce-apex-dev-agents

Conjunto de *skills* do Claude Code que implementa um pipeline completo de desenvolvimento Apex — da user story ao código pronto para deploy — com gates de qualidade automáticos em cada etapa.

O pipeline é orquestrado pelo agente **`apex-dev`**, que dispara em sequência cinco agentes especializados: `apex-spec` → `apex-arch` → `apex-impl` → `apex-test` → `apex-review`. Todos eles seguem as regras definidas em `apex-patterns`, que funciona como referência constitucional (Clean Code, SOLID, uso do base-framework interno).

## Pré-requisitos

- [Claude Code](https://claude.com/claude-code) configurado no projeto Salesforce (skills residem em `.claude/skills/`)
- [Salesforce CLI](https://developer.salesforce.com/tools/salesforcecli) (`sf`) instalado e autenticado na org/scratch org de destino
- Projeto Salesforce DX (`force-app/main/default/classes/...`)

## Visão geral do pipeline

| Ordem | Skill | Papel | Entrada | Saída | Gate |
|---|---|---|---|---|---|
| 1 | `apex-spec` | Analista | User story em linguagem natural | `docs/specs/YYYY-MM-DD-<feature>-spec.md` | Gate 1 |
| 2 | `apex-arch` | Arquiteto | Spec doc | `docs/specs/YYYY-MM-DD-<feature>-arch.md` | Gate 2 |
| 3 | `apex-impl` | Desenvolvedor | Spec + Arch doc | Classes `.cls` / `.cls-meta.xml` | Gate 3 |
| 4 | `apex-test` | Testador | Classes de produção | Classes de teste `.cls` | Gate 4 |
| 5 | `apex-review` | Revisor | Todo o código produzido | Relatório de review (ou nada, se aprovado) | Gate 5 |

Cada estágio só avança se o gate anterior passar. Não há execução paralela entre estágios — é uma regra rígida do orquestrador.

## Usando o `apex-dev`

### Modo A — Manual

Invoque cada skill individualmente, revisando o gate a cada passo:

```
/apex-spec    → produz docs/specs/YYYY-MM-DD-<feature>-spec.md; revisar Gate 1
/apex-arch    → produz docs/specs/YYYY-MM-DD-<feature>-arch.md; revisar Gate 2
/apex-impl    → produz arquivos .cls; revisar Gate 3
/apex-test    → produz classes de teste; revisar Gate 4
/apex-review  → loop de auto-fix; Gate 5
```

Use quando: trabalho exploratório, mudanças cirúrgicas, ou quando você quer controle manual total em cada gate.

### Modo B — Semiautomático

```
/apex-dev "texto da user story"
```

Orquestra as cinco skills em sequência. Só para quando um gate falha após esgotar as tentativas de retry da skill.

Use quando: você confia no pipeline e quer execução hands-off no caminho feliz.

### Modo C — Checkpoints

```
/apex-dev --checkpoint <stage1>,<stage2> "texto da user story"
```

Igual ao Modo B, mas pausa nos estágios nomeados para revisão humana mesmo quando o gate passa. Estágios válidos: `spec`, `arch`, `impl`, `test`, `review`.

Exemplo:
```
/apex-dev --checkpoint arch,review "texto da user story"
```

Use quando: um arquiteto precisa aprovar o design antes da implementação, ou um tech lead precisa dar sign-off antes de marcar o ciclo como concluído.

### Retomando ou abortando o ciclo

Se um gate falhar após esgotar as tentativas, o orquestrador imprime o relatório de falha (arquivo, linha, violação, sugestão) e aguarda ação humana — ele **não** re-executa a skill automaticamente.

```
/apex-dev --resume <stage>
```
Re-executa o estágio indicado e todos os subsequentes. Não repete estágios anteriores.

```
/apex-dev --abort
```
Interrompe o orquestrador. Nenhum arquivo é apagado; os artefatos produzidos permanecem no lugar.

### Ao final de um ciclo bem-sucedido

O `apex-dev` imprime um resumo com: caminho do spec, caminho do arch, classes criadas, classes de teste criadas, tentativa em que o Gate 5 passou, e o comando de deploy sugerido (`sf project deploy start --source-dir force-app`).

## Os agentes do pipeline

### `apex-spec` — Spec Agent
Transforma a user story em um doc de 5 seções: Problem Statement, Salesforce Context, Framework Components, Acceptance Criteria (testáveis, numerados) e Out of Scope. **Gate 1** exige que nenhuma seção fique vazia/"TBD", que os componentes de framework sejam citados por nome de classe e que não reste ambiguidade.

### `apex-arch` — Architecture Agent
Lê o spec aprovado e produz o doc de arquitetura: lista de classes (nome, modelo de sharing, classe/interface base, responsabilidade em uma frase), dependências como abstrações, mapeamento requisito → componente de framework, Custom Metadata necessários e caminhos de arquivo. **Gate 2** bloqueia reimplementação de infraestrutura já coberta pelo framework e proíbe acrônimos/abreviações em nomes.

### `apex-impl` — Implementation Agent
Implementa as classes exatamente como definidas no arch doc, em ordem de dependência, estendendo as classes base do framework (`Logger`, `Lists`/`Maps`, sem `System.debug`, sem SOQL/DML em loop). Roda `sf project deploy start --dry-run --source-dir force-app` após cada classe. **Gate 3** exige zero erros de compilação e zero violações das regras acima.

### `apex-test` — Test Agent
Escreve testes para cada classe de produção usando `FixtureFactory`/`HttpMock`/`MultiHttpMock` e `Assert.*` (nunca `System.assert*`), cobrindo caminho feliz, edge case e cenário bulk (200 registros) quando aplicável. **Gate 4** exige `sf apex test run -c` sem falhas e 100% de cobertura de linha nas classes novas.

### `apex-review` — Review Agent
Passa todo o código pelo checklist de Clean Code, SOLID e aderência ao base-framework. Roda um loop de auto-fix de até 3 tentativas (renomear acrônimos, trocar `System.debug` por `Logger`, extrair métodos, injetar dependências por construtor), sempre revalidando com `sf apex test run -c`. Se ainda houver violações na 4ª tentativa, **Gate 5** bloqueia e gera `.superpowers/review/YYYY-MM-DD-<feature>-review.md` com o relatório detalhado.

### `apex-patterns` — referência constitucional
Não é invocado diretamente. Define o *Agent Contract* (regras de Clean Code, SOLID e base-framework) que todos os agentes acima leem antes de agir, além de indexar as referências detalhadas em `references/` (`apex-dev-guide.md`, `apex-base-framework.md`, `repository.md`, `trigger-patterns.md`, `test-patterns.md`, `lightning-web-components.md`, `soql-sosl.md`, `deployment-devops.md`).

## Exemplo rápido

```
/apex-dev "Como gestor de vendas, quero que uma tarefa de follow-up seja criada
automaticamente quando uma Oportunidade muda para o estágio 'Negotiation',
para que nenhuma negociação em andamento fique sem acompanhamento."
```

Isso dispara o ciclo completo: gera o spec, a arquitetura, implementa as classes Apex (aderentes ao base-framework), escreve os testes com cobertura de 100% e roda o review com auto-fix, parando apenas se algum gate falhar após esgotar as tentativas.

## Estrutura do repositório

```
.claude/skills/
├── apex-dev/       # orquestrador do pipeline
├── apex-spec/       # etapa 1 — spec
├── apex-arch/       # etapa 2 — arquitetura
├── apex-impl/       # etapa 3 — implementação
├── apex-test/       # etapa 4 — testes
├── apex-review/     # etapa 5 — review + auto-fix
└── apex-patterns/    # referência constitucional (regras + docs de referência)
    └── references/
```

## Licença

MIT — veja [LICENSE](LICENSE).
