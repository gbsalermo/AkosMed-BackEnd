# CONTINUIDADE — AkosMed Backend

> Fonte oficial para retomar o projeto.  
> Toda nova sessão começa aqui.

---

# 1. ESTADO ATUAL

**Projeto:** AkosMed  
**Repositório:** `gbsalermo/AkosMed-BackEnd`  
**Package:** `br.com.akosmed`  
**Arquitetura:** Monólito modular  
**Banco durante desenvolvimento:** H2  
**Banco definitivo:** PostgreSQL após o Core funcional  
**Swagger/OpenAPI:** após PostgreSQL  
**Postman:** após Swagger  
**Status:** 🟡 DOCUMENTAÇÃO CONCLUÍDA / IMPLEMENTAÇÃO SPRING NÃO INICIADA  
**Etapa atual:** ETAPA 0 — Fundação  
**Próxima tarefa:** **0.1 — Gerar Spring Boot + H2 no repositório existente**

---

# 2. ORDEM OFICIAL

```text
ETAPA 0  — Fundação com H2
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
ETAPA 16 — AkosMed Patient / Kotlin + área de orientações e vídeos
ETAPA 17 — Akos Assistant
ETAPA 18 — Especializações
ETAPA 19 — Módulos adicionais / integrações
```

Detalhes: `11_ROADMAP_ETAPAS.md`.

---

# 3. PRINCÍPIO DE DESENVOLVIMENTO

Como nos projetos SGL/RAScomp:

```text
uma subetapa
↓
implementação completa
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

Não desenvolver vários CRUDs em paralelo.

---

# 4. DECISÕES OFICIAIS

## 4.1 H2 primeiro

Etapas 0–10:

```text
Spring Boot
JPA/Hibernate
H2
JUnit/Mockito
```

Sem PostgreSQL, Flyway, Swagger ou coleção Postman oficial no início.

---

## 4.2 PostgreSQL depois

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

- multi-tenancy;
- concorrência;
- timezone;
- queries;
- índices;
- constraints.

---

## 4.3 Swagger e Postman

```text
ETAPA 12 → Swagger/OpenAPI
ETAPA 13 → Postman
```

A API será documentada depois de estabilizada no banco definitivo.

---

## 4.4 Public ID obrigatório

Toda Entity persistida exposta externamente usa:

```text
id       Long → PK/FK interna
publicId UUID → API/DTO/URL/integração
```

Regras:

- `publicId` imutável;
- unique;
- gerado pela aplicação;
- API nunca expõe `Long id`;
- relacionamentos em requests usam `...PublicId`;
- resource lookup público usa `publicId + tenant`.

Referência: `21_PUBLIC_ID_E_CONCORRENCIA.md`.

---

## 4.5 Multi-tenancy

Modelo:

```text
Shared Database
Shared Schema
tenant_id
```

O tenant vem do contexto autenticado.

Não confiar em `tenantId` do request.

Repositories tenant-scoped:

```text
findByPublicIdAndTenantId(...)
findAllByTenantId(...)
existsBy...AndTenantId(...)
```

---

## 4.6 Concorrência de agendamento

Double booking é regra crítica.

```text
Paciente A → profissional X → 14:00
Paciente B → profissional X → 14:00
```

Resultado obrigatório:

```text
1 sucesso
1 resposta 409 AGENDAMENTO_CONFLITO
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
EventoAgendamento
↓
commit
```

No PostgreSQL/Testcontainers, testar duas transações realmente concorrentes e avaliar/adicionar exclusion constraint.

Referência: `21_PUBLIC_ID_E_CONCORRENCIA.md`.

---

## 4.7 Optimistic locking

Para entidades administrativas mutáveis, avaliar:

```java
@Version
private Long version;
```

Objetivo:

```text
evitar lost update
```

Conflito:

```text
409 RESOURCE_VERSION_CONFLICT
```

Não aplicar por padrão a históricos append-only como:

```text
EventoAgendamento
EvolucaoClinica
AuditLog
```

---

## 4.8 Idempotência

Operações críticas sujeitas a retry devem ser avaliadas para:

```http
Idempotency-Key
```

Candidatas:

- criação de agendamento;
- emissão de prescrição;
- pagamentos/documentos futuros.

Não criar infraestrutura genérica antes da necessidade real.

---

## 4.9 Clock centralizado

Services dependentes do tempo devem receber:

```java
Clock
```

Evitar `now()` espalhado pelo domínio.

Testes temporais usam:

```java
Clock.fixed(...)
```

Regra:

```text
instantes persistidos → UTC
exibição/cálculo local → Tenant.timezone
```

---

## 4.10 Correlation ID

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

Nunca logar conteúdo clínico sensível, senha ou token completo.

---

## 4.11 Segurança e autorização

Perfis tenant:

```text
ADMIN_TENANT
SECRETARIA
PROFISSIONAL
PACIENTE
AUDITOR
```

Na ETAPA 9 consolidar matriz:

```text
recurso × perfil × ação
```

e testar acessos permitidos e negados.

---

## 4.12 Operação/produção

Antes do MVP de produção:

- health checks/Actuator;
- secrets fora do Git;
- limites de upload;
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

## 4.13 Materiais terapêuticos e vídeos do paciente

O AkosMed Patient terá, após o Backend MVP 1.0, suporte a materiais enviados pelo profissional ao paciente.

Caso inicial:

```text
Fisioterapeuta
→ paciente
→ vídeo demonstrando o exercício correto
→ instruções de execução
→ visualização no app Kotlin
```

Não modelar o recurso exclusivamente como `VideoFisioterapia`.

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

O vídeo é o primeiro caso de uso, mas o mesmo domínio pode atender orientações pós-operatórias, fonoaudiologia, enfermagem, nutrição e outros acompanhamentos.

Regras principais:

- `OrientacaoPaciente` usa `id Long` interno + `publicId UUID` externo;
- paciente, profissional e atendimento, quando informado, pertencem ao mesmo tenant;
- paciente só acessa orientações vinculadas a ele;
- profissional só envia material dentro do tenant autorizado;
- binário de vídeo não fica no PostgreSQL;
- banco guarda metadata e referência do arquivo;
- usar `StorageService` para filesystem local no desenvolvimento e storage compatível com S3 em produção futura;
- evitar URL pública permanente; preferir URL temporária/assinada após autorização;
- limitar tamanho e MIME types antes de liberar upload em produção;
- conteúdo enviado pelo profissional não é editável pelo paciente.

Posição:

```text
ETAPA 7  → estabilizar StorageService/AnexoClinico
ETAPA 15 → API `/me/orientacoes` + operações do profissional
ETAPA 16 → tela "Orientações do profissional" no app Kotlin
```

Detalhes: `13_APPS_ASSISTENTES_E_PRESCRICOES.md`.

---

# 5. MODELO CLÍNICO PRINCIPAL

```text
Tenant
├── Unidade
├── Pessoa
│   ├── Paciente
│   │   ├── OrientacaoPaciente
│   │   └── Prontuario
│   │       └── Atendimento
│   │           ├── EvolucaoClinica
│   │           ├── Prescricao
│   │           │   └── ItemPrescricao
│   │           └── AnexoClinico
│   └── ProfissionalSaude
├── UsuarioTenant
│   └── UsuarioUnidade
├── Especialidade
├── Procedimento
├── DisponibilidadeProfissional
├── BloqueioAgenda
└── Agendamento
    └── EventoAgendamento
