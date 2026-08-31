# AkosMed — Documentação Oficial do Backend

**Produto:** AkosMed  
**Repositório:** `gbsalermo/AkosMed-BackEnd`  
**Package base:** `br.com.akosmed`  
**Escopo atual:** Backend + banco de dados  
**Arquitetura:** Monólito modular  
**Identificador externo:** `UUID publicId` obrigatório  
**PK interna:** `Long id`  
**Concorrência crítica:** agendamento protegido contra double booking  
**Objetivo da documentação:** permitir implementar, revisar e retomar a API com segurança mesmo quando outra IA ou ferramenta não possuir o histórico do projeto.

---

# Comece por aqui

Para assumir o projeto em uma nova sessão, leia nesta ordem:

1. **`CONTINUIDADE.md`** — estado real, etapa atual, decisões vigentes e próxima tarefa;
2. **`00_DOSSIE_PROJETO_AKOSMED.md`** — visão consolidada para handoff humano/IA;
3. **`11_ROADMAP_ETAPAS.md`** — ordem executável de desenvolvimento;
4. documentos técnicos da subetapa atual;
5. **`19_CHECKLIST_REVISAO_QUALIDADE.md`** antes de marcar a subetapa como concluída.

Não crie um novo roadmap em uma nova sessão e não reorganize as etapas por preferência própria.

Quando houver dúvida entre documentos sobre **estado ou sequência**:

```text
CONTINUIDADE.md → estado/decisões atuais
11_ROADMAP_ETAPAS.md → ordem executável
```

Se existir divergência técnica real entre arquivos, não escolher silenciosamente uma versão: registrar/corrigir a divergência antes de implementar a regra.

---

# Estado atual

**Status em 2026-08-31:** documentação revisada; implementação Spring Boot ainda não iniciada.

O repositório ainda não possui como etapa concluída:

```text
pom.xml
src/main/java
src/test/java
CRUDs
configuração H2
PostgreSQL
Swagger
Postman oficial
```

Próxima tarefa oficial:

> **ETAPA 0.1 — Gerar o projeto Spring Boot Maven com Java 21 e H2 no repositório existente.**

Detalhes e aceite: `CONTINUIDADE.md` e `11_ROADMAP_ETAPAS.md`.

---

# Stack por fase

## Desenvolvimento do Core

```text
Java 21
Spring Boot
Spring Web
Spring Data JPA
Bean Validation
H2
JUnit / Mockito
Spring Boot Test
```

## Segurança

Adicionada quando entrar a etapa de autenticação:

```text
Spring Security
JWT
```

## Persistência definitiva

Somente após o backend funcional em H2:

```text
PostgreSQL
Flyway
Testcontainers PostgreSQL
```

## Documentação e validação final da API

Depois da migração para PostgreSQL:

```text
Swagger / OpenAPI
Postman
```

A ordem oficial é:

```text
H2
↓
Backend funcional
↓
PostgreSQL
↓
Swagger
↓
Postman
↓
Fechamento MVP
```

---

# Regra de desenvolvimento

Implementar **verticalmente**, uma subetapa por vez:

```text
Regra
↓
Entity / Enum
↓
DTOs
↓
Repository
↓
Service
↓
Controller
↓
Validações / Exceptions
↓
Testes automatizados
↓
Execução local
↓
Checklist de qualidade
↓
Atualizar CONTINUIDADE
↓
Commit
```

Não criar todas as Entities antes de começar os Services e não desenvolver vários CRUDs em paralelo sem concluir a subetapa aberta.

---

# Documentos

