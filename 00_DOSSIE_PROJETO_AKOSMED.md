# 00 — Dossiê do Projeto AkosMed

> Documento de handoff para humanos e IAs que precisem assumir o projeto sem acesso ao histórico das conversas.
>
> **Última revisão documental:** 2026-08-31  
> **Repositório:** `gbsalermo/AkosMed-BackEnd`  
> **Branch oficial:** `main`

---

# 1. O que é o AkosMed

AkosMed é um SaaS backend para clínicas e consultórios, com foco inicial na operação ambulatorial diária.

O produto nasce como um **monólito modular Spring Boot**, multi-tenant e multi-unidade, preparado para atender desde consultórios individuais até clínicas multidisciplinares.

O Core deve cobrir:

- organização por Tenant e Unidade;
- autenticação e autorização;
- usuários, pessoas, pacientes e profissionais;
- especialidades e procedimentos;
- disponibilidade e bloqueios de agenda;
- agendamento com proteção contra double booking;
- check-in e atendimento;
- prontuário e evolução clínica;
- prescrição estruturada;
- anexos;
- agenda diária e notificações internas;
- auditoria e hardening;
- documentação e testes.

O produto **não começa** como sistema hospitalar completo.

---

# 2. Estado real do repositório

Na revisão de 2026-08-31, o repositório contém a documentação oficial do projeto e **a implementação Spring Boot ainda não foi iniciada**.

Não existem ainda como parte concluída do projeto:

```text
pom.xml
src/main/java
src/test/java
entidades JPA implementadas
Controllers
Services
Repositories
configuração H2
PostgreSQL
Swagger
coleção Postman oficial
```

Portanto, não assumir que CRUDs descritos nos documentos já existem.

**Etapa atual:** `ETAPA 0 — Fundação`  
**Próxima subetapa oficial:** `0.1 — Gerar Spring Boot Maven + Java 21 + H2`.

Fonte de estado: `CONTINUIDADE.md`.

---

# 3. Ordem de leitura para retomar o projeto

Uma nova IA ou pessoa deve ler nesta ordem:

```text
1. CONTINUIDADE.md
2. 00_DOSSIE_PROJETO_AKOSMED.md
3. 11_ROADMAP_ETAPAS.md
4. documentos técnicos da subetapa atual
5. 19_CHECKLIST_REVISAO_QUALIDADE.md antes de concluir
```

Documentos técnicos principais:

```text
04 → modelo de domínio MVP
06 → catálogo de APIs e regras
15 → Entity / DTO / Repository
16 → relacionamentos JPA
17 → HTTP / erros / paginação
18 → mapa de Services
21 → publicId + concorrência de agenda
22 → @Version + idempotência + Clock + operação
13 → apps, orientações, vídeos e prescrições futuras
20 → como usar outra IA para revisão
```

---

# 4. Hierarquia de autoridade documental

Os arquivos possuem responsabilidades diferentes.

## Estado e próxima tarefa

```text
CONTINUIDADE.md
```

É a fonte oficial para:

- onde o projeto está;
- qual subetapa está aberta;
- decisões recentes;
- o que já foi validado.

## Ordem de desenvolvimento

```text
11_ROADMAP_ETAPAS.md
```

É a fonte oficial para a sequência de etapas.

## Modelo de domínio do MVP

```text
04_ENTIDADES_E_RELACIONAMENTOS.md
```

É o modelo canônico do Core/MVP.

## Guardrails transversais obrigatórios

```text
21_PUBLIC_ID_E_CONCORRENCIA.md
22_CONCORRENCIA_IDEMPOTENCIA_CLOCK_OPERACAO.md
```

Esses documentos não são opcionais quando a feature toca seus temas.

## Evoluções pós-MVP

```text
13_APPS_ASSISTENTES_E_PRESCRICOES.md
12_IDEIAS_FUTURAS_E_NOMES.md
09_MODULOS_OPCIONAIS.md
```

Não antecipar itens desses arquivos durante o Core apenas porque já estão planejados.

Se houver uma divergência real entre documentos, **não escolher silenciosamente uma versão**. A decisão explícita mais recente registrada em `CONTINUIDADE.md` prevalece temporariamente e a documentação deve ser reconciliada antes de implementar a regra divergente.

---

# 5. Decisões técnicas já tomadas

Estas decisões não devem ser rediscutidas a cada sessão sem motivo concreto.

## Arquitetura

```text
Java 21
Spring Boot
Maven
monólito modular por feature
REST
JPA/Hibernate
```

Evitar arquitetura excessiva no MVP:

- sem microsserviços;
- sem CQRS;
- sem event sourcing;
- sem mensageria por padrão;
- sem GenericRepository/BaseService/BaseController genéricos;
- sem DDD/hexagonal formal aplicado mecanicamente em tudo.

## Banco por fase

Ordem obrigatória:

```text
H2 durante Core
↓
revisão funcional completa
↓
PostgreSQL + Flyway + Testcontainers
↓
Swagger / OpenAPI
↓
Postman
↓
fechamento Backend MVP 1.0
```

