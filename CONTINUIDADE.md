# CONTINUIDADE — AkosMed Backend

> Fonte oficial para retomar o projeto.  
> Toda nova sessão começa aqui.

---

# 1. ESTADO ATUAL

**Projeto:** AkosMed  
**Repositório:** `akosmed-backend`  
**Package:** `br.com.akosmed`  
**Arquitetura:** Monólito modular  
**Banco durante desenvolvimento:** H2  
**Banco definitivo:** PostgreSQL — somente após o Core funcional  
**Swagger/OpenAPI:** após PostgreSQL  
**Postman:** após Swagger  
**Status geral:** 🟡 DOCUMENTAÇÃO NO REPOSITÓRIO / IMPLEMENTAÇÃO SPRING NÃO INICIADA  
**Etapa atual:** ETAPA 0 — Fundação  
**Próxima tarefa:** **0.1 — Gerar projeto Spring Boot + H2 no repositório existente**

---

# 2. PRINCÍPIO DE DESENVOLVIMENTO

O AkosMed será construído como nos projetos SGL/RAScomp:

```text
uma subetapa
↓
implementação completa
↓
teste
↓
revisão
↓
registro no CONTINUIDADE
↓
commit
↓
próxima subetapa
```

Não desenvolver grandes blocos paralelos.

---

# 3. ORDEM OFICIAL

```text
ETAPA 0  — Fundação com H2
ETAPA 1  — Tenant + Unidade
ETAPA 2  — Identidade + Auth + isolamento multi-tenant
ETAPA 3  — Estrutura clínica base
ETAPA 4  — Disponibilidade + bloqueios
ETAPA 5  — Agendamento
ETAPA 6  — Prontuário + atendimento
ETAPA 7  — Prescrição + anexos
ETAPA 8  — Operação diária + notificações
ETAPA 9  — Auditoria + segurança
ETAPA 10 — Revisão funcional completa em H2
ETAPA 11 — PostgreSQL + Flyway + validação real
ETAPA 12 — Swagger / OpenAPI
ETAPA 13 — Postman + validação ponta a ponta
ETAPA 14 — Fechamento Backend MVP 1.0
ETAPA 15 — API do paciente
ETAPA 16 — AkosMed Patient / Kotlin
ETAPA 17 — Akos Assistant
ETAPA 18 — Especializações
ETAPA 19 — Módulos adicionais / integrações
```

Detalhes em `11_ROADMAP_ETAPAS.md`.

---

# 4. DECISÕES OFICIAIS

## 4.1 Arquitetura

Usar **monólito modular**.

Não implementar inicialmente:

- microsserviços;
- Kafka;
- RabbitMQ;
- Redis;
- Kubernetes;
- CQRS;
- event sourcing;
- generic repository;
- arquitetura distribuída.

---

## 4.2 H2 primeiro

Durante as etapas 0–10:

```text
Spring Boot
↓
JPA/Hibernate
↓
H2
```

Configuração sugerida:

### dev

```text
ddl-auto=update
```

### test

```text
ddl-auto=create-drop
```

Sem Flyway inicialmente.

---

## 4.3 PostgreSQL depois

A migração para PostgreSQL ocorre somente na ETAPA 11.

Nesta etapa:

- revisar modelagem;
- criar migrations;
- adicionar PostgreSQL Driver;
- adicionar Flyway;
- trocar `ddl-auto` para `validate`;
- executar bateria de testes;
- revalidar locks;
- revalidar multi-tenancy;
- revalidar constraints;
- usar Testcontainers.

---

## 4.4 Swagger depois do banco

Springdoc/OpenAPI será adicionado apenas na ETAPA 12.

Motivo:

documentar a API depois de:

- regras estabilizadas;
- entidades estabilizadas;
- banco definitivo validado.

---

## 4.5 Postman depois do Swagger

A coleção oficial Postman será criada na ETAPA 13.

Durante o desenvolvimento, a validação principal será:

- testes automatizados;
- execução local;
- chamadas manuais pontuais quando necessário.

Postman será o fechamento funcional da API.

---

## 4.6 Multi-tenancy

Modelo:

```text
Shared Database
Shared Schema
tenant_id
```

