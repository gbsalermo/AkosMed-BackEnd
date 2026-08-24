# 10 — Testes e Qualidade

# Objetivo

Cada etapa deve ser confiável antes de avançar.

A validação será feita em três momentos diferentes.

---

# FASE 1 — Durante o Core com H2

Ferramentas:

```text
JUnit 5
Mockito
Spring Boot Test
H2
```

Tipos:

- unitário;
- repository;
- service;
- controller;
- integração.

Não exigir Swagger/Postman nessa fase.

---

# Teste unitário

Usar quando a regra puder ser isolada.

Exemplos:

- transição de status;
- cálculo de duração;
- validação de datas;
- escolha de override de procedimento.

---

# Repository test

Validar queries importantes.

Principalmente:

```text
findByIdAndTenantId
existsBy...AndTenantId
busca de conflito
filtros por período
```

---

# Service test

É a prioridade.

O Service concentra regra de negócio.

Testar:

- happy path;
- recurso inexistente;
- tenant incorreto;
- status inválido;
- relacionamento incompatível;
- duplicidade;
- conflito.

---

# Controller test

Validar:

- status HTTP;
- `@Valid`;
- body;
- exceptions convertidas;
- autenticação quando entrar Security.

---

# Multi-tenant

Todo módulo tenant-scoped precisa de pelo menos um teste de isolamento.

Padrão:

```text
Tenant A → recurso A
Tenant B → usuário B

B tenta acessar A
↓
não retorna recurso
```

---

# Agenda

Testes obrigatórios:

- profissional sem disponibilidade;
- unidade incorreta;
- procedimento não habilitado;
- bloqueio;
- horário fora da janela;
- overlap;
- duração override;
- reagendamento conflitante;
- status inválido.

---

# Prontuário

Testar:

- criação sob demanda;
- apenas um prontuário por paciente;
- atendimento compatível;
- evolução criada;
- retificação preserva original;
- ausência de delete destrutivo.

---

# Prescrição

Testar:

- rascunho;
- adicionar item;
- editar item;
- remover item;
- emitir;
- bloquear edição após emissão;
- cancelamento;
- acesso cross-tenant negado.

---

# FASE 2 — PostgreSQL/Testcontainers

Entrada:

```text
ETAPA 11
```

Reexecutar cenários críticos no PostgreSQL real.

Obrigatório:

- migrations;
- constraints;
- multi-tenant;
- concorrência;
- lock;
- queries de período;
- paginação;
- transações.

---

# Teste de concorrência

Cenário:

```text
profissional P
horário 14:00–14:30

thread A tenta reservar
thread B tenta reservar
```

Resultado:

```text
1 sucesso
1 conflito
```

Nunca dois agendamentos.

---

# FASE 3 — Postman

Somente após Swagger.

Objetivo:

validação ponta a ponta da API já estabilizada.

Criar:

- happy paths;
- erros principais;
- autenticação;
- multi-tenant;
- fluxos completos.

---

# Fluxo E2E mínimo

```text
cria tenant
↓
cria unidade
↓
cria usuário/admin
↓
cria profissional
↓
cria paciente
↓
cria disponibilidade
↓
busca slot
↓
agenda
↓
confirma
↓
check-in
↓
inicia atendimento
↓
evolução
↓
prescrição
↓
conclui
```

---

# Regra de status

Teste escrito não significa concluído.

Registrar `[x]` somente quando:

- teste foi executado;
- resultado esperado ocorreu;
- falha foi corrigida;
- `mvn test` está verde.

---

# Definition of Done de uma subetapa

```text
[x] regra implementada
[x] testes executados
[x] tenant isolation revisado
[x] relacionamentos revisados
[x] exceptions revisadas
[x] código sem TODO crítico
[x] CONTINUIDADE atualizado
[x] commit
```

Swagger/Postman só entram nas respectivas etapas finais.
