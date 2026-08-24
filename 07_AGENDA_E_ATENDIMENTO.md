# 07 — Agenda e Atendimento

## Modelo simplificado

```text
DisponibilidadeProfissional
        +
BloqueioAgenda
        +
Agendamento existente
        =
Horários disponíveis
```

Sem tabela `Agenda` no MVP.

## Disponibilidade

Exemplo:

```text
Segunda — Unidade Centro
08:00–12:00
14:00–18:00
```

Representar como duas linhas.

Isso é mais simples do que modelar intervalos dentro da mesma linha.

## Duração

Ordem para resolver duração:

1. `ProfissionalProcedimento.duracaoMinutosOverride`;
2. `Procedimento.duracaoPadraoMinutos`.

## Geração de slots

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

Remover candidatos que colidam com bloqueio/agendamento.

## Service sugerido

```text
AvailabilityService
```

Método conceitual:

```text
listAvailableSlots(profissionalId, unidadeId, procedimentoId, data)
```

## Agendamento

`AgendamentoService.criar(...)` deve:

1. obter tenant;
2. validar unidade;
3. validar paciente;
4. bloquear profissional para escrita;
5. validar vínculo profissional/unidade;
6. validar procedimento;
7. calcular duração;
8. validar disponibilidade;
9. validar bloqueio;
10. validar conflito;
11. salvar;
12. registrar evento;
13. criar notificação se necessário.

## Reagendamento

Não sobrescrever histórico sem registro.

Opção prática:

- atualizar `inicio/fim` do agendamento;
- criar `EventoAgendamento` com horário anterior e motivo em metadata/campos.

Não precisa criar novo Agendamento para cada reagendamento no MVP.

## Estados

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

### Fluxo principal

```text
SOLICITADO -> CONFIRMADO -> CHECK_IN -> EM_ATENDIMENTO -> CONCLUIDO
```

### Fluxos alternativos

```text
SOLICITADO -> CANCELADO_*
CONFIRMADO -> CANCELADO_*
CONFIRMADO -> FALTOU
```

## Check-in

Significa apenas:

```text
Paciente chegou/foi registrado no local.
```

Não iniciar atendimento automaticamente.

## Atendimento

Quando profissional inicia:

1. validar status do agendamento;
2. criar `Atendimento`;
3. mudar agendamento para `EM_ATENDIMENTO`;
4. registrar evento.

Ao concluir:

1. fechar `Atendimento`;
2. mudar agendamento para `CONCLUIDO`;
3. registrar evento.

## Atendimento avulso

Permitido com `agendamento = null`.

Usos:

- encaixe;
- urgência;
- atendimento institucional;
- registro manual autorizado.

Mesmo assim a criação exige como entrada:

- unidade;
- paciente;
- profissional.

O Service resolve/cria o `Prontuario` do paciente e a Entity `Atendimento` referencia o Prontuario, sem duplicar `paciente_id`.

## Visão da secretaria

Endpoint pode retornar DTO específico:

```text
horario
paciente
profissional
procedimento
status
chegou
observacaoAdministrativa
```

## Visão do profissional

```text
horario
paciente
procedimento
status
aguardando
```

Evitar colocar prontuário inteiro no DTO da agenda.

## Pendências

No MVP, não criar entidade `Pendencia` genérica.

Calcular pendências por query:

- prescrições em rascunho;
- atendimentos abertos;
- consultas aguardando confirmação;
- pacientes em check-in.

Se o conceito crescer, criar entidade depois.


---

## Revisão obrigatória da agenda

Antes de concluir agenda/agendamento, verificar:

- `15_PADROES_ENTIDADES_DTOS_REPOSITORIES.md` para query de overlap;
- `16_RELACIONAMENTOS_JPA_CASCADE_FETCH.md` para relacionamentos;
- `18_MAPA_METODOS_SERVICES.md` para responsabilidades;
- `19_CHECKLIST_REVISAO_QUALIDADE.md` para concorrência/tenant.

A regra funcional é implementada em H2, mas concorrência real será revalidada no PostgreSQL.
