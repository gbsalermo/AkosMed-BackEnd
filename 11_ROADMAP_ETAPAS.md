# 11 — ROADMAP DE DESENVOLVIMENTO — AkosMed

> Roadmap executável.  
> Trabalhar uma subetapa por vez.  
> Só marcar `[x]` após teste executado.

---

# PADRÃO FIXO DE SUBETAPA

```text
1. regra
2. Entity/Enum
3. DTOs
4. Repository
5. Service
6. Controller
7. validações/exceptions
8. testes automatizados
9. execução local
10. revisão de qualidade
11. CONTINUIDADE
12. commit
```

Swagger e Postman entram somente no fechamento.

---

# ETAPA 0 — FUNDAÇÃO H2

## 0.1 Projeto base

- [ ] criar repositório `akosmed-backend`;
- [ ] Spring Boot Maven;
- [ ] Java 21;
- [ ] package `br.com.akosmed`;
- [ ] dependências: Web, JPA, Validation, H2, Test;
- [ ] executar `mvn test`;
- [ ] iniciar aplicação;
- [ ] commit inicial.

### Aceite

```text
BUILD SUCCESS
Aplicação inicia
Sem PostgreSQL
Sem Swagger
Sem entidades clínicas
```

---

## 0.2 H2 e profiles

Criar:

```text
application.yml
application-dev.yml
application-test.yml
```

### dev

H2 local para desenvolvimento.

### test

H2 em memória.

- [ ] console H2 opcional apenas em dev;
- [ ] `ddl-auto=update` dev;
- [ ] `ddl-auto=create-drop` test;
- [ ] SQL logging controlado.

---

## 0.3 Estrutura de packages

Criar somente a estrutura necessária:

```text
shared
tenant
identity
professional
patient
scheduling
clinical
prescription
notification
audit
```

Não criar classes vazias só para preencher estrutura.

---

## 0.4 Tratamento global de erros

Criar:

- [ ] `ApiErrorDTO`;
- [ ] `FieldErrorDTO`;
- [ ] `GlobalExceptionHandler`;
- [ ] `ResourceNotFoundException`;
- [ ] `BusinessException`.

Testar:

- [ ] validação 400;
- [ ] recurso inexistente 404;
- [ ] conflito/regra 409.

---

## 0.5 Smoke tests

- [ ] `contextLoads`;
- [ ] teste de controller simples;
- [ ] `mvn test`.

### ETAPA 0 concluída quando

Aplicação base estiver estável com H2.

---

# ETAPA 1 — TENANT E UNIDADE

## 1.1 Tenant

Campos:

```text
id
nome
nomeFantasia
slug
documento
status
timezone
createdAt
updatedAt
```

### Métodos esperados

```text
criar
buscarPorId
listar
atualizar
ativar
suspender
```

### Regras

- slug obrigatório e único;
- sem hard delete;
- timezone obrigatório;
- status controlado por ação.

### Testes

- criar;
- duplicidade de slug;
- buscar;
- atualizar;
- suspender;
- ativar.

---

## 1.2 Unidade

Campos:

```text
id
tenant
nome
codigo
telefone
email
endereco
ativo
```

### Regras

- código único dentro do tenant;
- unidade sempre possui tenant;
- tenant suspenso não recebe nova unidade;
- sem hard delete.

### Repository mínimo

```text
findByIdAndTenantId
findAllByTenantId
existsByCodigoAndTenantId
```

### Testes críticos

- mesmo código em tenants diferentes funciona;
- mesmo código no mesmo tenant falha;
- busca cross-tenant não retorna.

---

## 1.3 Revisão Etapa 1

Usar:

- `16_RELACIONAMENTOS_JPA_CASCADE_FETCH.md`;
- `19_CHECKLIST_REVISAO_QUALIDADE.md`.

---

# ETAPA 2 — IDENTIDADE, AUTH E MULTI-TENANCY

Adicionar agora:

```text
Spring Security
biblioteca JWT escolhida
```

## 2.1 Pessoa

Tenant-scoped.

Campos:

```text
id
tenant
nomeCompleto
nomeSocial
cpf
dataNascimento
telefone
emailContato
```

