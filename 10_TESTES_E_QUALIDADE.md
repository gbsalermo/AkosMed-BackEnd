# 10 — Testes e Qualidade

# Objetivo

Cada subetapa deve ser confiável antes de avançar.

No AkosMed, qualidade não significa apenas “teste escrito”. Significa:

```text
regra implementada
+
teste executado
+
contrato publicId correto
+
isolamento tenant validado
+
comportamento real conferido
```

A validação acontece em três fases principais.

---

# FASE 1 — Core com H2

Ferramentas:

```text
JUnit 5
Mockito
Spring Boot Test
H2
```

Tipos de teste:

- unitário;
- repository;
- service;
- controller;
- integração.

Não exigir PostgreSQL, Swagger ou Postman nesta fase.

---

# 1. Teste unitário

Usar quando a regra puder ser isolada.

Exemplos:

- transição de status;
- cálculo de duração;
- validação de datas;
- escolha de override de procedimento;
- regras de ativação/inativação;
- cálculo baseado em `Clock`.

Para regras temporais:

```java
Clock.fixed(...)
```

Evitar testes dependentes do horário real da máquina.

---

# 2. Repository test

Validar queries importantes.

Padrões tenant-scoped:

```text
findByPublicIdAndTenantId
findAllByTenantId
existsBy...AndTenantId
```

Validar também:

- unicidade lógica;
- filtros por período;
- busca de overlap;
- ordenação/paginação quando relevante;
- lock JPA quando a etapa exigir.

Não criar teste que normalize `findById(Long idDoCliente)` como padrão de entrada pública.

O `Long id` pode aparecer em teste interno de persistência, FK ou query interna, mas não como contrato da API.

---

# 3. Testes de publicId

Para cada recurso persistido exposto externamente, verificar conforme aplicável:

- `publicId` é gerado automaticamente;
- é único;
- não muda em update;
- CreateDTO não permite escolher `publicId`;
- ResponseDTO não expõe `Long id`;
- path usa UUID;
- relação recebida pela API usa `...PublicId`;
- UUID inexistente retorna 404;
- UUID de outro Tenant retorna 404.

Referência: `21_PUBLIC_ID_E_CONCORRENCIA.md`.

---

# 4. Service test

É a prioridade para regras de negócio.

Testar:

- happy path;
- recurso inexistente;
- Tenant incorreto;
- status inválido;
- relacionamento incompatível;
- duplicidade;
- conflito;
- autorização relevante;
- lost update quando `@Version` for usado;
- idempotência quando adotada;
- tempo com `Clock` quando aplicável.

O Service deve ser testado como fronteira de regra/transação, não apenas como delegador de Repository.

---

# 5. Controller test

Validar:

- status HTTP;
- `@Valid`;
- body;
- UUID em paths;
- erro padronizado;
- `correlationId`;
- exceptions convertidas;
- autenticação/autorização quando Spring Security entrar;
- ausência de `Long id` no contrato.

Não testar regra de domínio complexa apenas no Controller.

---

# 6. Multi-tenant

Todo módulo tenant-scoped precisa de testes de isolamento.

Padrão mínimo:

```text
Tenant A → recurso A
Tenant B → usuário B

B tenta acessar publicId de A
↓
404 / não retorna recurso
```

Cobrir leitura e escrita quando a feature permitir ambos.

Também testar relacionamento cruzado:

```text
request no Tenant A
usa profissionalPublicId do Tenant B
↓
falha
```

Não basta filtrar listagem; criação/atualização de relacionamentos também deve ser tenant-safe.

---

# 7. Segurança/autenticação

A partir da ETAPA 2, validar:

- login válido/inválido;
- Tenant inexistente/suspenso;
- `UsuarioTenant` inativo;
- token expirado/inválido;
- refresh revogado;
- perfil permitido/negado;
- acesso por Unidade;
- paciente acessando somente os próprios dados em `/me`.

Claims tenant-scoped devem seguir o contrato por publicId:

```text
usuarioPublicId
tenantPublicId
usuarioTenantPublicId
perfilTenant
```

Não tornar IDs `Long` sequenciais parte esperada do contrato de JWT.

---

# 8. Agenda e disponibilidade

Testes obrigatórios:

- profissional sem disponibilidade;
- Unidade incorreta;
- procedimento não habilitado;
- bloqueio;
- horário fora da janela;
- overlap total;
- overlap parcial;
- intervalo adjacente permitido;
- duração override;
- reagendamento conflitante;
- status inválido;
- Tenant incorreto.

Regra de overlap:

```text
existing.inicio < novoFim
AND
existing.fim > novoInicio
```

Intervalo:

```text
[inicio, fim)
```

---

# 9. Agendamento e concorrência

## H2/Core

Testar funcionalmente:

- slot livre;
- slot ocupado;
- criação e reagendamento usando a mesma regra;
- status que bloqueiam/liberam agenda;
- lock JPA configurado quando a ETAPA 5 chegar;
- teste concorrente básico quando viável.

A garantia final não é declarada apenas porque H2 passou.

## Resultado de domínio esperado

Duas tentativas para o mesmo profissional/slot:

```text
1 sucesso
1 AGENDAMENTO_CONFLITO
```

Nunca dois agendamentos ativos sobrepostos.

Referência: `21_PUBLIC_ID_E_CONCORRENCIA.md`.

---

# 10. Prontuário

Testar:

- criação sob demanda;
- apenas um prontuário por paciente/Tenant;
- atendimento compatível;
- evolução criada;
- retificação preserva original;
- ausência de delete destrutivo;
- cross-tenant;
- acesso não autorizado;
- histórico não sobrescrito silenciosamente.

---

# 11. Prescrição

Testar:

- criação em RASCUNHO;
- adicionar item;
- editar item;
- remover item;
- emitir;
- bloquear edição após emissão;
- cancelamento;
- paciente derivado do Atendimento/Prontuário;
- acesso cross-tenant negado;
- idempotência de emissão quando a decisão for implementada.

---

# 12. Anexos/storage

Quando a ETAPA 7 chegar, testar:

- metadata correta;
- Tenant/ownership antes de abrir storage;
- arquivo inexistente;
- inativação/remoção lógica quando aplicável;
- nenhum `storageKey` interno exposto em DTO;
- limites/MIME conforme o nível de implementação da etapa;
- falha de storage não deixando metadata inconsistente.

---

# 13. Optimistic locking

Quando uma Entity usar `@Version`, criar cenário com duas atualizações baseadas no mesmo estado.

Resultado:

```text
1 atualização vence
1 conflito
HTTP 409 RESOURCE_VERSION_CONFLICT na fronteira REST
```

Nunca permitir que a versão antiga sobrescreva silenciosamente a nova.

---

# 14. Idempotência

Quando uma operação realmente adotar `Idempotency-Key`, testar:

```text
mesma key + mesmo payload
→ mesmo efeito/resultado lógico
→ nenhum duplicado

mesma key + payload diferente
→ 409 IDEMPOTENCY_KEY_REUSED
```

Candidatas principais:

- criação de agendamento;
- emissão de prescrição.

Não criar esses testes antes de a infraestrutura existir na etapa correspondente.

---

# FASE 2 — PostgreSQL/Testcontainers

Entrada oficial:

```text
ETAPA 11
```

Adicionar/reexecutar cenários críticos em PostgreSQL real via Testcontainers.

Obrigatório validar:

- migrations Flyway;
- constraints;
- tipos PostgreSQL;
- multi-tenancy;
- locks;
- concorrência;
- queries de período;
- paginação;
- transações;
- timezone/instantes;
- índices e comportamento de queries críticas quando necessário.

---

# 15. Teste real de double booking

Cenário:

```text
profissional P
slot 14:00–14:30

transação A tenta reservar
transação B tenta reservar
```

Estratégia esperada:

```text
PESSIMISTIC_WRITE no profissional
→ uma transação prossegue
→ a outra espera
→ revalida
```

Resultado obrigatório:

```text
1 sucesso
1 conflito
0 overlaps persistidos
```

Repetir também para reagendamento concorrente.

Na ETAPA 11, avaliar/adicionar exclusion constraint PostgreSQL como segunda barreira.

A ETAPA 11 não é concluída se corrida concorrente ainda permitir overlap.

---

# 16. Testes de migration

Validar que um banco limpo consegue executar todas as migrations na ordem correta.

Verificar:

- FKs;
- unique constraints;
- nullability;
- tipos;
- índices necessários;
- exclusion constraint quando adotada;
- `ddl-auto=validate` compatível com as Entities.

Não aceitar migration “funcionando” apenas em banco já alterado manualmente.

---

# FASE 3 — Swagger + Postman

Swagger entra:

```text
ETAPA 12
```

Postman entra:

```text
ETAPA 13
```

Postman valida ponta a ponta a API já estabilizada no banco definitivo.

Criar cenários de:

- happy path;
- erros principais;
- autenticação;
- publicId;
- multi-tenant;
- autorização;
- conflitos;
- fluxo completo.

---

# 17. Fluxo E2E mínimo do MVP

```text
cria Tenant
↓
cria Unidade
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

Todos os identificadores HTTP do fluxo usam `publicId`.

Adicionar variações negativas de cross-tenant e conflito.

---

# 18. Pós-MVP — OrientacaoPaciente

Somente quando a ETAPA 15 chegar, testar:

- profissional envia orientação ao paciente correto;
- paciente só lista as próprias orientações;
- UUID de orientação de outro paciente/Tenant não é acessível;
- atendimento opcional é compatível com paciente/profissional;
- `storageKey` não aparece no DTO;
- URL de mídia exige autorização e é temporária/assinada quando suportada;
- MIME/tamanho são validados;
- remoção preserva regra de auditoria/histórico definida;
- conteúdo enviado não pode ser alterado pelo paciente.

Não implementar esses testes no Core apenas porque o recurso já está planejado.

---

# 19. Regra de status de conclusão

Teste escrito não significa concluído.

Registrar `[x]` somente quando:

- teste foi executado;
- resultado esperado ocorreu;
- falha encontrada foi corrigida;
- `mvn test` está verde;
- não há teste ignorado escondendo requisito crítico;
- `CONTINUIDADE.md` registra o estado real.

---

# 20. Definition of Done de uma subetapa

```text
[x] regra implementada
[x] publicId/contrato revisado
[x] testes executados
[x] tenant isolation revisado
[x] relacionamentos revisados
[x] exceptions revisadas
[x] Clock/concorrência/idempotência avaliados quando aplicável
[x] código sem TODO crítico
[x] checklist 19 executado
[x] CONTINUIDADE atualizado
[x] commit
```

Swagger/Postman só entram nas respectivas etapas finais.

---

# Regra final

```text
planejado ≠ implementado
implementado ≠ testado
testado uma vez ≠ seguro em outro banco/concorrência
```

Marcar etapa como concluída somente com evidência executada na fase correta.