Não antecipar PostgreSQL para as primeiras etapas.

Não antecipar Swagger antes da migração para PostgreSQL.

## Multi-tenancy

Modelo:

```text
Shared Database
Shared Schema
tenant_id
```

O tenant é resolvido pelo backend a partir do contexto autenticado.

Nunca confiar em `tenantId` recebido livremente do cliente.

## Identificadores

Toda Entity persistida usa:

```text
Long id        → PK/FK e uso interno
UUID publicId  → API, DTO, URL, JWT quando aplicável, apps e integrações
```

A API não expõe nem aceita PK `Long` como identificador de recurso.

Requests relacionais usam nomes como:

```text
pacientePublicId
profissionalPublicId
unidadePublicId
procedimentoPublicId
```

## Concorrência de agendamento

Double booking é requisito de produto, não melhoria futura.

Cenário obrigatório:

```text
Paciente A → profissional X → 14:00
Paciente B → profissional X → 14:00
```

Resultado:

```text
1 criação bem-sucedida
1 HTTP 409 AGENDAMENTO_CONFLITO
```

Estratégia:

```text
@Transactional
→ PESSIMISTIC_WRITE no profissional
→ revalidar disponibilidade
→ validar bloqueios
→ validar overlap
→ persistir
→ registrar EventoAgendamento
```

Na ETAPA 11, repetir o cenário com PostgreSQL/Testcontainers e avaliar/adicionar exclusion constraint como segunda barreira.

## Lost update

Usar `@Version` seletivamente nas entidades mutáveis em que duas alterações concorrentes possam sobrescrever dados.

Não aplicar mecanicamente em histórico append-only.

## Tempo

Services dependentes do tempo recebem `Clock`.

```text
persistência de instantes → UTC
exibição/cálculo local → Tenant.timezone
```

Evitar `now()` espalhado.

## Idempotência

Avaliar `Idempotency-Key` em operações críticas sujeitas a retry, principalmente:

- criação de agendamento;
- emissão de prescrição;
- operações financeiras/documentais futuras.

Não construir infraestrutura genérica antes de existir uma operação que precise dela.

## Rastreabilidade

Toda request deve aceitar ou gerar:

```text
X-Correlation-Id
```

Logs não devem conter senha, token completo ou conteúdo clínico sensível.

---

# 6. Organização do código planejada

Package base:

```text
br.com.akosmed
```

Estrutura por feature:

```text
br.com.akosmed
├── shared
├── tenant
├── identity
├── professional
├── patient
├── scheduling
├── clinical
├── prescription
├── notification
└── audit
```

Dentro de cada feature, conforme necessidade:

```text
controller
dto
entity
repository
service
```

Criar packages/classes somente quando a etapa precisar deles. Não preencher o projeto com classes vazias para “adiantar estrutura”.

---

# 7. Modelo de domínio principal

Resumo do Core:

```text
Tenant
├── Unidade
├── Pessoa
│   ├── Paciente
│   │   └── Prontuario
│   │       └── Atendimento
│   │           ├── EvolucaoClinica
│   │           ├── Prescricao
│   │           │   └── ItemPrescricao
│   │           └── AnexoClinico
│   └── ProfissionalSaude
│       ├── ProfissionalEspecialidade
│       ├── ProfissionalUnidade
│       ├── ProfissionalProcedimento
│       └── DisponibilidadeProfissional
├── UsuarioTenant
│   └── UsuarioUnidade
├── Especialidade
├── Procedimento
├── BloqueioAgenda
└── Agendamento
    └── EventoAgendamento
```

Outras entidades do Core incluem `Usuario`, `RefreshToken`, `Notificacao` e `AuditLog`.

Detalhes e constraints: `04_ENTIDADES_E_RELACIONAMENTOS.md`.

---

# 8. Regras JPA e contratos

## JPA

- relacionamentos to-one LAZY por padrão;
- evitar `ManyToMany`; usar entidade de vínculo;
- não usar `CascadeType.ALL` por conveniência;
- histórico clínico não sofre hard delete/cascade destrutivo;
- Entity não é serializada diretamente como contrato REST.

## Controller

- recebe/retorna DTO;
- usa `@Valid`;
- não acessa Repository;
- não contém regra de negócio.

## Service

- concentra regras;
- resolve tenant e ownership;
- valida relacionamentos e status;
- define fronteira transacional;
- aplica lock/Clock/idempotência quando necessário.

## Repository

Na fronteira de recurso tenant-scoped:

```text
findByPublicIdAndTenantId(...)
findAllByTenantId(...)
existsBy...AndTenantId(...)
```

`findById(Long)` só pode ser usado internamente quando o recurso já estiver resolvido/autorizado e não representar entrada externa.

---

# 9. Roadmap resumido