### Cuidados

- CPF pode ser nullable;
- se informado, único por tenant;
- não usar e-mail de contato como credencial.

---

## 2.2 Usuario

Global.

Campos:

```text
id
emailLogin
passwordHash
status
superAdmin
ultimoLogin
```

### Cuidados

- email de login único global;
- nunca retornar hash;
- password nunca entra em ResponseDTO.

---

## 2.3 UsuarioTenant

Liga:

```text
Usuario
Tenant
Pessoa
PerfilTenant
```

Campos:

```text
id
usuario
tenant
pessoa
perfilTenant
acessoTodasUnidades
ativo
```

Constraint:

```text
unique(usuario_id, tenant_id)
```

---

## 2.4 UsuarioUnidade

Usar somente se:

```text
acessoTodasUnidades = false
```

Constraint:

```text
unique(usuario_tenant_id, unidade_id)
```

Service deve garantir que unidade e UsuarioTenant pertencem ao mesmo tenant.

---

## 2.5 Login tenant-scoped

Request:

```json
{
  "email": "user@exemplo.com",
  "senha": "...",
  "tenantSlug": "clinica-vida"
}
```

Fluxo:

```text
Usuario
↓
Tenant por slug
↓
UsuarioTenant ativo
↓
senha
↓
token com tenantId + usuarioId + usuarioTenantId + perfil
```

---

## 2.6 Refresh token

Entidade:

```text
RefreshToken
id
usuarioTenant
tokenHash
expiraEm
revogadoEm
```

Nunca guardar refresh token puro se puder guardar hash.

---

## 2.7 TenantContext

Extrair do token.

O Controller não recebe `tenantId`.

---

## 2.8 Testes cross-tenant

Criar Tenant A/B e usuários A/B.

Testar leitura e escrita cruzadas.

### ETAPA 2 concluída quando

Isolamento estiver comprovado.

---

# ETAPA 3 — ESTRUTURA CLÍNICA BASE

## 3.1 Especialidade

Campos:

```text
id
tenant
nome
codigo
descricao
ativo
```

Regras:

- código único por tenant;
- inativação em vez de delete.

---

## 3.2 Procedimento

Campos:

```text
id
tenant
nome
codigo
descricao
duracaoPadraoMinutos
valorPadrao
ativo
```

Regras:

- duração > 0;
- dinheiro com BigDecimal;
- código único por tenant.

---

## 3.3 ProfissionalSaude

Campos:

```text
id
tenant
pessoa
tipoProfissional
conselho
numeroRegistro
ufRegistro
ativo
```

Regras:

- Pessoa deve pertencer ao mesmo tenant;
- não duplicar registro profissional dentro do tenant quando informado.

---

## 3.4 ProfissionalEspecialidade

Entidade de vínculo explícita.

Campos:

```text
id
profissional
especialidade
principal
```

Não usar `@ManyToMany`.

---

## 3.5 ProfissionalUnidade

Entidade de vínculo.

Profissional só pode atender em unidade vinculada.

---

## 3.6 ProfissionalProcedimento

Entidade de vínculo.

Campos extras:

```text
duracaoMinutosOverride
valorOverride
ativo
```

---

## 3.7 Paciente

Campos:

```text
id
tenant
pessoa
numeroProntuario
status
observacaoAdministrativa
createdAt
```

Regras:

- Pessoa mesmo tenant;
- número prontuário único por tenant;
- sem PatientUnit no MVP;
- inativar em vez de apagar.

### ETAPA 3 concluída quando

Cadastros básicos estiverem consistentes e isolados.

---

# ETAPA 4 — DISPONIBILIDADE E BLOQUEIOS

## 4.1 DisponibilidadeProfissional

Campos:

```text
id
tenant
profissional
unidade
diaSemana
horaInicio
horaFim
vigenciaInicio
vigenciaFim
ativo
```

Regras:

- profissional vinculado à unidade;
- início < fim;
- não criar janelas inválidas;
- unidade mesmo tenant.

---

## 4.2 BloqueioAgenda

Campos:

```text
id
tenant
profissional
unidade
inicio
fim
tipo
motivo
ativo
```

No MVP, unidade obrigatória.

Para bloquear todas as unidades, criar um bloqueio por unidade.

---

## 4.3 AvailabilityService

Método principal:

```text
buscarHorariosDisponiveis(
  profissionalId,
  unidadeId,
  procedimentoId,
  data
)
```

Considerar:

- disponibilidade;
- procedimento;
- override profissional-procedimento;
- bloqueios;
- agendamentos ativos;
- duração.

Não persistir slots.

---

# ETAPA 5 — AGENDAMENTO

## 5.1 Agendamento

Campos:

```text
id
tenant
unidade
paciente
profissional
procedimento
inicio
fim
status
origem
observacaoAdministrativa
criadoPorUsuario
createdAt
updatedAt
```

### Criar

Service deve:

1. resolver tenant;
2. carregar paciente tenant-scoped;
3. carregar profissional tenant-scoped;
4. validar profissional-unidade;
5. carregar procedimento;
6. determinar duração;
7. validar disponibilidade;
8. verificar conflito;
9. salvar;
10. criar EventoAgendamento.

---

## 5.2 Listagens

Métodos:

```text
buscarPorId
listarPorData
listarPorProfissionalEPeriodo
listarPorPaciente
listarPorUnidadeEData
```

Todas paginadas quando puder crescer.

---

## 5.3 Ações

```text
confirmar
cancelar
reagendar
checkIn
marcarFalta
```

Cada uma valida transição.

---

## 5.4 EventoAgendamento

Não editar histórico.

Salvar eventos de mudança.

---

## 5.5 Conflito no H2

Criar teste funcional de overlap.

A proteção de concorrência definitiva será revalidada no PostgreSQL.

---

# ETAPA 6 — PRONTUÁRIO E ATENDIMENTO

## 6.1 Prontuario

Criado sob demanda para Paciente.

Constraint:

```text
unique(tenant_id, paciente_id)
```

Sem endpoint de delete.

---

## 6.2 Atendimento

Campos:

```text
id
tenant
prontuario
unidade
profissional
agendamento nullable
inicio
fim
status
tipoAtendimento
```

Regras:

- profissional e prontuário mesmo tenant;
- agendamento, se informado, deve ser compatível;
- atendimento pode ser avulso.

---

## 6.3 EvolucaoClinica

Campos:

```text
id
tenant
atendimento
profissional
conteudo
retificacaoDe nullable
motivoRetificacao nullable
createdAt
```

Regra:

registro original não é sobrescrito.

---

# ETAPA 7 — PRESCRIÇÃO E ANEXOS

## 7.1 Prescricao

Estados:

```text
RASCUNHO
EMITIDA
CANCELADA
```

Métodos:

```text
criarRascunho
buscar
listarPorPaciente
listarPorAtendimento
emitir
cancelar
```

---

## 7.2 ItemPrescricao

Manipulável apenas enquanto RASCUNHO.

Métodos:

```text
adicionarItem
atualizarItem
removerItem
```

Após EMITIDA:

- não editar silenciosamente;
- nova prescrição ou fluxo de cancelamento/substituição.

---

## 7.3 Gerador de receita

No Core:

gerar representação a partir da Prescricao.

Assinatura digital fica para futuro.

---

## 7.4 AnexoClinico

Banco guarda metadata.

Arquivo via:

```text
StorageService
```

No H2/desenvolvimento:

filesystem local pode ser usado.

---

# ETAPA 8 — OPERAÇÃO DIÁRIA E NOTIFICAÇÕES

## 8.1 Agenda diária

Service monta visão do usuário autenticado.

---

## 8.2 Pacientes aguardando

Baseado em status CHECK_IN.

---

## 8.3 Pendências

Começar simples, calculadas por consulta.

Não criar motor de regras genérico.

---

## 8.4 Notificacao

Primeira versão in-app.

Sem WhatsApp/Telegram ainda.

---

# ETAPA 9 — AUDITORIA E SEGURANÇA

## 9.1 AuditLog

Registrar ações críticas.

