# Sistema de agendamento (TypeScript)

API para profissionais publicarem serviços e horários de atendimento, e clientes marcarem, remarcarem e cancelarem horários.

> **Status:** planejamento. Ainda não há código. Os documentos abaixo definem o que construir; a implementação é feita pelo autor, HU por HU.

## Documentação

| Documento | Para que serve |
|---|---|
| [docs/DOMINIO.md](docs/DOMINIO.md) | Glossário, **regras de negócio (RN01–RN12)**, modelo de dados, contrato da API e estrutura de pastas. |
| [docs/BACKLOG.md](docs/BACKLOG.md) | **Tarefas técnicas (T01–T05) e histórias de usuário (HU01–HU21)**, cada uma com critérios de aceitação, dicas e armadilhas. |

**Por onde começar:** leia o DOMINIO.md uma vez, depois siga o BACKLOG na ordem: T01 → T05, HU01 → HU03, e assim por diante.

## Stack sugerida

| Camada | Escolha | Por quê |
|---|---|---|
| Linguagem | **TypeScript** (`strict`) + Node.js (LTS) | Tipos pegam erros antes de rodar. |
| HTTP | **Express** (ou Fastify) | O mais conhecido; muito material para estudar. |
| Validação | **Zod** | Valida em tempo de execução e gera os tipos TypeScript. |
| Banco | **PostgreSQL** | Datas com fuso (`timestamptz`) e *exclusion constraints* que impedem overbooking. |
| Acesso ao banco | **Prisma** (ou Drizzle) | Migrations e tipos gerados do schema. |
| Datas e fusos | **Luxon** (ou date-fns-tz) | O `Date` nativo é fraco para fusos horários. |
| Autenticação | **JWT** + hash **argon2** ou **bcrypt** | Mesmo conceito do projeto do encurtador, agora em TypeScript. |
| Testes | **Vitest** + **Supertest** | Rápidos e com boa integração com TypeScript. |
| Infra | **Docker Compose**, **GitHub Actions** | Banco local sem instalar nada; CI desde o primeiro dia. |

Se preferir outra opção (NestJS, Drizzle, Jest...), tudo bem. Os critérios de aceitação **não dependem de ferramenta**.

## O que este projeto demonstra

- **Regras de negócio reais**: disponibilidade, conflitos, prazos de cancelamento, máquina de estados.
- **Fusos horários** tratados corretamente (um erro clássico de iniciantes).
- **Concorrência**: dois clientes disputando o mesmo horário, resolvido no banco.
- **Código testável**: regras puras em `domain/`, separadas de HTTP e de banco.
- **Engenharia**: CI, migrations, Docker, PRs pequenos.

## Como pedir ajuda

Para eu (ou qualquer pessoa) conseguir te ajudar rápido, traga:

1. **Qual HU ou tarefa** (ex.: "HU09, CA4").
2. **O que você esperava** e **o que aconteceu**.
3. **O erro completo**, copiado do terminal (não só a última linha).
4. **O trecho de código** envolvido, ou o caminho do arquivo.
5. **O que você já tentou.**

Exemplos de boas perguntas:

- "HU09, CA2: meu teste da tabela de exemplo devolve 09:15 como slot livre, mas devia estar ocupado. Aqui está `domain/slots.ts`..."
- "HU10, CA6: o teste de concorrência passa às vezes e falha às vezes. Como a constraint deveria ficar na migration?"
- "Não entendi a diferença entre 401 e 403 no CA4 da HU02."

Dúvidas de **conceito** (JWT, transação, fuso horário, exclusion constraint) também valem; os blocos "💡 Estudar" do backlog listam os temas de cada HU.

## Licença

A definir.
