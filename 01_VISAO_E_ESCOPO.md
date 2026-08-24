# 01 — Visão e Escopo

## Produto

AkosMed é uma plataforma backend para organizar o fluxo ambulatorial de clínicas e consultórios.

O objetivo não é começar como sistema hospitalar completo. O foco é a operação diária de atendimento.

## Problemas que o Core resolve

- organização de clínicas com uma ou várias unidades;
- usuários e profissionais;
- pacientes;
- procedimentos;
- disponibilidade de horários;
- agendamento;
- chegada/check-in;
- atendimento;
- prontuário;
- prescrição;
- documentos;
- visão do dia;
- notificações;
- auditoria.

## Cenários

### Consultório individual

```text
Tenant
└── Unidade
    └── Profissional
```

### Clínica multidisciplinar

```text
Tenant
└── Unidade
    ├── Medicina
    ├── Psicologia
    ├── Nutrição
    └── Fisioterapia
```

### Rede

```text
Tenant
├── Unidade Centro
├── Unidade Norte
└── Unidade Sul
```

## Usuários

MVP:

- `Usuario.superAdmin` para administração global da plataforma;
- `ADMIN_TENANT`;
- `SECRETARIA`;
- `PROFISSIONAL`;
- `PACIENTE`;
- `AUDITOR`.

## MVP 1.0

### Obrigatório

- multi-tenant;
- multi-unidade;
- autenticação;
- controle de acesso;
- profissionais;
- especialidades;
- pacientes;
- procedimentos;
- disponibilidade;
- bloqueios;
- agendamento;
- check-in;
- atendimento;
- prontuário;
- evolução clínica;
- prescrição estruturada;
- anexos básicos;
- agenda diária;
- notificações internas;
- auditoria crítica;
- documentação e testes.

### Preparar, mas não implementar agora

- app Kotlin do paciente;
- Akos Assistant;
- push;
- geração avançada de receita;
- módulos por especialidade.

### Não fazer no MVP

- microsserviços;
- financeiro completo;
- estoque;
- laboratório;
- TISS;
- RNDS;
- telemedicina;
- IA clínica;
- chat completo;
- customização dinâmica de permissões.

## Público e privado

O Core é o mesmo.

Diferenças devem surgir por módulos e regras, não por forks do backend.

## Princípio de produto

```text
Akos Core
 + configuração
 + especialização opcional
 = solução específica
```

## Critério de sucesso do MVP

Uma clínica deve conseguir:

1. cadastrar unidade;
2. cadastrar equipe;
3. cadastrar pacientes;
4. configurar horários;
5. agendar;
6. registrar chegada;
7. atender;
8. registrar evolução;
9. emitir prescrição estruturada;
10. consultar agenda e histórico;
11. operar sem acessar dados de outro tenant.


---

## Ordem de entrega do produto

O projeto será entregue em camadas de valor:

```text
1. Fundação técnica
2. Organização SaaS (Tenant/Unidade)
3. Segurança e identidade
4. Cadastros clínicos
5. Agenda e agendamento
6. Atendimento/prontuário
7. Prescrição/anexos
8. Operação diária
9. Hardening
10. Backend MVP
11. Paciente/mobile
12. Assistentes e especializações
```

A ordem executável oficial está em `11_ROADMAP_ETAPAS.md`.


---

## Estratégia de construção do backend

O AkosMed será desenvolvido primeiro com H2 para permitir evolução rápida do domínio.

Somente depois do Core funcional:

```text
H2
↓
revisão funcional
↓
PostgreSQL + Flyway
↓
Swagger
↓
Postman
↓
MVP
```

Essa ordem é intencional.

A documentação deve permitir que cada CRUD seja implementado e revisado sem depender de uma revisão direta dentro do repositório.
