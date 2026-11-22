# Backlog

Histórias de usuário (HU) com critérios de aceitação (CA). As regras citadas (`RN01`...) estão em [DOMINIO.md](DOMINIO.md).

**Como usar:** pegue **uma HU por vez**, na ordem sugerida. Marque os `[ ]` conforme cada critério passar nos testes. Uma HU só está pronta quando **todos** os CA passam e a [Definição de pronto](#definição-de-pronto) é cumprida.

**Legenda:** Prioridade **Must** = MVP · **Should** = depois do MVP · **Could** = bônus. Esforço **P** (1 dia), **M** (2 a 3 dias), **G** (uma semana).

## Visão geral e ordem sugerida

**MVP = T01–T05 + HU01–HU14.** A HU15 (remarcar) é _Should_: faça logo depois do MVP, ainda dentro do marco M4.

| Marco             | Conteúdo  | O que você aprende                                        |
| ----------------- | --------- | --------------------------------------------------------- |
| **M0 · Fundação** | T01–T05   | TypeScript estrito, projeto, banco com Docker, testes, CI |
| **M1 · Contas**   | HU01–HU03 | Hash de senha, JWT, middlewares, validação com Zod        |
| **M2 · Serviços** | HU04–HU06 | CRUD, autorização por dono, paginação                     |
| **M3 · Horários** | HU07–HU09 | Fuso horário, intervalos, função pura de slots            |
| **M4 · Agendar**  | HU10–HU15 | Transações, concorrência, máquina de estados              |
| **M5 · Extras**   | HU16–HU20 | E-mail, jobs, OpenAPI, deploy                             |
| **M6 · Front**    | HU21      | React consumindo a API (opcional)                         |

```
T01 → T02 → T03 → T04 → T05
                  ↓
HU01 → HU02 → HU03
        ↓
HU04 → HU05 → HU06
        ↓
HU07 → HU08 → HU09 ──→ HU10 → HU11 → HU12 → HU13 → HU14 → HU15
```

---

# Epic 0 · Fundação técnica (M0)

Não são histórias de negócio. Cada tarefa termina quando o item **"Pronto quando"** vale.

### T01 · Projeto TypeScript

- **Pronto quando:** `npm run build`, `npm run lint` e `npm run typecheck` rodam sem erro; `tsconfig` com `"strict": true`; ESLint e Prettier configurados; estrutura de pastas do [DOMINIO.md](DOMINIO.md) criada; `.gitignore` ignora `node_modules`, `dist` e `.env`.
- 💡 **Estudar:** `strict` no TypeScript, módulos ESM × CommonJS, `ts-node`/`tsx` para desenvolvimento.

### T02 · Servidor HTTP básico

- **Pronto quando:** `GET /health` responde `200 {"status":"ok"}`; erros saem no formato padrão (`{error:{code,message,details}}`); rota inexistente responde 404 no mesmo formato; exceção inesperada responde 500 **sem vazar a stack** ao cliente; variáveis de ambiente são validadas na inicialização (o app não sobe com configuração faltando).
- 💡 **Estudar:** middleware de erro do Express (4 parâmetros), Zod para validar `process.env`.
- ⚠️ **Armadilha:** erros lançados dentro de funções `async` no Express 4 não chegam ao middleware de erro sozinhos. Verifique a versão que você usa.

### T03 · Banco de dados

- **Pronto quando:** `docker compose up -d` sobe o PostgreSQL; o schema do [DOMINIO.md](DOMINIO.md) existe como migration (Prisma ou equivalente); `npm run db:migrate` e `npm run db:seed` funcionam em um banco vazio; `.env.example` documenta a `DATABASE_URL`.
- 💡 **Estudar:** o que é migration, `timestamptz` × `timestamp`, UUID como chave.

### T04 · Testes

- **Pronto quando:** `npm test` roda testes unitários (sem banco) e de integração (com banco de teste separado do de desenvolvimento); cada teste de integração começa com o banco limpo; existe pelo menos um teste de `GET /health`.
- 💡 **Estudar:** Vitest, Supertest, por que testes não podem depender da ordem de execução.

### T05 · Integração contínua

- **Pronto quando:** um workflow do GitHub Actions roda lint, typecheck e testes (com um serviço PostgreSQL) em cada push e PR, e fica verde; README explica como rodar o projeto do zero.
- 💡 **Dica:** você já fez algo parecido no projeto do encurtador (`.github/workflows/ci.yml`). Reaproveite a estrutura.

---

# Epic 1 · Contas e autenticação (M1)

### HU01 · Cadastro

**Como** visitante, **quero** criar uma conta como cliente ou profissional, **para** usar o sistema.
`Must · M`

**Critérios de aceitação**

- [ ] **CA1.** Dado um nome, um e-mail ainda não usado, uma senha com 8 a 128 caracteres e um papel (`CUSTOMER` ou `PROVIDER`), quando envio `POST /auth/register`, então recebo **201** com `id`, `name`, `email` e `role`, **sem** o hash nem a senha.
- [ ] **CA2.** O e-mail é normalizado (`trim` + minúsculas). Cadastrar `Ana@X.com` quando já existe `ana@x.com` retorna **409** `EMAIL_TAKEN`.
- [ ] **CA3.** Dados inválidos (e-mail mal formado, senha curta, papel desconhecido) retornam **400** `VALIDATION_ERROR`, com `details` indicando **cada campo** com problema.
- [ ] **CA4.** A senha é guardada com hash (argon2 ou bcrypt). Nunca aparece em resposta nem em log.
- [ ] **CA5.** Se o papel é `PROVIDER`, o `timezone` é obrigatório e precisa ser um fuso IANA válido (**400** caso contrário). Para `CUSTOMER`, é opcional.

💡 **Estudar:** hash × criptografia, _salt_, `Intl.supportedValuesOf('timeZone')` para validar fusos, erro de chave única do Prisma (`P2002`).
⚠️ **Armadilha:** verificar "e-mail existe?" e depois inserir tem uma janela de corrida. A garantia de verdade é o índice único; trate o erro dele.

### HU02 · Login

**Como** usuário cadastrado, **quero** entrar com e-mail e senha, **para** acessar as funções do meu papel.
`Must · M`

**Critérios de aceitação**

- [ ] **CA1.** Com credenciais corretas, `POST /auth/login` retorna **200** com `accessToken` (JWT) e `expiresIn`.
- [ ] **CA2.** E-mail inexistente e senha errada retornam **401** com **a mesma** mensagem (não revela quais e-mails têm conta).
- [ ] **CA3.** O token carrega o `id` do usuário (`sub`) e o `role`, e expira (ex.: 1 hora). O segredo vem de variável de ambiente.
- [ ] **CA4.** Rota protegida sem token, com token inválido ou expirado retorna **401** `UNAUTHENTICATED`.
- [ ] **CA5.** Rota restrita a um papel, acessada por outro papel, retorna **403** `FORBIDDEN`.

💡 **Estudar:** JWT (cabeçalho, payload, assinatura), diferença entre **401** (quem é você?) e **403** (você não pode), middlewares `authenticate` e `requireRole('PROVIDER')`.
⚠️ **Armadilha:** o payload do JWT é **legível** por qualquer um (só é assinado, não secreto). Nunca coloque senha nem dados sensíveis nele.

### HU03 · Meu perfil

**Como** usuário logado, **quero** ver e editar meus dados, **para** mantê-los atualizados.
`Must · P`

**Critérios de aceitação**

- [ ] **CA1.** `GET /me` retorna os dados do usuário do token (sem hash).
- [ ] **CA2.** `PATCH /me` altera `name` e, para profissionais, `timezone` (validado como na HU01).
- [ ] **CA3.** E-mail e papel **não** podem ser alterados por esta rota (campos ignorados ou **400**; escolha um e documente).
- [ ] **CA4.** Mudar o fuso **não altera** agendamentos existentes (estão em UTC), mas passa a valer para os cálculos de horário dali em diante.

---

# Epic 2 · Serviços (M2)

### HU04 · Cadastrar serviço

**Como** profissional, **quero** cadastrar os serviços que ofereço, **para** que clientes possam agendá-los.
`Must · P`

**Critérios de aceitação**

- [ ] **CA1.** `POST /services` com `name`, `durationMinutes` e `priceCents` (opcional) cria o serviço vinculado a mim e retorna **201**.
- [ ] **CA2.** `durationMinutes` precisa estar entre 15 e 480 e ser múltiplo de 15 (RN02); `name` tem 2 a 80 caracteres; `priceCents` é inteiro ≥ 0. Violações retornam **400**.
- [ ] **CA3.** Cliente tentando criar serviço recebe **403**.
- [ ] **CA4.** O serviço nasce com `active = true`.

💡 **Dica:** guarde dinheiro em **centavos inteiros**. Números decimais (`0.1 + 0.2`) geram erro de arredondamento.

### HU05 · Gerenciar meus serviços

**Como** profissional, **quero** listar, editar e desativar meus serviços, **para** manter meu catálogo atualizado.
`Must · P`

**Critérios de aceitação**

- [ ] **CA1.** `GET /services` lista **só os meus**, paginado, com filtro opcional `?active=true|false`.
- [ ] **CA2.** `PATCH /services/:id` edita `name`, `durationMinutes`, `priceCents` e `active`, com as mesmas validações da HU04.
- [ ] **CA3.** Editar ou ver serviço de **outro** profissional retorna **404** (RN08).
- [ ] **CA4.** Não existe `DELETE`: o serviço sai de circulação com `active = false` (RN09).
- [ ] **CA5.** Alterar a duração de um serviço **não** muda agendamentos já feitos (RN11, depende da HU10).

### HU06 · Catálogo público

**Como** cliente, **quero** ver os profissionais e seus serviços ativos, **para** escolher com quem agendar.
`Must · P`

**Critérios de aceitação**

- [ ] **CA1.** `GET /providers` (sem login) lista profissionais, paginado, com `id`, `name` e `timezone`. **Não** expõe e-mail nem hash.
- [ ] **CA2.** `GET /providers/:id/services` lista só serviços `active = true` desse profissional.
- [ ] **CA3.** `:id` de alguém que não é profissional, ou inexistente, retorna **404**.
- [ ] **CA4.** Aceita `?search=` para filtrar profissionais pelo nome (sem diferenciar maiúsculas de minúsculas).

---

# Epic 3 · Disponibilidade (M3)

### HU07 · Definir meu horário semanal

**Como** profissional, **quero** informar em que dias e horários atendo, **para** que clientes só vejam horários em que estou disponível.
`Must · M`

**Critérios de aceitação**

- [ ] **CA1.** `PUT /availability` recebe a lista completa de intervalos `{weekday, startTime, endTime}` e **substitui** a anterior (operação atômica: ou troca tudo, ou nada).
- [ ] **CA2.** `weekday` vai de 0 (domingo) a 6; horários no formato `HH:mm`, em minutos múltiplos de 15; `endTime` > `startTime`. Violações retornam **400**.
- [ ] **CA3.** Intervalos do **mesmo dia** não podem se sobrepor (**400**). Pode haver vários intervalos no mesmo dia (ex.: manhã e tarde, com almoço no meio).
- [ ] **CA4.** Lista vazia é válida e significa "não atendo".
- [ ] **CA5.** `GET /availability` devolve a configuração atual, ordenada por dia e horário.
- [ ] **CA6.** Trocar a disponibilidade **não cancela** agendamentos já marcados.

💡 **Estudar:** como detectar sobreposição de intervalos (ordenar por início e comparar vizinhos); transação.

### HU08 · Bloquear períodos

**Como** profissional, **quero** bloquear datas e horários específicos, **para** não receber agendamentos nesses períodos.
`Must · M`

**Critérios de aceitação**

- [ ] **CA1.** `POST /time-offs` com `startsAt`, `endsAt` (instantes ISO 8601) e `reason` opcional cria o bloqueio (**201**).
- [ ] **CA2.** `endsAt` precisa ser depois de `startsAt` (**400**).
- [ ] **CA3.** Se o bloqueio sobrepõe agendamentos `SCHEDULED`, retorna **409** `TIME_OFF_CONFLICT` com a lista dos agendamentos conflitantes (RN10).
- [ ] **CA4.** `GET /time-offs` lista os meus bloqueios futuros, paginado. `DELETE /time-offs/:id` remove um meu; de outro profissional retorna **404**.
- [ ] **CA5.** Horários bloqueados deixam de aparecer na consulta de slots (HU09).

### HU09 · Consultar horários livres ⭐ _coração do sistema_

**Como** cliente, **quero** ver os horários livres de um profissional para um serviço em uma data, **para** escolher quando agendar.
`Must · G`

`GET /providers/:id/slots?serviceId=<uuid>&date=2026-12-01` (sem login). A data é o **dia local do profissional** (RN01).

**Critérios de aceitação**

- [ ] **CA1.** Retorna a lista de `{startsAt, endsAt}` em UTC, ordenada, mais o `timezone` do profissional.
- [ ] **CA2.** Só aparecem inícios na **grade de 15 minutos** em que `[início, início + duração)` cabe inteiro em um intervalo de disponibilidade daquele dia da semana (RN02, RN03a).
- [ ] **CA3.** Horários que sobrepõem um agendamento `SCHEDULED` ou um bloqueio **não** aparecem. Agendamentos `CANCELED` não bloqueiam nada (RN03b, c).
- [ ] **CA4.** Horários com menos de 60 minutos de antecedência e datas além de 60 dias à frente **não** aparecem (RN03d, e).
- [ ] **CA5.** Agendamentos **colados** são permitidos: se há agendamento terminando às 10:00, um slot às 10:00 é válido (RN04).
- [ ] **CA6.** `date` inválida ou fora da janela de 60 dias retorna **400**; `serviceId` inexistente, desativado ou de **outro profissional** retorna **404**.
- [ ] **CA7.** Dia sem disponibilidade devolve lista vazia (**200**), não erro.

**Exemplo para testar** (disponibilidade 09:00–12:00, serviço de 60 min, já existe agendamento 10:00–11:00):

| Início | Cabe?                | Sobrepõe?         | Slot? |
| ------ | -------------------- | ----------------- | ----- |
| 09:00  | sim (até 10:00)      | não               | ✅    |
| 09:15  | sim (até 10:15)      | sim (10:00–10:15) | ❌    |
| 09:45  | sim (até 10:45)      | sim               | ❌    |
| 10:00  | sim                  | sim               | ❌    |
| 11:00  | sim (até 12:00)      | não (colado)      | ✅    |
| 11:15  | não (passa de 12:00) | n/a               | ❌    |

💡 **Como abordar:** escreva primeiro a função **pura** `calcularSlots({ intervalos, agendamentos, bloqueios, duracaoMin, dataLocal, timezone, agora })` em `domain/slots.ts`, **sem banco**, e cubra com testes em tabela. Só depois ligue na rota.
💡 **Estudar:** biblioteca de datas com fuso (**Luxon** ou **date-fns-tz**; evite `Date` puro), "dia local" × instante UTC, intervalos semiabertos.
⚠️ **Armadilhas:** (1) o dia local do profissional não é o dia UTC; 23:00 em São Paulo já é "amanhã" em UTC. (2) Mudança de horário de verão: "09:00 local" pode não existir ou existir duas vezes em algum dia do ano, em fusos que têm horário de verão. Faça pelo menos um teste com `America/New_York`. (3) Nunca use `new Date()` dentro da função: receba `agora` como parâmetro, senão o teste não é reproduzível.

---

# Epic 4 · Agendamentos (M4)

### HU10 · Agendar um horário ⭐

**Como** cliente, **quero** reservar um slot livre, **para** garantir meu atendimento.
`Must · G`

`POST /appointments` com `{ serviceId, startsAt }`. O profissional vem do serviço.

**Critérios de aceitação**

- [ ] **CA1.** Cliente logado com um `startsAt` que seja um slot livre recebe **201** com o agendamento: `status = SCHEDULED`, `endsAt = startsAt + duração`, e **cópia** da duração e do preço do serviço (RN11).
- [ ] **CA2.** O servidor **recalcula** a validade do horário (reutilizando a lógica da HU09); nunca confia no que o front mostrou.
- [ ] **CA3.** Horário **fora da disponibilidade**, fora da grade, no passado ou com menos de 60 min de antecedência retorna **422** `INVALID_SLOT`.
- [ ] **CA4.** Horário **já ocupado** retorna **409** `SLOT_UNAVAILABLE`.
- [ ] **CA5.** Serviço desativado retorna **422** `SERVICE_INACTIVE`; serviço inexistente retorna **404**.
- [ ] **CA6.** **Concorrência:** duas requisições simultâneas para o mesmo horário resultam em exatamente **um** 201 e **um** 409. Existe um teste automatizado que dispara as duas ao mesmo tempo (`Promise.all`).
- [ ] **CA7.** Profissional tentando agendar recebe **403**. Sem login, **401**.
- [ ] **CA8.** Editar o serviço depois **não** altera a duração nem o preço registrados neste agendamento.

💡 **Estudar:** transações, condição de corrida (_race condition_), **exclusion constraint** do PostgreSQL.
💡 **Como garantir o CA6 (do mais robusto ao mais simples):**

1. **Constraint no banco** (recomendado): extensão `btree_gist` e `EXCLUDE USING gist (provider_id WITH =, tstzrange(starts_at, ends_at) WITH &&) WHERE (status = 'SCHEDULED')`. No Prisma isso entra como SQL manual na migration (ajuste os nomes das colunas ao que o Prisma gerar, ou use `@map`). O banco recusa o segundo insert, e você traduz o erro para **409**.
2. Transação com trava (`SELECT ... FOR UPDATE` no profissional) ou _advisory lock_.
   ⚠️ **Armadilha:** "verificar se está livre e depois inserir" **sem** nenhuma das duas proteções passa nos testes simples e falha com dois usuários ao mesmo tempo. Por isso o CA6 existe.

### HU11 · Meus agendamentos (cliente)

**Como** cliente, **quero** ver meus agendamentos, **para** saber o que tenho marcado.
`Must · P`

**Critérios de aceitação**

- [ ] **CA1.** `GET /appointments` como cliente retorna **só os meus**, paginado, ordenado por `startsAt`.
- [ ] **CA2.** Filtros opcionais: `?status=`, `?from=` e `?to=` (instantes ISO 8601). `from` maior que `to` retorna **400**.
- [ ] **CA3.** Cada item traz nome do serviço e do profissional (sem dados sensíveis).
- [ ] **CA4.** `GET /appointments/:id` de agendamento de **outra pessoa** retorna **404** (RN08).

💡 **Dica:** evite o problema "N+1" (uma consulta por item). Use `include` do Prisma ou um `join`.

### HU12 · Cancelar agendamento

**Como** cliente ou profissional, **quero** cancelar um agendamento, **para** liberar o horário.
`Must · M`

**Critérios de aceitação**

- [ ] **CA1.** `POST /appointments/:id/cancel` (com `reason` opcional) muda o status para `CANCELED` e registra `canceledAt`, `canceledBy` e o motivo (**200**).
- [ ] **CA2.** **Cliente** só cancela com pelo menos 24 h de antecedência; depois disso, **422** `CANCELLATION_WINDOW_PASSED` (RN06).
- [ ] **CA3.** **Profissional** dono do agendamento cancela a qualquer momento (antes de terminar).
- [ ] **CA4.** Cancelar agendamento que não está `SCHEDULED` retorna **409** `INVALID_STATUS` (RN05).
- [ ] **CA5.** Agendamento de outra pessoa retorna **404**.
- [ ] **CA6.** Depois de cancelar, o horário volta a aparecer na HU09 e pode ser reservado de novo.

💡 **Dica:** isole a regra "pode cancelar?" em `domain/cancelamento.ts` como função pura recebendo `agora`, `inicio` e `quemCancela`. Teste os limites: 24 h exatas, 23 h 59 min.

### HU13 · Minha agenda (profissional)

**Como** profissional, **quero** ver minha agenda de um período, **para** me preparar para os atendimentos.
`Must · P`

**Critérios de aceitação**

- [ ] **CA1.** `GET /appointments?from=&to=` como profissional retorna os agendamentos **da minha agenda**, ordenados por `startsAt`.
- [ ] **CA2.** Sem `from` e `to`, o padrão é **hoje**, considerando o dia no **meu fuso**.
- [ ] **CA3.** Intervalo maior que 31 dias retorna **400**.
- [ ] **CA4.** Cada item traz nome do cliente, serviço, horário e status. Filtro opcional `?status=`.
- [ ] **CA5.** Nunca aparecem agendamentos de outro profissional.

### HU14 · Concluir ou marcar falta

**Como** profissional, **quero** registrar se o atendimento aconteceu ou se o cliente faltou, **para** manter o histórico correto.
`Must · P`

**Critérios de aceitação**

- [ ] **CA1.** `POST /appointments/:id/complete` e `POST /appointments/:id/no-show` mudam o status para `COMPLETED` e `NO_SHOW` (**200**).
- [ ] **CA2.** Só o profissional dono do agendamento pode (cliente recebe **403**; profissional de outra agenda recebe **404**).
- [ ] **CA3.** Só é possível **depois do início** do horário; antes disso, **422**.
- [ ] **CA4.** Só a partir de `SCHEDULED`; qualquer outro status retorna **409** `INVALID_STATUS` (RN05).

💡 **Dica:** centralize as transições válidas em um único lugar (`domain/status.ts`), por exemplo um mapa `{ SCHEDULED: ['CANCELED','COMPLETED','NO_SHOW'], ... }`. As HU12 e HU14 passam a usá-lo.

### HU15 · Remarcar

**Como** cliente, **quero** mudar o horário de um agendamento, **para** não precisar cancelar e perder a vaga caso o novo horário esteja ocupado.
`Should · M`

**Critérios de aceitação**

- [ ] **CA1.** `POST /appointments/:id/reschedule` com `{ startsAt }` aplica as **mesmas validações** da HU10 (CA2 a CA4) e retorna **200** com o agendamento atualizado (mesmo `id`).
- [ ] **CA2.** Se o novo horário falhar (ocupado, inválido), o agendamento **original continua exatamente como estava**.
- [ ] **CA3.** Só vale para agendamento `SCHEDULED`; cliente respeita o prazo de 24 h da RN06 (**422**).
- [ ] **CA4.** O próprio agendamento **não conta como conflito consigo mesmo**: mover de 10:00 para 10:15 (sobreposição parcial com o horário atual) funciona.
- [ ] **CA5.** O profissional também pode remarcar; agendamento de outra pessoa retorna **404**.

⚠️ **Armadilha:** o CA4 é o erro mais comum. Se a consulta de conflitos não **exclui o agendamento que está sendo movido**, o sistema acha que 10:15 está ocupado pelo próprio cliente. Se você usar a _exclusion constraint_ da HU10, o `UPDATE` do horário já é protegido pelo banco.

---

# Epic 5 · Extras (M5)

### HU16 · E-mails de confirmação

**Como** cliente, **quero** receber um e-mail quando agendar, remarcar ou cancelar, **para** ter o registro.
`Should · M`

- [ ] **CA1.** Agendar, remarcar e cancelar disparam e-mail ao cliente (e ao profissional, no caso de cancelamento pelo cliente), com serviço, data e hora **no fuso de quem recebe**.
- [ ] **CA2.** O envio fica atrás de uma interface `Mailer`; em desenvolvimento usa um servidor de e-mail falso (Mailpit em Docker); nos testes usa um _fake_ em memória.
- [ ] **CA3.** **Falha ao enviar e-mail não derruba a operação**: o agendamento continua criado; o erro vai para o log.
- [ ] **CA4.** Os testes verificam o conteúdo do e-mail sem enviar nada de verdade.

💡 **Estudar:** injeção de dependência (passar o `Mailer` em vez de importá-lo direto), _fake_ × _mock_.

### HU17 · Lembrete 24 h antes

**Como** cliente, **quero** receber um lembrete um dia antes, **para** não esquecer.
`Could · G`

- [ ] **CA1.** Um job periódico envia lembrete para agendamentos `SCHEDULED` que começam em cerca de 24 h.
- [ ] **CA2.** **Idempotente**: cada agendamento recebe **no máximo um** lembrete, mesmo se o job rodar duas vezes (campo `reminderSentAt`).
- [ ] **CA3.** Agendamentos cancelados ou já passados são ignorados.
- [ ] **CA4.** A lógica de "quais agendamentos precisam de lembrete" é uma função testável, separada do agendador.

💡 **Estudar:** `node-cron` (simples) ou filas com BullMQ + Redis (robusto), idempotência.

### HU18 · Cliente sem conflito _(RN12)_

`Could · P`

- [ ] **CA1.** Cliente que já tem um agendamento `SCHEDULED` sobreposto (com qualquer profissional) recebe **409** `CUSTOMER_BUSY` ao agendar ou remarcar.
- [ ] **CA2.** Agendamentos colados (um termina quando o outro começa) são permitidos.

### HU19 · Documentação da API (OpenAPI)

`Should · M`

- [ ] **CA1.** `/docs` mostra uma interface Swagger com todas as rotas, corpos, respostas e códigos de erro.
- [ ] **CA2.** Os schemas vêm dos mesmos schemas Zod usados na validação (uma fonte só da verdade).
- [ ] **CA3.** Dá para fazer login e testar rotas protegidas pelo botão **Authorize**.

### HU20 · Deploy

`Should · M`

- [ ] **CA1.** Imagem Docker de produção (multi-stage, usuário sem root) sobe com um comando.
- [ ] **CA2.** Migrations rodam no deploy; segredos vêm de variáveis de ambiente, nunca do repositório.
- [ ] **CA3.** Health check configurado; a API publicada responde em uma URL pública, que está no README.
- [ ] **CA4.** Deploy automático a cada merge na `main` (se a plataforma suportar).

💡 **Dica:** o processo é o mesmo do encurtador (Render + Postgres). Veja o `render.yaml` e o `Dockerfile` daquele repositório.

---

# Epic 6 · Front-end (M6, opcional)

### HU21 · Interface web

**Como** cliente e profissional, **quero** usar o sistema por uma interface, **para** não depender de chamadas manuais à API.
`Could · G`

- [ ] **CA1.** Telas: cadastro, login, catálogo de profissionais, escolha de serviço → **data → horário** → confirmação, meus agendamentos (com cancelar) e, para o profissional, agenda e configuração de horários.
- [ ] **CA2.** Horários exibidos no fuso **do navegador do usuário**, com o fuso do profissional indicado quando for diferente.
- [ ] **CA3.** Erros da API (409, 422...) viram mensagens claras para a pessoa, não códigos.
- [ ] **CA4.** Funciona em celular.

💡 **Sugestão:** React + Vite + TypeScript, `fetch` ou TanStack Query. Gere os tipos a partir do OpenAPI da HU19 para o front e o back nunca divergirem.

---

# Definição de pronto

Uma HU só vai para "feito" quando:

- [ ] Todos os critérios de aceitação passam em **testes automatizados** (unitários para regras puras, integração para as rotas).
- [ ] `lint`, `typecheck` e `test` passam localmente **e no CI**.
- [ ] Não há `any` nem `@ts-ignore` sem comentário explicando o motivo.
- [ ] Erros seguem o formato padrão e usam os códigos do [DOMINIO.md](DOMINIO.md).
- [ ] Entrada de toda rota é validada (Zod); nada de dado do cliente chega ao banco sem validação.
- [ ] README e DOMINIO foram atualizados se algo mudou.
- [ ] Os commits são pequenos, com mensagens claras (`feat:`, `fix:`, `test:`, `docs:`, `chore:`), e a HU entrou por Pull Request.

# Como organizar o trabalho

- **Uma branch por HU**: `feat/hu09-slots`, `feat/hu10-agendar`. Um PR por HU, com o número da HU no título.
- **Issues no GitHub:** uma por HU, copiando os CA como checklist. Use um **Project** (quadro Kanban) com as colunas _Backlog → Fazendo → Em revisão → Feito_.
- **Teste primeiro nas regras difíceis** (HU09, HU12, HU15): escreva os casos da tabela de exemplo como testes, veja falhar, depois implemente.
- **Travou por mais de 1 hora?** Veja a seção "Como pedir ajuda" no [README](../README.md).