```

Modelo canônico: `04_ENTIDADES_E_RELACIONAMENTOS.md`.

`OrientacaoPaciente` é evolução planejada pós-MVP e não deve ser antecipada durante as etapas do Core.

---

# 6. REGRAS DE IMPLEMENTAÇÃO

Controller:

- recebe/retorna DTO;
- usa `@Valid`;
- não consulta Repository;
- não contém regra de negócio.

Service:

- regra de negócio;
- `@Transactional`;
- tenant;
- ownership;
- status;
- relacionamentos;
- concorrência quando aplicável.

Repository:

- trabalha com Entity;
- query tenant-scoped;
- não recebe DTO.

DTO:

- não expõe Entity;
- não expõe `Long id`;
- usa publicId para referências externas.

JPA:

- LAZY por padrão;
- sem `ManyToMany` no Core;
- sem `CascadeType.ALL` por conveniência;
- sem hard delete de histórico clínico.

---

# 7. CHECKLIST DE UMA SUBETAPA

```text
[ ] regra confirmada
[ ] Entity/Enum
[ ] publicId
[ ] @Version avaliado
[ ] DTOs
[ ] Repository tenant-scoped
[ ] Service
[ ] Controller
[ ] validações/exceptions
[ ] Clock se houver tempo
[ ] concorrência se houver disputa
[ ] idempotência avaliada se houver retry
[ ] testes
[ ] cross-tenant
[ ] checklist 19 executado
[ ] CONTINUIDADE atualizado
[ ] commit
```

---

# 8. DOCUMENTOS DE USO DURANTE IMPLEMENTAÇÃO

## Antes de Entity/DTO/Repository

- `04_ENTIDADES_E_RELACIONAMENTOS.md`
- `15_PADROES_ENTIDADES_DTOS_REPOSITORIES.md`
- `16_RELACIONAMENTOS_JPA_CASCADE_FETCH.md`

## Antes de Controller/API

- `06_APIS_CRUDS_E_REGRAS.md`
- `17_PADROES_HTTP_ERROS_PAGINACAO.md`

## Antes de Service

- `18_MAPA_METODOS_SERVICES.md`

## Concorrência/publicId

- `21_PUBLIC_ID_E_CONCORRENCIA.md`

## Consistência/tempo/operação

- `22_CONCORRENCIA_IDEMPOTENCIA_CLOCK_OPERACAO.md`

## Apps, orientações, vídeos e prescrições

- `13_APPS_ASSISTENTES_E_PRESCRICOES.md`

## Antes de marcar concluído

- `19_CHECKLIST_REVISAO_QUALIDADE.md`

## Revisão com outra IA

- `20_GUIA_USO_IA_PARA_REVISAO.md`

---

# 9. ETAPA ATUAL

## ETAPA 0 — Fundação

- [x] Repositório criado.
- [x] Documentação oficial versionada.
- [ ] 0.1 Gerar Spring Boot Maven + Java 21.
- [ ] 0.2 Configurar H2 dev/test.
- [ ] 0.3 Criar packages iniciais conforme necessidade.
- [ ] 0.4 Criar tratamento global de erros.
- [ ] 0.5 Criar Clock + CorrelationIdFilter.
- [ ] 0.6 Smoke tests / `mvn test`.
- [ ] 0.7 Validar aplicação iniciando.

---

# 10. PRÓXIMA TAREFA EXATA

## 0.1 — Gerar Spring Boot + H2

Dependências iniciais:

- Spring Web;
- Spring Data JPA;
- Validation;
- H2 Database;
- Spring Boot Test.

Não adicionar ainda:

- PostgreSQL Driver;
- Flyway;
- Springdoc;
- JWT;
- mensageria.

Aceite:

```text
mvn test → BUILD SUCCESS
aplicação inicia
sem PostgreSQL
sem Swagger
sem entidades clínicas antecipadas
```

Depois atualizar este arquivo e avançar para 0.2.
