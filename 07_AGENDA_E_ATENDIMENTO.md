# 07 — Agenda e Atendimento

> Referência funcional para disponibilidade, agendamento, check-in e atendimento.  
> A implementação ocorre nas ETAPAS 4–6 do roadmap.

---

# 1. Modelo de agenda

```text
DisponibilidadeProfissional
        +
BloqueioAgenda
        +
Agendamentos que ocupam agenda
        =
Horários disponíveis
```

Não existe tabela `Agenda` no MVP.

Não persistir slots calculados.

---

# 2. Identificadores

Na API/DTO:

```text
profissionalPublicId
unidadePublicId
procedimentoPublicId
pacientePublicId
agendamentoPublicId
```

No banco:

```text
Long id / FKs internas
```

O Service resolve UUID + Tenant antes de usar IDs internos.

---

# 3. Disponibilidade

Exemplo:

```text
Segunda — Unidade Centro
08:00–12:00
14:00–18:00
```

Representar como duas linhas.

Isso é mais simples e consultável do que guardar vários intervalos em uma única coluna/estrutura.

Cada disponibilidade valida:

- profissional;
- Unidade;
- mesmo Tenant;
- vínculo profissional-Unidade;
- `horaInicio < horaFim`;
- vigência válida;
- status ativo.

---

# 4. Duração

Ordem para resolver duração:

```text
1. ProfissionalProcedimento.duracaoMinutosOverride
2. Procedimento.duracaoPadraoMinutos
```

Duração deve ser positiva.

Não confiar em duração arbitrária enviada pelo cliente no fluxo normal de agendamento.

---

# 5. Geração de slots

Exemplo:

```text
Janela: 08:00–10:00
Duração: 30 min
```

Candidatos:

```text
08:00–08:30
08:30–09:00
09:00–09:30
09:30–10:00
```

Remover candidatos que colidam com:

- bloqueio ativo;
- agendamento em status que ocupa agenda;
- limites de vigência/disponibilidade.

A resposta de disponibilidade é uma fotografia do momento; **não reserva o horário**.

---

# 6. AvailabilityService

Método conceitual:

```text
buscarHorariosDisponiveis(
    UUID profissionalPublicId,
    UUID unidadePublicId,
    UUID procedimentoPublicId,
    LocalDate data
)
```

Responsabilidades:

1. resolver Tenant;
2. carregar profissional tenant-scoped;
3. carregar Unidade tenant-scoped;
4. validar vínculo;
5. carregar Procedimento;
6. resolver duração;
7. carregar disponibilidade da data;
8. remover bloqueios;
9. remover agendamentos que ocupam agenda;
10. devolver slots no contrato REST.

Timezone local é determinado por:

```text
Tenant.timezone
```

Regra temporal dependente do “agora” deve usar `Clock`.

---

# 7. Intervalos e overlap

O projeto usa intervalos:

```text
[inicio, fim)
```

Conflito existe quando:

```text
existing.inicio < novoFim
AND
existing.fim > novoInicio
```

Logo:

```text
14:00–14:30
14:30–15:00
```

é permitido.

Já:

```text
14:00–14:30
14:15–14:45
```

é conflito.

Centralizar a lógica para criação e reagendamento.

---

# 8. Status que ocupam agenda

Inicialmente:

```text
SOLICITADO
CONFIRMADO
CHECK_IN
EM_ATENDIMENTO
```

Não ocupam:

```text
CANCELADO_PACIENTE
CANCELADO_CLINICA
FALTOU
```

`CONCLUIDO` representa histórico passado e não deve gerar conflito com um novo slot futuro; consultas por período/status precisam respeitar a semântica temporal real.

Centralizar o conjunto de status para não duplicar regra em queries diferentes.

---

# 9. AgendamentoService.criar

Fluxo obrigatório:

```text
@Transactional
↓
resolver Tenant
↓
resolver paciente por publicId + Tenant
↓
resolver/bloquear profissional por publicId + Tenant
↓
resolver Unidade
↓
validar vínculo profissional-Unidade
↓
resolver Procedimento
↓
validar profissional-Procedimento
↓
calcular duração/fim
↓
validar disponibilidade
↓
validar bloqueios
↓
validar overlap
↓
salvar Agendamento
↓
registrar EventoAgendamento
↓
notificar quando aplicável
↓
commit
```

O `PESSIMISTIC_WRITE` deve estar dentro da mesma transação que revalida e salva.

Não fazer apenas:

```text
GET slots → escolhe → INSERT
```

sem revalidação transacional.

---

# 10. Double booking

Cenário crítico:

```text
14:00 livre
Paciente A observa livre
Paciente B observa livre
A e B enviam POST quase ao mesmo tempo
```

Resultado obrigatório:

```text
1 request → sucesso
1 request → 409 AGENDAMENTO_CONFLITO
```

Nunca dois agendamentos ativos sobrepostos para o mesmo profissional.

Proteção:

```text
Service validation
+
PESSIMISTIC_WRITE
+
PostgreSQL exclusion constraint quando validada na ETAPA 11
```

Detalhes: `21_PUBLIC_ID_E_CONCORRENCIA.md`.

---

# 11. Reagendamento

Reagendamento usa a mesma proteção da criação.

Fluxo:

```text
@Transactional
→ carregar Agendamento por publicId + Tenant
→ lock do profissional de destino
→ validar novo slot
→ overlap excluindo o próprio Agendamento
→ atualizar inicio/fim
→ registrar EventoAgendamento
```

Não sobrescrever histórico sem evento.

No MVP pode manter o mesmo registro de Agendamento e registrar horário anterior/motivo no evento.

