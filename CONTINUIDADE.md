# CONTINUIDADE — AkosMed Backend

> **Fonte oficial para retomar o projeto.**  
> Toda nova sessão humana ou com IA começa aqui.

**Última revisão documental geral:** 2026-08-31  
**Repositório:** `gbsalermo/AkosMed-BackEnd`  
**Branch oficial:** `main`

---

# 1. ESTADO REAL DO PROJETO

**Projeto:** AkosMed  
**Package base planejado:** `br.com.akosmed`  
**Arquitetura:** monólito modular por feature  
**Banco durante o Core:** H2  
**Banco definitivo:** PostgreSQL após o Core funcional  
**Swagger/OpenAPI:** depois do PostgreSQL  
**Postman oficial:** depois do Swagger

**Status:** 🟡 **DOCUMENTAÇÃO REVISADA / IMPLEMENTAÇÃO SPRING BOOT AINDA NÃO INICIADA**  
**Etapa atual:** `ETAPA 0 — Fundação H2`  
**Próxima tarefa exata:** **0.1 — Gerar Spring Boot Maven + Java 21 + H2 no repositório existente**

Na revisão de 2026-08-31, o repositório ainda não possui como implementação concluída:

```text
pom.xml
src/main/java
src/test/java
Entities JPA
Repositories
Services
Controllers
configuração H2
Spring Security/JWT
PostgreSQL
Flyway
Swagger
coleção Postman oficial
```

Portanto, outra IA **não deve confundir os contratos/documentos planejados com código já implementado**.

---

# 2. LEITURA OBRIGATÓRIA PARA RETOMAR

Ordem recomendada:

```text
1. CONTINUIDADE.md
2. 00_DOSSIE_PROJETO_AKOSMED.md
3. 11_ROADMAP_ETAPAS.md
4. documentos técnicos da subetapa atual
5. 19_CHECKLIST_REVISAO_QUALIDADE.md antes de concluir
```

Funções:

```text
CONTINUIDADE.md
→ estado real, decisões vigentes e próxima tarefa

00_DOSSIE_PROJETO_AKOSMED.md
→ contexto consolidado/handoff para outra IA ou pessoa

11_ROADMAP_ETAPAS.md
→ sequência executável e critérios de aceite

04_ENTIDADES_E_RELACIONAMENTOS.md
→ modelo canônico do domínio MVP

21_PUBLIC_ID_E_CONCORRENCIA.md
22_CONCORRENCIA_IDEMPOTENCIA_CLOCK_OPERACAO.md
→ guardrails transversais obrigatórios
```

Se houver divergência documental real, não escolher silenciosamente uma versão. Registrar e reconciliar a documentação antes de implementar a regra divergente.

---

# 3. ORDEM OFICIAL DO PRODUTO

