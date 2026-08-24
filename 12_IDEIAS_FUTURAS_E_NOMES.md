# 12 — Visão Futura do Produto

## Marca

Nome definido:

# AkosMed

Estrutura possível:

```text
AkosMed
├── Akos Core
├── AkosMed Web
├── AkosMed Patient
├── Akos Assistant
├── Akos Dental
├── Akos Psi
├── Akos Vision
└── Akos Enterprise
```

Repositório:

```text
akosmed-backend
```

Package:

```text
br.com.akosmed
```

Antes de uso comercial, validar disponibilidade de marca/domínio.

## Akos Assistant

Objetivo:

- agenda;
- próximos pacientes;
- check-in;
- alertas;
- pendências;
- resumo diário.

A vantagem de implementar primeiro como endpoints é poder usar a mesma lógica em:

- web;
- mobile;
- bot;
- push.

## AkosMed Patient

App Kotlin/Android.

Funções futuras:

- próximas consultas;
- agendamentos;
- horários disponíveis;
- prescrições;
- documentos;
- notificações;
- contato administrativo.

## Lembretes de medicamento

A estrutura `ItemPrescricao` já permite calcular:

- tratamento ativo;
- data de término;
- frequência;
- intervalos teóricos.

Futuro:

- lembrete de dose;
- aviso de fim do tratamento;
- adesão opcional.

Não fazer isso antes da prescrição estar sólida.

## Contato com clínica

Preferir primeiro solicitações estruturadas.

Chat em tempo real só vale se houver demanda.

## IA

Possíveis usos futuros:

- resumo administrativo;
- classificação de solicitações;
- busca assistida;
- apoio de documentação.

Não deixar IA decidir regras críticas do domínio nem substituir validações clínicas.

## Enterprise

Possíveis recursos:

- instância dedicada;
- banco dedicado;
- SSO;
- auditoria avançada;
- integrações corporativas;
- limites/configurações próprias.

Implementar apenas com cliente real exigindo.


---

## Ordem futura preservada

As ideias deste arquivo não alteram o Core.

A prioridade continua:

```text
Backend MVP
↓
API paciente
↓
Kotlin
↓
Akos Assistant
↓
Especializações/módulos
```
