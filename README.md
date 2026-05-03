# 🧭 Sistema de Onboard de Novo Colaborador

> Sistema web para gestão do processo de integração de novos colaboradores, com controle de etapas, atribuição de tarefas por responsável e acompanhamento de progresso até a conclusão do onboarding.

---

## 📋 Sumário

- [Visão Geral](#visão-geral)
- [Stack Tecnológica](#stack-tecnológica)
- [Estrutura do Projeto](#estrutura-do-projeto)
- [Modelagem de Dados](#modelagem-de-dados)
- [Regras de Negócio](#regras-de-negócio)
- [Como Rodar o Projeto](#como-rodar-o-projeto)
- [Decisões de Design](#decisões-de-design)
- [Contexto Profissional](#contexto-profissional)

---

## Visão Geral

O **Sistema de Onboard de Novo Colaborador** é uma aplicação web desenvolvida para simular o fluxo de integração de novos funcionários em uma empresa, cobrindo desde o cadastro do contratado até a conclusão de todas as etapas do programa de onboarding. O sistema permite:

- Cadastrar novos colaboradores com cargo, departamento e data de admissão
- Criar planos de onboarding reutilizáveis por cargo ou departamento
- Dividir o plano em etapas sequenciais (Dia 1, Semana 1, Primeiro Mês)
- Atribuir tarefas individuais a responsáveis internos com prazo definido
- Acompanhar o progresso geral do onboarding com percentual calculado automaticamente
- Bloquear o avanço de etapa enquanto houver tarefas pendentes na etapa atual
- Detectar tarefas atrasadas e emitir alerta quando o onboarding está estagnado

O domínio foi escolhido por refletir um processo presente em qualquer empresa de médio e grande porte, tornando as regras de negócio reconhecíveis e demonstrando capacidade de modelar fluxos de trabalho reais com estado e sequenciamento.

---

## Stack Tecnológica

| Camada | Tecnologia |
|---|---|
| Linguagem | Java 17 |
| Framework | Spring Boot 3.2 |
| ORM | Hibernate via Spring Data JPA |
| Banco (dev) | H2 (em memória) |
| Banco (prod) | PostgreSQL |
| Frontend | HTML + CSS + JavaScript puro |
| Testes | JUnit 5 + Mockito |
| Build | Maven |
| Deploy | Railway |

---

## Estrutura do Projeto

```
sistema-onboard/
├── src/
│   ├── main/
│   │   ├── java/com/onboard/principal/
│   │   │   ├── model/
│   │   │   │   ├── NovoColaborador.java
│   │   │   │   ├── Departamento.java
│   │   │   │   ├── Cargo.java
│   │   │   │   ├── PlanoOnboard.java
│   │   │   │   ├── EtapaOnboard.java
│   │   │   │   └── TarefaOnboard.java
│   │   │   ├── repository/        ← próxima etapa
│   │   │   ├── service/           ← próxima etapa
│   │   │   └── controller/        ← próxima etapa
│   │   └── resources/
│   │       ├── application.properties
│   │       └── static/            ← HTML, CSS, JS
│   └── test/
│       └── java/com/onboard/principal/
│           └── service/           ← próxima etapa
└── pom.xml
```

---

## Modelagem de Dados

O sistema é composto por 6 entidades com os seguintes relacionamentos:

```
Cargo            (1) ──────────── (N) NovoColaborador
Departamento     (1) ──────────── (N) NovoColaborador
PlanoOnboard     (1) ──────────── (N) NovoColaborador
PlanoOnboard     (1) ──────────── (N) EtapaOnboard
EtapaOnboard     (1) ──────────── (N) TarefaOnboard
NovoColaborador  (1) ──────────── (N) TarefaOnboard
```

### Entidades

**NovoColaborador** — profissional em processo de integração
- `id` (UUID), `nome`, `email`, `telefone`, `dataAdmissao`
- `progressoGeral` (BigDecimal calculado), `statusOnboard` (EM_ANDAMENTO, CONCLUIDO, ATRASADO)
- Chaves estrangeiras: `cargo_id`, `departamento_id`, `plano_id`
- Relacionamento: um colaborador executa um plano de onboarding e possui muitas tarefas

**Cargo** — função exercida pelo novo colaborador
- `id` (UUID), `titulo`, `nivel` (JUNIOR, PLENO, SENIOR, GESTOR)
- Relacionamento: um cargo é ocupado por muitos colaboradores

**Departamento** — área da empresa onde o colaborador será alocado
- `id` (UUID), `nome`, `gestor`
- Relacionamento: um departamento recebe muitos colaboradores

**PlanoOnboard** — modelo reutilizável de integração por cargo ou departamento
- `id` (UUID), `nome`, `descricao`, `duracaoEstimadaDias`
- Relacionamento: um plano contém muitas etapas e pode ser atribuído a muitos colaboradores

**EtapaOnboard** — fase sequencial dentro do plano de onboarding
- `id` (UUID), `titulo`, `ordem`, `descricao`
- `statusEtapa` (BLOQUEADA, EM_ANDAMENTO, CONCLUIDA)
- Chave estrangeira: `plano_id`
- Relacionamento: uma etapa contém muitas tarefas

**TarefaOnboard** — atividade individual dentro de uma etapa
- `id` (UUID), `titulo`, `descricao`, `responsavel`, `prazo`, `dataConclusao`
- `status` (PENDENTE, EM_ANDAMENTO, CONCLUIDA, ATRASADA)
- Chaves estrangeiras: `etapa_id`, `colaborador_id`
- Relacionamento: pertence a uma etapa e está vinculada a um colaborador

### Exemplos de tarefas por etapa

| Etapa | Tarefa | Responsável |
|---|---|---|
| Dia 1 | Entrega de crachá e equipamentos | TI |
| Dia 1 | Apresentação à equipe | RH |
| Semana 1 | Configuração de acessos e sistemas | TI |
| Semana 1 | Leitura do manual de conduta | Colaborador |
| Semana 1 | Reunião de alinhamento com gestor | Gestor |
| Primeiro Mês | Conclusão do treinamento inicial | Colaborador |
| Primeiro Mês | Avaliação de período de experiência | RH |

---

## Regras de Negócio

### Progresso geral do onboarding

```
progresso (%) = (tarefas concluídas / total de tarefas) × 100
```

| Progresso | Status |
|---|---|
| 0% a 25% | 🔴 Iniciando |
| 26% a 74% | 🟡 Em andamento |
| 75% a 99% | 🟢 Quase concluído |
| 100% | ✅ Onboarding concluído |

### Bloqueio de avanço de etapa

| Situação | Comportamento |
|---|---|
| Todas as tarefas da etapa concluídas | Próxima etapa liberada automaticamente |
| Existe tarefa pendente ou atrasada na etapa atual | Próxima etapa permanece bloqueada |
| Tentativa de concluir etapa com pendências | Exceção de negócio lançada pelo Service |

### Detecção de tarefa atrasada

Uma tarefa é marcada como `ATRASADA` automaticamente quando a data de `prazo` é ultrapassada e o `status` ainda não é `CONCLUIDA`. A verificação ocorre a cada consulta no Service.

### Alerta de onboarding estagnado

O onboarding de um colaborador recebe alerta de estagnação quando nenhuma tarefa foi concluída nos últimos **7 dias** e o status geral ainda não é `CONCLUIDO`.

---

## Como Rodar o Projeto

### Pré-requisitos
- Java 17+
- Maven 3.8+

### Passos

```bash
# 1. Clone o repositório
git clone https://github.com/seu-usuario/sistema-onboard.git

# 2. Acesse a pasta do projeto
cd sistema-onboard

# 3. Rode a aplicação
./mvnw spring-boot:run
```

A aplicação sobe em `http://localhost:8080`. O banco H2 é criado automaticamente em memória — nenhuma instalação adicional necessária.

---

## Decisões de Design

**Por que UUID como chave primária?**
UUIDs são a prática adotada em sistemas empresariais modernos por não exporem sequência numérica, facilitarem a integração entre sistemas e serem mais seguros em APIs públicas.

**Por que `PlanoOnboard` é uma entidade separada de `NovoColaborador`?**
O plano de onboarding é um modelo reutilizável — o mesmo plano pode ser aplicado a todos os contratados de um cargo. Acoplá-lo diretamente ao colaborador criaria duplicação de dados e impediria reaproveitar a estrutura para futuras contratações.

**Por que `EtapaOnboard` tem um campo `ordem`?**
O sequenciamento das etapas é uma regra de negócio central — Dia 1 deve sempre preceder Semana 1, que deve preceder Primeiro Mês. Persistir a ordem como dado evita depender de ordenação implícita por data de criação, que é frágil e difícil de testar.

**Por que o progresso geral é calculado e não persistido?**
O percentual de conclusão depende diretamente do estado das tarefas, que muda constantemente. Persistir esse valor criaria risco de inconsistência. O cálculo sob demanda no Service garante que o dado retornado sempre reflita o estado atual do onboarding.

**Por que bloquear o avanço de etapa no Service e não no Controller?**
Regras de negócio pertencem à camada de Service — não à de Controller. Validar no Controller expõe lógica de domínio na camada errada e dificulta o teste unitário com Mockito, que opera diretamente nos Services.

---

## Contexto Profissional

Este projeto faz parte de um portfólio estruturado para transição de carreira para Engenharia de Software, com foco em vagas Java Júnior no segmento de ERP (TOTVS e similares).

O domínio de onboarding foi escolhido por ser um processo universal em empresas de médio e grande porte — e por introduzir no portfólio o conceito de **fluxo com estado e sequenciamento**, que eleva a complexidade técnica em relação ao projeto anterior e demonstra evolução consistente na modelagem de sistemas.

O portfólio segue uma progressão planejada de complexidade:

| # | Projeto | Status |
|---|---|---|
| 1 | Calculadora de Precificação e Comissão — SST | ✅ Concluído |
| 2 | Sistema de Onboard de Novo Colaborador | 🔧 Em andamento |
| 3 | CRM com Pipeline de Vendas | 📋 Backlog |
| 4 | Sistema de Metas e OKRs | 📋 Backlog |
| 5 | Dashboard de Rastreamento Logístico | 📋 Backlog |
| 6 | Simulador de Roteirização de Entregas | 📋 Backlog |