```text
ETAPA 0  — Fundação H2
ETAPA 1  — Tenant + Unidade
ETAPA 2  — Identidade + Auth + multi-tenant
ETAPA 3  — Estrutura clínica base
ETAPA 4  — Disponibilidade + bloqueios
ETAPA 5  — Agendamento
ETAPA 6  — Prontuário + atendimento
ETAPA 7  — Prescrição + anexos + StorageService
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

Detalhamento completo: `11_ROADMAP_ETAPAS.md`.

---

# 4. PRINCÍPIO DE DESENVOLVIMENTO

Como nos projetos anteriores que inspiram o processo do AkosMed, trabalhar verticalmente:

```text
uma subetapa
↓
regra confirmada
↓
implementação completa da camada necessária
↓
testes executados
↓
checklist de qualidade
↓
atualizar CONTINUIDADE
↓
commit
↓
próxima subetapa
```

Não:

- desenvolver todos os CRUDs em paralelo;
- criar todas as Entities antecipadamente;
- criar infraestrutura futura apenas para “adiantar”;
- marcar `[x]` sem teste executado;
- reorganizar o roadmap a cada nova sessão.

---

# 5. DECISÕES OFICIAIS

## 5.1 Arquitetura

```text
Java 21
Spring Boot
Maven
REST
Spring Data JPA / Hibernate
monólito modular por feature
```

Estrutura planejada:

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

Evitar no MVP sem necessidade concreta:

- microsserviços;
- CQRS;
- event sourcing;
- mensageria;
- GenericRepository;
- BaseService/BaseController universais;
- arquitetura hexagonal formal em cada feature;
- abstrações criadas apenas por antecipação.

---

## 5.2 H2 primeiro

Etapas 0–10:

```text
Spring Boot
JPA/Hibernate
H2
JUnit/Mockito
Spring Boot Test
```

A intenção é estabilizar domínio, regras, relacionamentos e contratos antes da persistência definitiva.

Não adicionar no começo:

```text
PostgreSQL
Flyway
Swagger
Postman oficial
```

---

## 5.3 PostgreSQL depois

ETAPA 11:

```text
PostgreSQL Driver
Flyway
ddl-auto=validate
Testcontainers PostgreSQL
locks reais
constraints
migrations
```

Revalidar:

- publicId;
- multi-tenancy;
- concorrência;
- timezone;
- queries;
- paginação;
- índices;
- constraints;
- transações;
- optimistic locking.

---

## 5.4 Swagger e Postman

```text
ETAPA 12 → Swagger/OpenAPI
ETAPA 13 → Postman
```

A API será documentada formalmente depois de estabilizada no banco definitivo.

Os contratos planejados já ficam documentados durante o Core para manter consistência.

---

## 5.5 Public ID obrigatório

Toda Entity persistida usa:

```text
id       Long → PK/FK e uso interno
publicId UUID → API/DTO/URL/JWT quando aplicável/apps/integrações
```

Regras:

- `publicId` gerado pela aplicação;
- unique;
- imutável;
- CreateDTO não escolhe publicId;
- ResponseDTO não expõe `Long id`;
- relacionamento externo usa `...PublicId`;
- resource lookup tenant-scoped usa `publicId + tenantId interno`;
- UUID não substitui autorização.

Referência: `21_PUBLIC_ID_E_CONCORRENCIA.md`.

---

## 5.6 Multi-tenancy

Modelo:

```text
Shared Database
Shared Schema
tenant_id
```

O Tenant é determinado pelo contexto autenticado.

Não confiar em:

```text
tenantId vindo do request
```

Na fronteira de recurso tenant-scoped:

```text
findByPublicIdAndTenantId(...)
findAllByTenantId(...)
existsBy...AndTenantId(...)
```

`tenantId Long` usado internamente é resolvido pelo servidor.

Cross-tenant deve preferencialmente responder 404 para não revelar existência.

---

## 5.7 JWT e identidade externa

Token tenant-scoped deve preferir identificadores públicos:

```text
usuarioPublicId
tenantPublicId
usuarioTenantPublicId
perfilTenant
superAdmin=false
```

O filtro pode resolver os IDs internos e mantê-los apenas no contexto do servidor.

Não transformar `Long id` sequencial em contrato externo por conveniência.

Referência: `03_MULTI_TENANCY_E_SEGURANCA.md`.

---

## 5.8 Concorrência de agendamento

Double booking é requisito crítico.

Cenário:

```text
Paciente A → profissional X → 14:00
Paciente B → profissional X → 14:00
```

Resultado obrigatório:

```text
1 criação bem-sucedida
1 HTTP 409 AGENDAMENTO_CONFLITO
```

Fluxo:

```text
@Transactional
↓
PESSIMISTIC_WRITE no profissional
↓
revalidar disponibilidade
↓
validar bloqueios
↓
consultar overlap
↓
salvar
↓
registrar EventoAgendamento
↓
commit
```

Overlap:

```text
existing.inicio < novoFim
AND
existing.fim > novoInicio
```

Intervalo:

```text
[inicio, fim)
```

Logo, 14:00–14:30 e 14:30–15:00 podem coexistir.

Na ETAPA 11:

- duas transações reais em Testcontainers/PostgreSQL;
- confirmar que a segunda espera/revalida;
- repetir reagendamento;
- avaliar/adicionar exclusion constraint como segunda barreira.

Referência: `21_PUBLIC_ID_E_CONCORRENCIA.md`.

---

## 5.9 Optimistic locking

Para entidades administrativas mutáveis, avaliar:

```java
@Version
private Long version;
```

Objetivo:

```text
evitar lost update
```

Conflito esperado:

```text
409 RESOURCE_VERSION_CONFLICT
```

Não aplicar mecanicamente em histórico append-only:

```text
EventoAgendamento
EvolucaoClinica
AuditLog
```

---

## 5.10 Idempotência

Operações críticas sujeitas a retry devem ser avaliadas para:

```http
Idempotency-Key
```

Candidatas:

- criação de agendamento;
- emissão de prescrição;
- pagamentos/documentos futuros.

Comportamento quando adotado:

```text
mesma key + mesmo payload
→ sem efeito duplicado

