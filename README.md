# AkosMed — Documentação Oficial do Backend

**Produto:** AkosMed  
**Repositório:** `gbsalermo/AkosMed-BackEnd`  
**Package base:** `br.com.akosmed`  
**Escopo atual:** Backend + banco de dados  
**Arquitetura:** Monólito modular  
**Identificador externo:** `UUID publicId` obrigatório  
**PK interna:** `Long id`  
**Concorrência crítica:** agendamento protegido contra double booking  
**Objetivo da documentação:** permitir implementar e revisar a API com segurança mesmo sem revisão direta dentro do repositório.

---

# Como trabalhar com esta documentação

Há dois documentos principais:

1. **`CONTINUIDADE.md`**  
   Estado real do projeto, decisões oficiais, etapa atual e próxima tarefa.

2. **`11_ROADMAP_ETAPAS.md`**  
   Ordem executável de desenvolvimento, dividida em etapas e subetapas.

Os demais documentos funcionam como **manual técnico e checklist de revisão**.

Quando houver dúvida entre dois arquivos:

> `CONTINUIDADE.md` + `11_ROADMAP_ETAPAS.md` têm prioridade para ordem de execução.

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
Entity
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
Atualizar CONTINUIDADE
  ↓
Commit
```

Não criar todas as entities antes de começar os Services.

---

# Documentos

| Arquivo | Função |
|---|---|
| `CONTINUIDADE.md` | Estado oficial, decisões e próxima tarefa |
| `01_VISAO_E_ESCOPO.md` | Visão do produto e limites |
| `02_ARQUITETURA.md` | Organização do backend |
| `03_MULTI_TENANCY_E_SEGURANCA.md` | Isolamento, autenticação e autorização |
| `04_ENTIDADES_E_RELACIONAMENTOS.md` | Modelo de domínio |
| `05_BANCO_DE_DADOS.md` | H2 no desenvolvimento e PostgreSQL no fechamento |
| `06_APIS_CRUDS_E_REGRAS.md` | Catálogo de endpoints e regras |
| `07_AGENDA_E_ATENDIMENTO.md` | Disponibilidade, bloqueios e fluxo de consulta |
| `08_PRONTUARIO_E_MODULOS_CLINICOS.md` | Prontuário e extensões |
| `09_MODULOS_OPCIONAIS.md` | O que fica para depois |
| `10_TESTES_E_QUALIDADE.md` | Estratégia de testes |
| `11_ROADMAP_ETAPAS.md` | Sequência executável |
| `12_IDEIAS_FUTURAS_E_NOMES.md` | Evoluções de produto |
| `13_APPS_ASSISTENTES_E_PRESCRICOES.md` | Akos Assistant, Kotlin e prescrições |
| `14_GUIA_IMPLEMENTACAO_PRATICA.md` | Como implementar uma feature |
| `15_PADROES_ENTIDADES_DTOS_REPOSITORIES.md` | Padrões concretos de Entity/DTO/Repository |
| `16_RELACIONAMENTOS_JPA_CASCADE_FETCH.md` | Relacionamentos JPA, cascade e fetch |
| `17_PADROES_HTTP_ERROS_PAGINACAO.md` | Contratos HTTP e respostas |
| `18_MAPA_METODOS_SERVICES.md` | Métodos esperados por módulo |
| `19_CHECKLIST_REVISAO_QUALIDADE.md` | Checklist para revisar cada etapa |
| `20_GUIA_USO_IA_PARA_REVISAO.md` | Como usar IA externa como segunda revisão |
| `21_PUBLIC_ID_E_CONCORRENCIA.md` | Regra oficial de publicId e controle de concorrência |
| `22_CONCORRENCIA_IDEMPOTENCIA_CLOCK_OPERACAO.md` | Optimistic lock, idempotência, Clock, correlation ID, health, backup e operação |

---

# Visão resumida do domínio

```text
Tenant
├── Unidade
├── Pessoa
│   ├── Paciente
│   └── ProfissionalSaude
├── UsuarioTenant
│   └── UsuarioUnidade
├── Especialidade
├── Procedimento
├── DisponibilidadeProfissional
├── BloqueioAgenda
├── Agendamento
│   └── EventoAgendamento
└── Prontuario
    └── Atendimento
        ├── EvolucaoClinica
        ├── Prescricao
        │   └── ItemPrescricao
        └── AnexoClinico
```

---

# Status atual

**Planejamento revisado e pronto para implementação.**

Próxima tarefa oficial:

> **ETAPA 0.1 — Gerar o projeto Spring Boot com H2 no repositório existente.**

---

# Convenções obrigatórias do produto

## Public ID

Toda entidade persistida do AkosMed usa:

```text
id Long        → somente banco/backend
publicId UUID  → API, DTOs, URLs, integrações
```

A API nunca deve devolver ou exigir o `Long id`.

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

Proteção:

```text
@Transactional
→ lock pessimista no profissional
→ revalidar disponibilidade
→ validação de overlap
→ insert
```

Na migração para PostgreSQL, avaliar/adicionar exclusion constraint como segunda barreira.

## Consistência e operação

O produto também adota:

```text
@Version seletivo
→ evita lost update em cadastros mutáveis

Clock centralizado
→ permite regras temporais determinísticas e testáveis

Idempotency-Key quando necessário
→ evita efeito duplicado por retry

X-Correlation-Id
→ rastreabilidade ponta a ponta

Health checks
Backup + restore testado
Secrets fora do Git
Limites de upload
Matriz de autorização por perfil
```

Esses mecanismos entram nas etapas corretas. Não devem virar infraestrutura genérica antes da necessidade.
