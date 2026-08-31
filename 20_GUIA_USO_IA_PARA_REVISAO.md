# 20 — Guia para Usar IA como Segunda Revisão e Continuidade

> Este documento existe para que uma IA externa consiga revisar ou continuar o AkosMed **sem inventar estado, arquitetura ou roadmap**.

A IA é ferramenta de revisão/implementação assistida. A fonte de verdade continua sendo:

```text
documentação oficial
+
código real do repositório
+
testes executados
+
comportamento observado
```

---

# 1. Antes de qualquer análise

Uma nova IA deve receber/ler, nesta ordem:

```text
1. CONTINUIDADE.md
2. 00_DOSSIE_PROJETO_AKOSMED.md
3. 11_ROADMAP_ETAPAS.md
4. documentos técnicos da subetapa atual
```

Nunca pedir apenas:

> “continue o projeto”

sem dar acesso a esses documentos.

A IA deve primeiro responder internamente a quatro perguntas:

```text
Qual é a etapa atual?
Qual é a próxima subetapa exata?
O que já foi realmente implementado/testado?
Quais documentos governam essa subetapa?
```

Se o repositório não possui o código de uma feature, a IA não pode tratá-la como concluída apenas porque existe documentação planejando-a.

---

# 2. Hierarquia documental

Para estado e decisões recentes:

```text
CONTINUIDADE.md
```

Para contexto consolidado/handoff:

```text
00_DOSSIE_PROJETO_AKOSMED.md
```

Para ordem de execução:

```text
11_ROADMAP_ETAPAS.md
```

Para domínio MVP:

```text
04_ENTIDADES_E_RELACIONAMENTOS.md
```

Para guardrails transversais:

```text
21_PUBLIC_ID_E_CONCORRENCIA.md
22_CONCORRENCIA_IDEMPOTENCIA_CLOCK_OPERACAO.md
```

Se houver divergência real entre documentos, a IA não deve escolher silenciosamente a versão que preferir. Deve apontar a inconsistência e reconciliar a documentação antes de implementar a regra divergente.

---

# 3. Decisões que uma IA não deve desfazer por padrão

O AkosMed já decidiu:

```text
Java 21 + Spring Boot + Maven
monólito modular por feature
H2 durante o Core
PostgreSQL somente na ETAPA 11
Swagger somente na ETAPA 12
Postman oficial somente na ETAPA 13
Long id somente interno
UUID publicId no contrato externo
Shared Database + Shared Schema + tenant_id
Controller sem regra/Repository
Service como fronteira de negócio/transação
LAZY por padrão
sem ManyToMany no Core
sem CascadeType.ALL por conveniência
sem hard delete de histórico clínico
Clock centralizado
concorrência de agendamento protegida
```

Sugestões diferentes podem ser discutidas, mas não aplicadas automaticamente como “melhoria”.

---

# 4. Estado atual conhecido na revisão documental de 2026-08-31

Na revisão de 2026-08-31:

```text
documentação existente
implementação Spring Boot ainda não iniciada
ETAPA 0 aberta
próxima tarefa = 0.1 Projeto base Spring Boot + H2
```

Uma IA futura deve confirmar o estado mais recente em `CONTINUIDADE.md` e no código antes de usar este trecho como status atual.

---

# 5. Arquivos mínimos para revisar um CRUD implementado

Enviar/disponibilizar:

```text
CONTINUIDADE.md
trecho atual do roadmap
Entity
Enums
CreateDTO
UpdateDTO, se existir
ResponseDTO/SummaryDTO
Repository
Service
Controller
Exceptions relacionadas
Testes
```

Também informar:

```text
tenant-scoped ou global
regras do domínio
relacionamentos esperados
estado/status da feature
```

Para uma feature temporal/concorrente, incluir também os documentos 21/22.

---

# 6. Prompt-base de revisão de CRUD

```text
Revise esta implementação do AkosMed contra a documentação oficial.

Antes de revisar:
1. leia CONTINUIDADE.md;
2. leia 00_DOSSIE_PROJETO_AKOSMED.md;
3. leia a subetapa correspondente em 11_ROADMAP_ETAPAS.md;
4. use os documentos técnicos indicados pela subetapa.

Contexto obrigatório:
- Java 21 + Spring Boot + JPA
- desenvolvimento do Core em H2 até a ETAPA 10
- PostgreSQL entra na ETAPA 11
- Swagger entra na ETAPA 12
- arquitetura multi-tenant
- tenantId nunca é confiado ao request
- toda Entity persistida usa Long id interno + UUID publicId externo
- API/DTO/URL não expõe Long id
- relacionamentos em requests usam ...PublicId
- Controller não contém regra de negócio nem acessa Repository
- Service concentra validações e @Transactional
- DTOs não expõem Entity diretamente
- Fetch LAZY por padrão
- não usar CascadeType.ALL sem justificativa
- regras temporais usam Clock

Verifique:
1. aderência à subetapa atual
2. publicId e contratos
3. relacionamentos JPA
4. DTOs
5. Repository
6. regras do Service
7. isolamento multi-tenant
8. autorização
9. status HTTP/erros
10. concorrência/lost update/idempotência quando aplicável
11. riscos de N+1
12. testes faltantes
13. segurança
14. qualquer funcionalidade antecipada fora do roadmap

Não reescreva tudo automaticamente.
Primeiro liste problemas por severidade:
CRÍTICO / IMPORTANTE / MELHORIA.
```