Evitar relacionamento JPA pesado no AuditLog.

Guardar IDs simples.

---

## 9.2 Hardening

- refresh rotation;
- revogação;
- CORS;
- logs;
- secrets;
- acesso por unidade;
- revisão de endpoints.

---

# ETAPA 10 — REVISÃO FUNCIONAL COMPLETA EM H2

Antes de PostgreSQL:

- [ ] todos os testes passam;
- [ ] nenhuma regra depende de SQL específico;
- [ ] relacionamentos revisados;
- [ ] cascades revisados;
- [ ] endpoints revisados;
- [ ] paginação revisada;
- [ ] exceptions revisadas;
- [ ] tenant isolation revisado;
- [ ] código morto removido;
- [ ] TODO crítico resolvido.

Executar fluxo ponta a ponta por teste:

```text
Tenant
→ Unidade
→ Usuário
→ Profissional
→ Paciente
→ Disponibilidade
→ Agendamento
→ Check-in
→ Atendimento
→ Evolução
→ Prescrição
```

---

# ETAPA 11 — POSTGRESQL + FLYWAY

Agora adicionar:

```text
PostgreSQL Driver
Flyway
Testcontainers PostgreSQL
```

## 11.1 Revisão de modelagem

Antes das migrations:

- nomes de tabelas;
- FKs;
- índices;
- uniques;
- nullability;
- BigDecimal;
- timestamps;
- tamanhos de varchar/text.

---

## 11.2 Migrations

Criar migrations a partir do modelo estabilizado.

Produção:

```text
ddl-auto=validate
```

---

## 11.3 Reexecutar testes no PostgreSQL

Obrigatório revalidar:

- multi-tenant;
- constraints;
- paginação;
- consultas;
- locks;
- concorrência do agendamento.

---

## 11.4 Double booking real

Testcontainers/PostgreSQL:

duas transações concorrentes para mesmo profissional/horário.

Apenas uma pode concluir.

Se necessário, ajustar lock/constraint.

---

# ETAPA 12 — SWAGGER / OPENAPI

Somente agora adicionar Springdoc.

Documentar:

- tags por módulo;
- DTOs;
- exemplos;
- status HTTP;
- erros;
- JWT;
- filtros;
- paginação;
- ações de domínio.

Não documentar endpoint que ainda está instável.

---

# ETAPA 13 — POSTMAN

Criar coleção oficial:

```text
00 Auth
01 Tenant
02 Unidade
03 Usuários
04 Especialidades
05 Procedimentos
06 Profissionais
07 Pacientes
08 Disponibilidade
09 Agendamentos
10 Prontuário
11 Atendimento
12 Prescrições
13 Anexos
14 Operação diária
15 Auditoria
```

Criar environment:

```text
baseUrl
accessToken
tenantSlug
ids de teste
```

Validar happy path + erros principais.

---

# ETAPA 14 — FECHAMENTO BACKEND MVP 1.0

- [ ] testes H2;
- [ ] testes PostgreSQL;
- [ ] migrations;
- [ ] Swagger;
- [ ] Postman;
- [ ] README de execução;
- [ ] revisão de segurança;
- [ ] documentação atualizada;
- [ ] release/tag.

---

# ETAPA 15 — API DO PACIENTE

Criar endpoints `/me`.

Sem permitir paciente escolher outro `pacienteId`.

---

# ETAPA 16 — AKOSMED PATIENT / KOTLIN

Projeto separado.

---

# ETAPA 17 — AKOS ASSISTANT

Consumir serviços/endpoints já existentes.

---

# ETAPA 18 — ESPECIALIZAÇÕES

Dental / Psychology / Vision.

---

# ETAPA 19 — MÓDULOS ADICIONAIS

Financeiro, convênio, estoque, laboratório, TISS, RNDS etc.

---

# REGRA DE RETOMADA

1. abrir `CONTINUIDADE.md`;
2. identificar a próxima subetapa;
3. abrir os documentos técnicos correspondentes;
4. implementar apenas essa subetapa;
5. executar testes;
6. passar checklist `19`;
7. registrar resultado;
8. commit;
9. avançar.