| Arquivo | Função |
|---|---|
| `00_DOSSIE_PROJETO_AKOSMED.md` | Handoff consolidado para outra IA/pessoa |
| `CONTINUIDADE.md` | Estado oficial, decisões e próxima tarefa |
| `01_VISAO_E_ESCOPO.md` | Visão do produto e limites |
| `02_ARQUITETURA.md` | Organização do backend |
| `03_MULTI_TENANCY_E_SEGURANCA.md` | Isolamento, autenticação e autorização |
| `04_ENTIDADES_E_RELACIONAMENTOS.md` | Modelo canônico do domínio MVP |
| `05_BANCO_DE_DADOS.md` | H2 no desenvolvimento e PostgreSQL no fechamento |
| `06_APIS_CRUDS_E_REGRAS.md` | Catálogo de endpoints e regras |
| `07_AGENDA_E_ATENDIMENTO.md` | Disponibilidade, bloqueios e fluxo de consulta |
| `08_PRONTUARIO_E_MODULOS_CLINICOS.md` | Prontuário e extensões |
| `09_MODULOS_OPCIONAIS.md` | O que fica para depois |
| `10_TESTES_E_QUALIDADE.md` | Estratégia de testes |
| `11_ROADMAP_ETAPAS.md` | Sequência executável |
| `12_IDEIAS_FUTURAS_E_NOMES.md` | Evoluções de produto |
| `13_APPS_ASSISTENTES_E_PRESCRICOES.md` | Akos Assistant, Kotlin, prescrições e orientações/vídeos |
| `14_GUIA_IMPLEMENTACAO_PRATICA.md` | Como implementar uma feature |
| `15_PADROES_ENTIDADES_DTOS_REPOSITORIES.md` | Padrões concretos de Entity/DTO/Repository |
| `16_RELACIONAMENTOS_JPA_CASCADE_FETCH.md` | Relacionamentos JPA, cascade e fetch |
| `17_PADROES_HTTP_ERROS_PAGINACAO.md` | Contratos HTTP e respostas |
| `18_MAPA_METODOS_SERVICES.md` | Métodos esperados por módulo |
| `19_CHECKLIST_REVISAO_QUALIDADE.md` | Checklist para revisar cada etapa |
| `20_GUIA_USO_IA_PARA_REVISAO.md` | Como outra IA deve revisar sem desviar do projeto |
| `21_PUBLIC_ID_E_CONCORRENCIA.md` | Regra oficial de publicId e controle de concorrência |
| `22_CONCORRENCIA_IDEMPOTENCIA_CLOCK_OPERACAO.md` | Optimistic lock, idempotência, Clock, correlation ID, health, backup e operação |

---

# Visão resumida do domínio

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
├── UsuarioTenant
│   └── UsuarioUnidade
├── Especialidade
├── Procedimento
├── DisponibilidadeProfissional
├── BloqueioAgenda
└── Agendamento
    └── EventoAgendamento
```

`OrientacaoPaciente` é uma evolução planejada **pós-MVP**, detalhada em `13_APPS_ASSISTENTES_E_PRESCRICOES.md`, e não deve ser antecipada durante o Core.

---

# Convenções obrigatórias do produto

## Public ID

Toda Entity persistida do AkosMed usa:

```text
id Long        → somente banco/backend
publicId UUID  → API, DTOs, URLs, integrações e apps
```

A API nunca deve devolver ou exigir o `Long id` como identificador de recurso.

Relacionamentos em requests usam `...PublicId`.

## Multi-tenancy

Modelo:

```text
Shared Database + Shared Schema + tenant_id
```

O tenant vem do contexto autenticado. Nunca confiar em `tenantId` livre do request.

## Concorrência de agendamento

Caso crítico:

```text
Paciente A tenta 14:00
Paciente B tenta 14:00
mesmo profissional
```

Resultado obrigatório:

```text
1 agendamento criado
1 resposta 409 AGENDAMENTO_CONFLITO
```

Proteção planejada:

```text
@Transactional
→ lock pessimista no profissional
→ revalidar disponibilidade
→ validar bloqueios/overlap
→ insert
```

Na migração para PostgreSQL, validar com Testcontainers e avaliar/adicionar exclusion constraint como segunda barreira.

## Consistência e operação

O produto também adota:

```text
@Version seletivo
Clock centralizado
Idempotency-Key quando necessário
X-Correlation-Id
health checks
backup + restore testado
secrets fora do Git
limites de upload
matriz de autorização por perfil
```

Esses mecanismos entram nas etapas corretas; não devem virar infraestrutura genérica antecipada.

---

# Evolução pós-MVP já decidida

Depois do Backend MVP 1.0:

```text
ETAPA 15 → API do paciente + orientações/materiais terapêuticos
ETAPA 16 → AkosMed Patient em Kotlin + orientações/vídeos
ETAPA 17 → Akos Assistant
```

O primeiro caso de uso de `OrientacaoPaciente` é permitir que um fisioterapeuta envie ao paciente um vídeo demonstrando a execução correta de um exercício, mantendo o domínio genérico para `VIDEO`, `DOCUMENTO`, `LINK` e `TEXTO`.

Detalhes: `13_APPS_ASSISTENTES_E_PRESCRICOES.md`.