Se no futuro houver troca simultânea entre profissionais, locks devem ser adquiridos em ordem determinística para reduzir risco de deadlock.

---

# 12. Idempotência

Criação de agendamento é candidata a:

```http
Idempotency-Key
```

Motivos:

- timeout;
- retry automático;
- duplo clique;
- reconexão mobile futura.

Não criar infraestrutura genérica antes da subetapa decidir implementá-la.

Se adotada, seguir `22_CONCORRENCIA_IDEMPOTENCIA_CLOCK_OPERACAO.md`.

---

# 13. Estados do Agendamento

```text
SOLICITADO
CONFIRMADO
CHECK_IN
EM_ATENDIMENTO
CONCLUIDO
FALTOU
CANCELADO_PACIENTE
CANCELADO_CLINICA
```

Fluxo principal:

```text
SOLICITADO
→ CONFIRMADO
→ CHECK_IN
→ EM_ATENDIMENTO
→ CONCLUIDO
```

Fluxos alternativos:

```text
SOLICITADO → CANCELADO_*
CONFIRMADO → CANCELADO_*
CONFIRMADO → FALTOU
```

Transições exatas devem ser centralizadas/testadas.

Não aceitar `status` livre em update genérico.

---

# 14. Check-in

Significa:

```text
Paciente chegou/foi registrado no local.
```

Não iniciar Atendimento automaticamente.

Regras detalhadas de antecedência/atraso, se surgirem, devem usar `Clock` e entrar explicitamente no roadmap/regra.

---

# 15. Iniciar Atendimento

Quando o profissional inicia:

1. resolver Agendamento pelo `publicId + Tenant`;
2. validar status permitido;
3. validar autorização/profissional;
4. obter/criar Prontuário do Paciente;
5. garantir que não exista Atendimento incompatível/duplicado para o Agendamento;
6. criar `Atendimento`;
7. mudar Agendamento para `EM_ATENDIMENTO`;
8. registrar evento.

Tudo na transação adequada.

---

# 16. Concluir Atendimento

Fluxo:

1. carregar Atendimento tenant-scoped;
2. validar `EM_ANDAMENTO`;
3. definir `fim` usando `Clock`;
4. alterar Atendimento para `CONCLUIDO`;
5. alterar Agendamento associado para `CONCLUIDO` quando aplicável;
6. registrar evento/auditoria conforme regra.

Atendimento concluído não volta livremente a `EM_ANDAMENTO`.

---

# 17. Atendimento avulso

Permitido com:

```text
agendamento = null
```

Casos:

- encaixe;
- urgência;
- atendimento institucional;
- registro manual autorizado.

Entrada usa UUIDs públicos para:

- Unidade;
- Paciente;
- Profissional.

O Service resolve/cria o Prontuário e `Atendimento` referencia Prontuário, sem duplicar `paciente_id`.

---

# 18. Visão da secretaria

DTO específico pode retornar:

```text
horario
paciente summary
profissional summary
procedimento summary
status
chegou
observacaoAdministrativa
```

Summaries expõem `publicId`, não PK Long.

Não incluir prontuário inteiro na agenda.

---

# 19. Visão do profissional

```text
horario
paciente summary
procedimento summary
status
aguardando
```

Endpoints `/me` resolvem o profissional autenticado, sem `profissionalPublicId` escolhido pelo próprio usuário.

---

# 20. Pendências

No MVP, não criar entidade `Pendencia` genérica.

Calcular por query:

- prescrições em rascunho;
- atendimentos abertos;
- consultas aguardando confirmação;
- pacientes em check-in.

Se o conceito crescer e ganhar ciclo de vida próprio, modelar depois em etapa específica.

---

# 21. Testes obrigatórios

Disponibilidade:

- [ ] janelas válidas;
- [ ] período sem agenda;
- [ ] override de duração;
- [ ] bloqueio;
- [ ] cross-tenant.

Agendamento:

- [ ] slot válido;
- [ ] fora da disponibilidade;
- [ ] procedimento não habilitado;
- [ ] Unidade incorreta;
- [ ] overlap parcial/total;
- [ ] intervalo adjacente permitido;
- [ ] status que ocupam/liberam;
- [ ] reagendamento conflitante;
- [ ] publicId de outro Tenant;
- [ ] corrida concorrente básica em H2.

Atendimento:

- [ ] check-in não inicia automaticamente;
- [ ] iniciar com status inválido falha;
- [ ] somente um Atendimento por Agendamento quando a constraint/regra for adotada;
- [ ] Atendimento avulso;
- [ ] conclusão;
- [ ] cross-tenant.

PostgreSQL/Testcontainers na ETAPA 11:

- [ ] duas transações reais no mesmo slot;
- [ ] exatamente um sucesso + um conflito;
- [ ] reagendamento concorrente;
- [ ] exclusion constraint validada quando implementada.

---

# 22. Documentos obrigatórios na revisão

Antes de concluir agenda/agendamento:

- `15_PADROES_ENTIDADES_DTOS_REPOSITORIES.md`;
- `16_RELACIONAMENTOS_JPA_CASCADE_FETCH.md`;
- `18_MAPA_METODOS_SERVICES.md`;
- `19_CHECKLIST_REVISAO_QUALIDADE.md`;
- `21_PUBLIC_ID_E_CONCORRENCIA.md`;
- `22_CONCORRENCIA_IDEMPOTENCIA_CLOCK_OPERACAO.md`.

A regra funcional nasce no Core/H2; a garantia concorrente definitiva é revalidada no PostgreSQL.