Mesmo enquanto estivermos no H2, toda entidade tenant-scoped já deve respeitar o isolamento.

Repositories explícitos:

```text
findByIdAndTenantId(...)
findAllByTenantId(...)
existsBy...AndTenantId(...)
```

Não confiar em `tenantId` vindo do body da requisição.

---

## 4.7 Identidade

`Usuario` é global.

`Pessoa` é tenant-scoped.

```text
Usuario
   ↓
UsuarioTenant
   ├── Tenant
   └── Pessoa
```

Isso permite que a mesma credencial participe de múltiplas organizações sem misturar seus dados de domínio.

Privilégio global:

```text
Usuario.superAdmin
```

Perfis dentro de Tenant:

```text
ADMIN_TENANT
SECRETARIA
PROFISSIONAL
PACIENTE
AUDITOR
```

---

## 4.8 Agenda simplificada

Não criar entidade `Agenda` no MVP.

Usar:

```text
DisponibilidadeProfissional
BloqueioAgenda
Agendamento
```

Horários disponíveis são calculados.

Não persistir slots futuros.

---

## 4.9 IDs

Usar `Long` no MVP.

Facilita:

- debugging;
- testes;
- H2;
- Postman;
- JPA.

`UUID/publicId` pode entrar no futuro se houver necessidade externa.

---

## 4.10 Prontuário

```text
Paciente
  1
  ↓
Prontuario
  ↓
Atendimento
  ├── EvolucaoClinica
  ├── Prescricao
  └── AnexoClinico
```

Sem hard delete do histórico clínico.

---

## 4.11 Prescrição

Não criar catálogo farmacológico completo no MVP.

Cada `ItemPrescricao` armazena os dados necessários da orientação prescrita.

Campos principais:

```text
nomeMedicamento
concentracao
formaFarmaceutica
dose
unidadeDose
viaAdministracao
frequencia
vezesAoDia
intervaloHoras
duracaoDias
dataInicio
dataFim
instrucoes
usoContinuo
seNecessario
```

---

# 5. PADRÃO DE IMPLEMENTAÇÃO DE UMA SUBETAPA

Para uma feature normal:

```text
[ ] confirmar regra no roadmap
[ ] modelar relacionamento
[ ] criar Entity
[ ] criar Enums
[ ] criar DTO Create
[ ] criar DTO Update se necessário
[ ] criar DTO Response
[ ] criar Repository
[ ] criar Service
[ ] criar Controller
[ ] criar validações
[ ] criar exceptions
[ ] criar testes
[ ] executar testes
[ ] revisar tenant isolation
[ ] revisar relacionamentos/cascade/fetch
[ ] atualizar CONTINUIDADE
[ ] commit
```

Swagger e Postman **não fazem parte desse checklist durante o Core**.

---

# 6. PADRÕES DE CAMADA

## Controller

Pode:

- receber DTO;
- usar `@Valid`;
- chamar Service;
- escolher status HTTP.

Não pode:

- consultar Repository;
- decidir regra de negócio;
- validar conflito de agenda;
- escolher tenant manualmente.

---

## Service

Responsável por:

- regra de negócio;
- transação;
- carregar relacionamentos;
- validar ownership;
- validar tenant;
- validar status;
- salvar;
- retornar DTO.

Escrita:

```text
@Transactional
```

Leitura:

```text
@Transactional(readOnly = true)
```

---

## Repository

Trabalha com Entity.

Não recebe DTO.

Não contém regra de negócio.

Consultas tenant-scoped precisam do `tenantId` quando aplicável.

---

## DTO

Não expor Entity diretamente.

Separar:

```text
CreateDTO
UpdateDTO
ResponseDTO
SummaryDTO
```

quando necessário.

---

# 7. ENTIDADES DO MVP

## Organização

- Tenant
- Unidade

## Identidade

- Pessoa
- Usuario
- UsuarioTenant
- UsuarioUnidade
- RefreshToken

## Clínica

- Especialidade
- Procedimento
- ProfissionalSaude
- ProfissionalEspecialidade
- ProfissionalUnidade
- ProfissionalProcedimento
- Paciente

## Agenda