mesma key + payload diferente
→ 409 IDEMPOTENCY_KEY_REUSED
```

Não criar infraestrutura genérica na ETAPA 0.

---

## 5.11 Clock centralizado

Services dependentes do tempo recebem:

```java
Clock
```

Evitar `now()` espalhado pelo domínio.

Testes temporais:

```java
Clock.fixed(...)
```

Regra:

```text
instantes persistidos → UTC
exibição/cálculo local → Tenant.timezone
```

---

## 5.12 Correlation ID

Toda request deve aceitar ou gerar:

```http
X-Correlation-Id
```

O mesmo valor acompanha:

```text
request
logs
ApiErrorDTO
AuditLog quando aplicável
```

Nunca logar:

- senha;
- JWT completo;
- refresh token;
- evolução clínica;
- prontuário completo;
- receita completa;
- anexos/mídias clínicas.

---

## 5.13 Segurança e autorização

Perfis tenant:

```text
ADMIN_TENANT
SECRETARIA
PROFISSIONAL
PACIENTE
AUDITOR
```

Na ETAPA 9 consolidar:

```text
recurso × perfil × ação
```

Toda operação sensível também considera Tenant, ownership/vínculo e acesso por Unidade quando aplicável.

Paciente autenticado em `/me` deve ser resolvido pelo backend; não recebe `pacientePublicId` para escolher os próprios dados.

---

## 5.14 Operação/produção

Antes do MVP de produção:

- health checks/Actuator;
- secrets fora do Git;
- limites de upload;
- MIME/extensões permitidas;
- logs/correlationId;
- backup automático;
- retenção;
- criptografia;
- procedimento de restore;
- teste real de restore;
- observabilidade mínima.

Backup sem teste de restore não é considerado concluído.

Referência: `22_CONCORRENCIA_IDEMPOTENCIA_CLOCK_OPERACAO.md`.

---

## 5.15 Storage

Arquivos clínicos grandes não ficam como BLOB no PostgreSQL por padrão.

Banco guarda metadata e referência:

```text
storageKey
mimeType
tamanho
hash
...
```

Abstração:

```text
StorageService
```

Desenvolvimento:

```text
filesystem local
```

Produção futura:

```text
S3/storage compatível
```

`storageKey` é interno e não sai no DTO.

Autorização ocorre antes de abrir/entregar o arquivo.

---

## 5.16 Materiais terapêuticos e vídeos do paciente

Recurso planejado **depois do Backend MVP 1.0**.

Caso inicial:

```text
Fisioterapeuta
→ paciente
→ vídeo demonstrando o exercício correto
→ instruções de execução
→ AkosMed Patient
```

Não modelar como `VideoFisioterapia`.

Usar conceito genérico:

```text
OrientacaoPaciente
```

Tipos previstos:

```text
VIDEO
DOCUMENTO
LINK
TEXTO
```

Regras principais:

- `Long id` interno + `UUID publicId` externo;
- Tenant obrigatório;
- paciente/profissional no mesmo Tenant;
- Atendimento opcional precisa ser compatível;
- paciente só vê orientações vinculadas a ele;
- profissional precisa de autorização;
- paciente não edita conteúdo enviado;
- mídia não fica no PostgreSQL;
- StorageService reutilizado;
- evitar URL pública permanente;
- preferir URL temporária/assinada;
- validar MIME/tamanho;
- criação/remoção auditada conforme matriz.

Posição:

```text
ETAPA 7  → estabilizar StorageService/AnexoClinico
ETAPA 14 → fechar Backend MVP 1.0
ETAPA 15 → API /me + OrientacaoPaciente
ETAPA 16 → app Kotlin + área de orientações/vídeos
```

Detalhes: `13_APPS_ASSISTENTES_E_PRESCRICOES.md`.

---

# 6. MODELO CLÍNICO PRINCIPAL — MVP

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

Também fazem parte do Core conforme etapa:

```text
Usuario
RefreshToken
Notificacao
AuditLog
```

Modelo canônico: `04_ENTIDADES_E_RELACIONAMENTOS.md`.

`OrientacaoPaciente` é pós-MVP e não deve ser antecipada durante as ETAPAS 0–14.

---

# 7. REGRAS DE IMPLEMENTAÇÃO

## Controller

- recebe DTO;
- usa `@Valid`;
- retorna DTO;
- paths usam UUID público;
- não consulta Repository;
- não contém regra de negócio;
- não decide Tenant.

## Service

- concentra regra de negócio;
- resolve Tenant/ownership;
- valida relacionamentos/status;
- é fronteira transacional;
- aplica lock/Clock/idempotência quando necessário;
- pode usar IDs Long apenas depois de resolver/autorizá-los internamente.

## Repository

- trabalha com Entity;
- query tenant-scoped;
- entrada externa resolvida por `publicId + tenantId interno`;
- não recebe DTO;
- `findById(Long)` apenas em fluxo interno controlado.

## DTO

- não expõe Entity;
- não expõe `Long id`;
- não recebe `tenantId` livre;
- relacionamentos externos usam `...PublicId`;
- não expõe `storageKey`, hash ou segredo.

## JPA

- LAZY por padrão;
- sem `ManyToMany` no Core;
- sem `CascadeType.ALL` por conveniência;
- sem hard delete de histórico clínico;
- bidirecionalidade somente quando necessária.

---

# 8. CHECKLIST DE UMA SUBETAPA

```text
[ ] regra confirmada
[ ] escopo da subetapa preservado
[ ] Entity/Enum quando aplicável
[ ] id Long interno + publicId UUID
[ ] @Version avaliado
[ ] DTOs sem Long id
[ ] Repository tenant-scoped
[ ] Service
[ ] Controller
[ ] validações/exceptions
[ ] Clock se houver regra temporal
[ ] concorrência se houver recurso disputado
[ ] idempotência avaliada se houver retry
[ ] testes happy path
[ ] testes de erro/conflito
[ ] teste cross-tenant quando aplicável
[ ] mvn test executado
[ ] checklist 19 executado
[ ] CONTINUIDADE atualizado
[ ] commit
```

---

# 9. DOCUMENTOS POR TIPO DE TRABALHO

## Handoff / estado

- `00_DOSSIE_PROJETO_AKOSMED.md`
- `CONTINUIDADE.md`
- `11_ROADMAP_ETAPAS.md`

## Entity/DTO/Repository

- `04_ENTIDADES_E_RELACIONAMENTOS.md`
- `15_PADROES_ENTIDADES_DTOS_REPOSITORIES.md`
- `16_RELACIONAMENTOS_JPA_CASCADE_FETCH.md`

## Controller/API

- `06_APIS_CRUDS_E_REGRAS.md`
- `17_PADROES_HTTP_ERROS_PAGINACAO.md`

## Service

- `18_MAPA_METODOS_SERVICES.md`

## Auth/multi-tenant

- `03_MULTI_TENANCY_E_SEGURANCA.md`

## Agenda

- `07_AGENDA_E_ATENDIMENTO.md`
- `21_PUBLIC_ID_E_CONCORRENCIA.md`

## Prontuário

- `08_PRONTUARIO_E_MODULOS_CLINICOS.md`

## Banco

- `05_BANCO_DE_DADOS.md`

## Testes

- `10_TESTES_E_QUALIDADE.md`
- `19_CHECKLIST_REVISAO_QUALIDADE.md`

## Concorrência/consistência/operação

- `21_PUBLIC_ID_E_CONCORRENCIA.md`
- `22_CONCORRENCIA_IDEMPOTENCIA_CLOCK_OPERACAO.md`

## Apps/orientações/prescrições futuras

- `13_APPS_ASSISTENTES_E_PRESCRICOES.md`

## Revisão com outra IA

- `20_GUIA_USO_IA_PARA_REVISAO.md`

---

# 10. ETAPA ATUAL

## ETAPA 0 — Fundação H2

- [x] repositório criado;
- [x] documentação oficial criada;
- [x] documentação geral revisada para handoff em 2026-08-31;
- [ ] **0.1 gerar Spring Boot Maven + Java 21 + dependências iniciais**;
- [ ] 0.2 configurar H2 e profiles;
- [ ] 0.3 criar packages conforme necessidade;
- [ ] 0.4 criar tratamento global de erros;
- [ ] 0.5 criar Clock + CorrelationIdFilter;
- [ ] 0.6 smoke tests / `mvn test`;
- [ ] validar aplicação iniciando.

A revisão documental não conclui 0.1.

---

# 11. PRÓXIMA TAREFA EXATA — 0.1

Gerar projeto Spring Boot Maven no repositório existente.

Dependências iniciais:

```text
Spring Web
Spring Data JPA
Bean Validation
H2 Database
Spring Boot Test
```

Configuração:

```text
Java 21
package br.com.akosmed
```

Não adicionar ainda:

```text
PostgreSQL Driver
Flyway
Spring Security/JWT
Springdoc/Swagger
mensageria
```

Critério de aceite:

```text
mvn test → BUILD SUCCESS
aplicação inicia
sem PostgreSQL
sem Swagger
sem entidades clínicas antecipadas
```

Depois:

1. atualizar este arquivo;
2. marcar somente os itens realmente executados no roadmap;
3. commit;
4. avançar para 0.2.

---

# 12. REVISÃO DOCUMENTAL DE 2026-08-31

Objetivo da revisão:

> permitir que outra IA compreenda projeto, decisões, estado e planejamento sem depender do histórico das conversas.

Ações realizadas:

- criado `00_DOSSIE_PROJETO_AKOSMED.md`;
- README transformado em porta de entrada de handoff;
- reforçada hierarquia entre CONTINUIDADE, dossiê e roadmap;
- corrigidos exemplos antigos que expunham `Long id` em DTO/URL;
- alinhadas queries públicas para `publicId + tenantId interno`;
- JWT alinhado a claims públicas;
- catálogo HTTP alinhado a UUID;
- mapa de Services alinhado a métodos por `publicId`;
- estratégia de testes alinhada a publicId/tenant/concorrência;
- guia de IA transformado em protocolo de continuidade;
- roadmap pós-MVP detalhado para `OrientacaoPaciente` e AkosMed Patient/Kotlin;
- preservadas as decisões H2 → PostgreSQL → Swagger → Postman;
- preservada a proteção obrigatória contra double booking.

Documentos revisados que já estavam coerentes foram mantidos sem alteração desnecessária.

### Estado após a revisão

```text
DOCUMENTAÇÃO → pronta para handoff
IMPLEMENTAÇÃO → ainda começa em 0.1
```

---

# 13. REGRA PARA QUALQUER IA QUE ASSUMIR O PROJETO

```text
NÃO criar novo roadmap
NÃO reorganizar etapas por preferência própria
NÃO assumir que planejamento = código implementado
NÃO expor Long id na API
NÃO aceitar tenantId livre do frontend
NÃO antecipar PostgreSQL/Swagger
NÃO antecipar OrientacaoPaciente durante o Core
NÃO remover proteção de concorrência para simplificar
NÃO marcar [x] sem teste executado
```

Sempre confirmar o estado real no repositório antes de trabalhar.