```text
ETAPA 0  — Fundação H2
ETAPA 1  — Tenant + Unidade
ETAPA 2  — Identidade + Auth + multi-tenant
ETAPA 3  — Estrutura clínica base
ETAPA 4  — Disponibilidade + bloqueios
ETAPA 5  — Agendamento
ETAPA 6  — Prontuário + atendimento
ETAPA 7  — Prescrição + anexos
ETAPA 8  — Operação diária + notificações
ETAPA 9  — Auditoria + segurança
ETAPA 10 — Revisão funcional completa em H2
ETAPA 11 — PostgreSQL + Flyway + Testcontainers
ETAPA 12 — Swagger / OpenAPI
ETAPA 13 — Postman + validação ponta a ponta
ETAPA 14 — Fechamento Backend MVP 1.0
ETAPA 15 — API do paciente + orientações/materiais terapêuticos
ETAPA 16 — AkosMed Patient / Kotlin + orientações e vídeos
ETAPA 17 — Akos Assistant
ETAPA 18 — Especializações
ETAPA 19 — Módulos adicionais / integrações
```

Subetapas e critérios de aceite: `11_ROADMAP_ETAPAS.md`.

---

# 10. Evolução pós-MVP já planejada

## AkosMed Patient

App Android/Kotlin.

Primeiras funções:

- login;
- próximas consultas;
- detalhes;
- prescrições;
- notificações.

Depois:

- autoagendamento;
- documentos;
- orientações do profissional;
- vídeos e materiais terapêuticos.

## OrientacaoPaciente

Conceito genérico planejado para a fase pós-MVP.

Primeiro caso de uso:

```text
fisioterapeuta envia vídeo demonstrando exercício correto ao paciente
```

Tipos planejados:

```text
VIDEO
DOCUMENTO
LINK
TEXTO
```

O binário não fica no PostgreSQL. O banco guarda metadata e `storageKey`; acesso deve ser autorizado e, em produção, preferencialmente por URL temporária/assinada.

**Não implementar durante o Core.** A API entra na ETAPA 15 e a interface Kotlin na ETAPA 16.

## Akos Assistant

Deve reutilizar Services/endpoints existentes para:

- agenda;
- próximos pacientes;
- check-in;
- alertas;
- pendências;
- resumo diário.

Bot/app é adapter; regra de negócio permanece no backend.

---

# 11. Próxima tarefa exata

## ETAPA 0.1 — Projeto base

Criar Spring Boot Maven com:

```text
Java 21
Spring Web
Spring Data JPA
Bean Validation
H2
Spring Boot Test
```

Não adicionar ainda:

```text
PostgreSQL Driver
Flyway
Springdoc/Swagger
JWT/Spring Security
mensageria
```

Critério de aceite:

```text
mvn test → BUILD SUCCESS
aplicação inicia
package base br.com.akosmed
sem PostgreSQL
sem Swagger
sem entidades clínicas antecipadas
```

Depois:

1. atualizar `CONTINUIDADE.md`;
2. marcar apenas os itens realmente testados em `11_ROADMAP_ETAPAS.md`;
3. commit;
4. avançar para 0.2.

---

# 12. Protocolo obrigatório para outra IA

Ao assumir o projeto:

1. não criar um novo roadmap;
2. não reorganizar etapas por preferência própria;
3. não marcar tarefa como concluída sem evidência/teste;
4. não avançar uma subetapa não validada;
5. não implementar funcionalidades futuras só porque já estão documentadas;
6. não substituir `publicId` por `Long id` na API;
7. não mover PostgreSQL/Swagger para o início;
8. não remover guardrails de tenant/concorrência para simplificar;
9. antes de implementar, ler os documentos técnicos da subetapa;
10. depois de implementar, executar `19_CHECKLIST_REVISAO_QUALIDADE.md`;
11. atualizar `CONTINUIDADE.md` com estado real, testes executados e próxima tarefa;
12. se documentação e código divergirem, registrar a divergência e corrigir a fonte de verdade antes de seguir.

---

# 13. Armadilhas que devem ser detectadas em revisão

Considere erro de arquitetura/contrato se aparecer:

```text
ResponseDTO expondo Long id
@PathVariable Long id em recurso público
DTO recebendo tenantId livre
JWT usando IDs sequenciais como contrato externo sem justificativa
Controller chamando Repository
Entity sendo retornada diretamente
FetchType.EAGER para mascarar problema de sessão
CascadeType.ALL por conveniência
ManyToMany nos vínculos profissionais
hard delete de histórico clínico
Agendamento sem revalidação dentro da transação
checagem de overlap isolada sem proteção concorrente
LocalDateTime.now()/Instant.now() espalhado em Services
PostgreSQL/Flyway antes da ETAPA 11
Swagger antes da ETAPA 12
OrientacaoPaciente implementada durante o Core
```

---

# 14. Definition of Done documental

Uma subetapa só pode ser considerada concluída quando houver:

```text
regra implementada
+
testes executados e verdes
+
isolamento tenant revisado quando aplicável
+
publicId revisado
+
relacionamentos/JPA revisados
+
concorrência/Clock/idempotência avaliados quando aplicável
+
checklist 19 executado
+
CONTINUIDADE atualizado
+
commit
```

Este dossiê descreve o projeto; ele **não substitui** os detalhes técnicos dos arquivos especializados nem o estado real registrado em `CONTINUIDADE.md`.