- DisponibilidadeProfissional
- BloqueioAgenda
- Agendamento
- EventoAgendamento

## Prontuário

- Prontuario
- Atendimento
- EvolucaoClinica

## Prescrição e arquivos

- Prescricao
- ItemPrescricao
- AnexoClinico

## Operação

- Notificacao
- AuditLog

---

# 8. ITENS ADIADOS

Não implementar agora:

- Perfil/Permissão totalmente dinâmicos;
- PacienteUnidade;
- catálogo farmacológico;
- CID estruturado;
- financeiro;
- convênios;
- estoque;
- laboratório;
- TISS;
- RNDS;
- telemedicina;
- assinatura digital;
- billing SaaS;
- Dental;
- Psychology;
- Vision;
- app Kotlin;
- bot externo;
- IA clínica.

---

# 9. REGRAS CRÍTICAS

## Multi-tenant

Tenant A nunca acessa recurso do Tenant B.

Testar explicitamente.

---

## Relacionamentos

Não usar `CascadeType.ALL` por padrão.

Não usar `EAGER` por conveniência.

Consultar `16_RELACIONAMENTOS_JPA_CASCADE_FETCH.md`.

---

## Agenda

Impedir overlap de profissional.

No H2:

- implementar regra;
- criar testes de conflito.

No PostgreSQL:

- revalidar concorrência real;
- revalidar lock.

---

## Status

Evitar alteração livre:

```text
PUT status
```

Usar ações:

```text
/confirmar
/check-in
/cancelar
/reagendar
/marcar-falta
```

---

## Prontuário

Não atualizar registro clínico histórico silenciosamente.

Retificação cria novo registro/vínculo.

---

# 10. DOCUMENTOS PARA REVISAR DURANTE IMPLEMENTAÇÃO

Antes de implementar uma entidade:

- `04_ENTIDADES_E_RELACIONAMENTOS.md`
- `15_PADROES_ENTIDADES_DTOS_REPOSITORIES.md`
- `16_RELACIONAMENTOS_JPA_CASCADE_FETCH.md`

Antes de criar Controller/API:

- `06_APIS_CRUDS_E_REGRAS.md`
- `17_PADROES_HTTP_ERROS_PAGINACAO.md`

Antes de escrever Service:

- `18_MAPA_METODOS_SERVICES.md`

Antes de marcar concluído:

- `19_CHECKLIST_REVISAO_QUALIDADE.md`

Ao usar outra IA para revisar:

- `20_GUIA_USO_IA_PARA_REVISAO.md`

---

# 11. STATUS DA ETAPA ATUAL

## ETAPA 0 — Fundação

- [ ] 0.1 Criar projeto Spring Boot. *(repositório já criado)*
- [ ] 0.2 Configurar H2 dev/test.
- [ ] 0.3 Criar estrutura inicial de packages.
- [ ] 0.4 Criar tratamento global de erros.
- [ ] 0.5 Criar smoke tests.
- [ ] 0.6 Validar execução local.

---

# 12. PRÓXIMA TAREFA EXATA

## 0.1 — Spring Boot + H2

Repositório já criado:

```text
gbsalermo/AkosMed-BackEnd
```

Agora gerar o projeto Spring Boot dentro dele.

Dependências:

- Spring Web
- Spring Data JPA
- Validation
- H2 Database
- Spring Boot Test

Não adicionar ainda:

- PostgreSQL Driver
- Flyway
- Springdoc
- JWT
- bibliotecas de mensageria

Depois:

1. rodar `mvn test`;
2. iniciar aplicação;
3. registrar resultado;
4. atualizar este arquivo;
5. commit;
6. avançar para 0.2.

---

# 13. REGISTRO DA INICIALIZAÇÃO DO REPOSITÓRIO

## 2026-08-23

- [x] Repositório `gbsalermo/AkosMed-BackEnd` criado.
- [x] Documentação oficial v5 adicionada à branch `main`.
- [x] README inicializado.
- [x] Roadmap e guias técnicos versionados.
- [ ] Projeto Spring Boot ainda não gerado.
- [ ] H2 ainda não configurado.

A criação do repositório, por si só, não conclui a ETAPA 0.1.