---

# 7. Revisão de Entity

Perguntar explicitamente:

```text
- possui id Long interno + publicId UUID?
- publicId é gerado pela aplicação, unique e imutável?
- Tenant está presente quando necessário?
- FK está no lado correto?
- precisa ser bidirecional?
- cascade é seguro?
- fetch está LAZY?
- constraint lógica está faltando?
- nullable está coerente?
- money/date types estão adequados?
- precisa de @Version?
- há risco de delete em cascata/histórico destruído?
```

Não aceitar Entity “mais simples” se a simplificação remover publicId ou isolamento tenant.

---

# 8. Revisão de DTO e contrato

Checar:

```text
CreateDTO não recebe Long id
CreateDTO não escolhe publicId
request não escolhe tenantId
relações usam ...PublicId
ResponseDTO não expõe Long id
SummaryDTO não expõe Long id
Entity não é serializada diretamente
paths usam UUID publicId
```

Exemplo correto:

```text
AgendamentoCreateDTO.profissionalPublicId
GET /api/v1/agendamentos/{publicId}
```

---

# 9. Revisão de Repository

Na fronteira tenant-scoped, procurar:

```text
findByPublicIdAndTenantId
findAllByTenantId
existsBy...AndTenantId
```

Sinal crítico:

```text
Controller/Service recebe Long id do cliente
→ repository.findById(id)
```

`findById(Long)` não é proibido internamente depois da resolução/autorização, mas não pode reintroduzir PK interna como contrato externo.

---

# 10. Revisão de Service

Perguntar:

```text
- Tenant é resolvido internamente?
- publicId é combinado com Tenant?
- relacionamentos pertencem ao mesmo Tenant?
- @Transactional está correto?
- transição de status está protegida?
- exception é adequada?
- método faz responsabilidades demais?
- existe race condition?
- precisa de PESSIMISTIC_WRITE?
- precisa de @Version?
- pode sofrer retry e precisar de Idempotency-Key?
- regra temporal usa Clock?
```

A IA deve diferenciar:

```text
PESSIMISTIC_WRITE → disputa por recurso
@Version          → lost update
Idempotency-Key   → retry duplicado
```

---

# 11. Revisão de agendamento

Sempre fornecer também:

```text
DisponibilidadeProfissional
BloqueioAgenda
ProfissionalProcedimento
Agendamento
EventoAgendamento
21_PUBLIC_ID_E_CONCORRENCIA.md
```

Pedir análise de:

- Tenant;
- publicId;
- Unidade;
- disponibilidade;
- duração;
- bloqueios;
- status que ocupam agenda;
- overlap;
- reagendamento;
- transação;
- lock;
- idempotência quando adotada.

Regra de overlap:

```text
existing.inicio < novoFim
AND
existing.fim > novoInicio
```

Concorrência definitiva só é declarada depois de PostgreSQL/Testcontainers na ETAPA 11.

Resultado obrigatório em corrida pelo mesmo slot:

```text
1 sucesso
1 AGENDAMENTO_CONFLITO
```

---

# 12. Revisão de prontuário

Pedir explicitamente:

```text
verifique risco de alteração ou destruição de histórico clínico
```

Checar:

- hard delete;
- cascade remove;
- orphanRemoval indevido;
- update destrutivo;
- retificação;
- autorização;
- cross-tenant;
- logs com conteúdo sensível.

A IA não deve “simplificar” removendo histórico/auditoria necessária.

---

# 13. Revisão de prescrição

Checar:

- nasce em RASCUNHO;
- item editável apenas em RASCUNHO;
- emissão imutabiliza o fluxo normal;
- cancelamento preserva histórico;
- paciente é derivado do Atendimento/Prontuário conforme modelo;
- profissional/atendimento pertencem ao mesmo Tenant;
- Response do paciente não expõe além do permitido;
- emissão precisa de idempotência quando a etapa decidir implementá-la.

---

# 14. Revisão de storage/anexos

Checar:

- binário fora do PostgreSQL;
- banco guarda metadata/storageKey;
- `storageKey` não sai no ResponseDTO;
- autorização ocorre antes de download;
- Tenant/ownership;
- nome original não vira chave física diretamente;
- tamanho/MIME;
- falha de storage e persistência não deixam estado inconsistente.

---

# 15. Orientações e vídeos do paciente

`OrientacaoPaciente` é recurso planejado **pós-MVP**.

Não aceitar sugestão de implementá-lo durante o Core só porque já existe especificação.

Posição oficial:

```text
ETAPA 7  → StorageService/AnexoClinico
ETAPA 14 → fecha Backend MVP 1.0
ETAPA 15 → API do paciente + OrientacaoPaciente
ETAPA 16 → app Kotlin + orientações/vídeos
```

Quando chegar à ETAPA 15, revisar:

- `id Long` interno + `publicId UUID` externo;
- Tenant;
- paciente/profissional;
- Atendimento opcional compatível;
- tipos VIDEO/DOCUMENTO/LINK/TEXTO;
- StorageService;
- URL privada/temporária;
- MIME/tamanho;
- paciente não altera material enviado;
- cross-tenant e ownership.

Referência: `13_APPS_ASSISTENTES_E_PRESCRICOES.md`.

---

# 16. Revisão de testes

Primeiro pedir:

```text
Liste casos faltantes sem gerar testes ainda.
Separe:
- happy path
- validation
- not found
- publicId
- conflict
- tenant isolation
- status transition
- security
- concurrency
- clock/time
- optimistic locking
- idempotency
```

Depois implementar apenas os testes compatíveis com a subetapa atual.

Não marcar `[x]` porque o arquivo de teste existe. O teste precisa ter sido executado.

---

# 17. Auditoria de etapa completa

Ao finalizar uma etapa, fornecer à IA:

```text
CONTINUIDADE.md
00_DOSSIE_PROJETO_AKOSMED.md
trecho da etapa do ROADMAP
lista de arquivos criados/alterados
principais Entities
Services
Controllers
Repositories
Testes
resultado do mvn test
```

Pedir:

```text
Compare implementação e documentação e identifique:
- item faltante
- item marcado concluído sem evidência
- uso indevido de Long id no contrato
- risco multi-tenant
- relação JPA problemática
- teste ausente
- endpoint divergente
- regra de concorrência faltante
- dívida técnica que deve impedir avanço
- código de etapa futura antecipado
```

---

# 18. IA assumindo uma nova sessão

Uma IA que vai **continuar**, e não apenas revisar, deve seguir:

```text
1. ler CONTINUIDADE
2. confirmar estado no repositório
3. ler 00_DOSSIE
4. abrir a subetapa no ROADMAP
5. abrir documentos técnicos associados
6. implementar somente a subetapa aberta
7. executar testes
8. executar checklist 19
9. atualizar CONTINUIDADE
10. commit
11. só então avançar
```

Não criar roadmap alternativo.

Não “adiantar tudo que for possível”.

Não marcar etapas futuras como parcialmente concluídas por causa de classes utilitárias antecipadas.

---

# 19. Não aceitar cegamente sugestões de IA

Desconfiar se a IA sugerir sem necessidade concreta:

- microsserviços cedo;
- Kafka para CRUD;
- Redis sem problema real;
- PostgreSQL antes da ETAPA 11;
- Swagger antes da ETAPA 12;
- `CascadeType.ALL` em tudo;
- `FetchType.EAGER` para mascarar lazy loading;
- GenericRepository;
- BaseService/BaseController universais;
- retornar Entity;
- Lombok `@Data` em Entity sem análise;
- MapStruct obrigatório;
- PK `Long` em URL “por simplicidade”;
- `tenantId` vindo do frontend;
- 10 interfaces extras sem necessidade;
- arquitetura hexagonal completa no MVP;
- IA clínica decidindo regras críticas.

Comparar sempre com os documentos oficiais.

---

# 20. Se a IA encontrar uma decisão melhor

Ela pode propor mudança, mas deve separar claramente:

```text
REGRA ATUAL DOCUMENTADA
vs
PROPOSTA DE ALTERAÇÃO
```

Antes de aplicar uma mudança estrutural:

1. explicar impacto;
2. apontar documentos afetados;
3. verificar compatibilidade com etapas já concluídas;
4. atualizar fontes de verdade se a decisão for aceita;
5. só então alterar implementação.

Nunca mudar arquitetura e deixar documentação antiga contradizendo o código.

---

# 21. Critério de confiança

IA é segunda revisão.

A fonte de verdade é:

```text
regra documentada
+
teste executado
+
comportamento real
```

Se IA e teste discordarem, investigar.

Se código e documentação discordarem, não assumir automaticamente que o código está certo.

Se duas IAs discordarem, voltar às invariantes do produto e aos testes.

---

# Prompt curto para retomar o projeto

```text
Vamos continuar o AkosMed usando o repositório oficial.

Antes de alterar código:
1. leia CONTINUIDADE.md;
2. leia 00_DOSSIE_PROJETO_AKOSMED.md;
3. leia 11_ROADMAP_ETAPAS.md;
4. confirme no repositório se o estado documentado bate com o código.

Não crie novo roadmap e não reorganize etapas.
Implemente apenas a próxima subetapa oficial.
Respeite publicId, multi-tenancy, concorrência, Clock e demais guardrails documentados.
Execute os testes e o checklist 19 antes de marcar a subetapa concluída.
Atualize CONTINUIDADE.md ao final com o estado real.
```
