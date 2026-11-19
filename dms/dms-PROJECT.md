# DMS

## Contexto e objetivo

- Solução **genérica e reutilizável** para migrar bases SQL Server on-premises para **Amazon RDS for PostgreSQL** ou **Amazon Aurora PostgreSQL** (gerenciado). Backup, alta disponibilidade e patch passam para a AWS, e a licença de SQL Server deixa de existir.
- Neste documento, "RDS" designa o destino PostgreSQL gerenciado, seja RDS for PostgreSQL ou Aurora PostgreSQL.
- **Duas rotas no mesmo ferramental (D21):**
  - **rota devops:** bases com app modernizada; app e base viram juntas;
  - **rota tradops:** bases com app ainda não modernizada; só a base vira para a conta tradops, e as apps on-prem passam a acessá-la via DX.
- **Caso de referência:** a base **SQNPRD** (bancos `db1` e `db2`).
- Atender a versão modernizada da aplicação, que já roda na AWS.
- Adotar o modelo de dados corporativo. Exemplo: `cliente_contato` → `pessoa` + `contato`; `id_cliente_contato` → `id_pessoa`.
- **Meta de portfólio:** 1.200 bases SQL Server migradas até o início de 2028. A execução acontece ao longo de 2027 e migra a maior quantidade possível, conforme o investimento escolhido pelo cliente (D17). Hipótese: "migrada" = virada concluída até o início de 2028.

## Escopo

- **Dentro:**
  - conversão do schema e migração dos dados de cada base SQL Server (schema `dbo`) para o RDS for PostgreSQL (no caso de referência, `db1` e `db2` vão para o schema `schemaapp`);
  - migração para a conta tradops como rota opcional, nas bases cujas apps ainda não modernizaram (D21);
  - replicação contínua (CDC) durante a convivência;
  - virada e janela de fallback com CDC reverso;
  - desligamento do on-prem;
  - normalização para o modelo corporativo.
- **Processo em 2 passos**, executados pelo time de migração:
  1. transformação de schema no **schema-forge**;
  2. migração de dados pelos **data-devops-pipelines**.
- **Fora:**
  - democratização dos dados (`batch-democratizacao` → datamesh S3 sor/sot/spec): rotina de responsabilidade da área de dados da sigla, fora do escopo do projeto.
- **A confirmar:** tratamento dos consumidores ocultos da base (jobs, relatórios, integrações, linked servers, SSIS), que depende de inventário.

## Requisitos-chave

- **Guardrail (conta devops):** as apps on-premises não acessam a base na conta devops. Por isso a app e a base viram juntas, com troca no DNS, na entrada de tráfego ou por toggles. Na conta tradops, o acesso direto das apps on-prem à base é o propósito do ambiente (D21).
- **Limites do DMS:** o DMS só copia tabela → tabela, coluna → coluna e linha a linha. Ele aceita renomear, mudar tipo, aplicar fórmula com dados da mesma linha e filtrar. Ele não divide tabelas, não transforma colunas em linhas, não deduplica, não faz lookup em outra tabela e não junta tabelas. As transformações estruturais exigem scripts, o STM (source-to-target mapping) e a tabela de cross-reference de chaves `map_pessoa`.
- **Replicação contínua:** o atraso deve ser de segundos e a replicação deve se manter por dias ou semanas enquanto a app nova é testada.
- **Equivalência de comportamento** entre SQL Server e PostgreSQL: collation, maiúsculas/minúsculas, fuso horário e SQL embutido nas apps.

## AS-IS

Caso de referência:

- **Base:** SQNPRD em SQL Server, no itaú-datacenter. Contém os bancos `db1` e `db2` e o log de transações (insert/update/delete).
- **Apps legadas on-prem:** APIs, batch, integrações e o `middleware-consumer`. Todas leem e gravam no SQNPRD.
- **Conectividade:** Direct Connect (dx) entre o itaú-datacenter e a AWS.
- **Consumidores ocultos:** possíveis jobs, relatórios, integrações, linked servers e SSIS, ainda não inventariados.

## TO-BE

### Ferramental de migração
- **schema-forge (passo 1):** pipeline de modelagem que gera o STM e aciona os agentes de IA/automação (schema-forge agents). Exemplos do STM: `id-cli → id-pessoa`, `nm-cli → nome`, `address → ende`.
- **data-devops-pipelines (passo 2):** fluxo "Migração On-Premises para AWS RDS com DMS" em 8 etapas, da criação da base ao monitoramento. Inclui o mapeamento das particularidades encontradas no processo, feito pelo analista Alexander Ascar. Os templates de criação da base cobrem os dois motores (D20); no Aurora, incluem o parameter group de cluster com logical replication ligada para o CDC reverso.
- **Breaking glass:** acesso direto do time ao ambiente de migração para ajustes críticos pontuais.

### Conta-target AWS
- **RDS for PostgreSQL Multi-AZ ou Aurora PostgreSQL (D20)**, schema `schemaapp` (`db1`, `db2`).
- **Task AWS DMS `full-load-and-cdc`:**
  - lê as tabelas e o log do SQL Server via DX, passando pela conta transitiva de firewall & data-security rules;
  - grava o full load (`rds-fullload`) e aplica o CDC no RDS.
- **CDC reverso (RDS → SQL Server):** ligação direta do DMS para a base on-prem, sem passar pela conta transitiva.
- **Monitoramento e segurança/segredos** ligados à task do DMS (a confirmar).
- **Apps modernizadas:** leem e gravam no RDS e publicam ou consomem via **kaas-convivencia**.
- **Caminhos de convivência:**
  - apps on-prem → API Gateway → Lambda `middleware-dt-handler` → RDS;
  - kaas-convivencia → `middleware-consumer` on-prem.
- **Democratização (fora do escopo):** o `batch-democratizacao` lê o RDS e publica no **datamesh** (S3, camadas sor/sot/spec). É uma rotina sob responsabilidade da área de dados da sigla e aparece no desenho só como contexto.

### Conta tradops (rota tradops, D21)
- **Destino intermediário opcional:** RDS for PostgreSQL ou Aurora PostgreSQL. Com Babelfish, o motor é obrigatoriamente Aurora (D22). As apps legadas on-prem acessam a base diretamente via DX.
- **App legada:** usa Babelfish ou é convertida para PostgreSQL, conforme decisão por base (D22).
- **Virada:** só a base vira. Hipótese: congelar a escrita e repontar a conexão das apps legadas (DNS ou connection string), sem o caminho API Gateway → Lambda `middleware-dt-handler`.
- **Hipótese:** com Babelfish em modo multi-db, `db1.dbo` vira `db1_dbo` no PostgreSQL, e não `schemaapp`.
- **Hipótese:** a conversão é 1:1, e a normalização (D3) fica para a passagem tradops → devops, quando a app modernizar.
- **Hipótese:** os passos 1 e 2, a janela de fallback com CDC reverso (D5) e o portão único (D19) valem igual nas duas rotas.

### Controle de tráfego
- **Portal Manager** com toggles `read-cloud` / `write-cloud`. Ele pausa a escrita das apps on-prem e ativa a stack AWS na virada e no rollback.

### Equipe
- **Time de migração:** DBAs e time da app. Executa os passos 1 e 2 e opera a virada e os fluxos de exceção.
- A composição do time base, a organização em clusters, o modelo de apoio às siglas e o apoio dos analistas Itaú estão em "Entrega e migração".

## Decisões

- **D1 — Migrar para PostgreSQL gerenciado: Amazon RDS for PostgreSQL Multi-AZ ou Amazon Aurora PostgreSQL (com réplica em outra AZ).** Status: decidida. Motivo: transfere backup, alta disponibilidade e patch para a AWS e elimina a licença de SQL Server.
- **D2 — Usar AWS DMS full load + CDC, rodando na AWS e conectando ao on-prem via DX/VPN.** Status: decidida. Motivo: cópia online com atraso de segundos e convivência validada pelo próprio DMS.
- **D3 — Converter o schema 1:1 na migração e normalizar para o modelo corporativo na cloud após desligar o on-prem (expand/contract).** Status: proposta. Motivo: é o caminho mais barato e com rollback simples. Condição: na virada, a app nova grava no modelo antigo por meio de uma anticorruption layer.
- **D4 — Alternativa à D3: normalizar durante a migração (`stg` → `corp` por SQL/dbt; volta `corp` → `rev` por trigger → DMS).** Status: proposta. Só se aplica se a app nova falar apenas o modelo novo. É mais cara e tem rollback mais complexo.
- **D5 — Manter uma janela de fallback com CDC reverso (RDS → SQL Server).** Status: decidida. As apps legadas ficam implantadas e sem tráfego. Regra: dado errado ou causa desconhecida → rollback; erro de app com correção conhecida → correção na AWS dentro do prazo (breaking glass).
- **D6 — O CDC reverso liga o DMS diretamente à base on-prem, sem passar pela conta transitiva.** Status: decidida.
- **D7 — Estimar pelo modelo de portes P/M/G (sem GG). Quando a base fica entre portes, vale o porte da maior dimensão (GB, procs ou apps).** Status: decidida (modelo); proposta (regra de enquadramento). Motivo: o modelo dá uma estimativa padronizada, e a regra pela maior dimensão evita subestimar.
- **D8 — A solução é genérica e reutilizável, e não específica de uma base. O SQNPRD é o caso de referência.** Status: decidida. Motivo: permitir o uso em diferentes bases.
- **D9 — Time base por base com 3 perfis: 1 DBA ou engenheiro de dados, 1 dev da app e 1 QA.** Status: decidida. Motivo: cobre as 4 etapas do modelo com um responsável por etapa.
- **D10 — Organizar os profissionais em três clusters: plataforma central, squads de migração (um time base por base, nas siglas sem capacidade própria) e célula de fallback. Nas siglas com capacidade, o time atua com hands-on, apoio e virada (D18).** Status: decidida. Motivo: o squad é a unidade de escala, porque cada squad adicionado acrescenta uma base em paralelo, e a célula libera o squad ao fim do hypercare.
- **D11 — Tratar os analistas Itaú como apoio por base, fora dos squads: um ponto focal Itaú por base, com agenda reservada no kick-off da base e esforço estimado em linha própria por etapa, sob compromisso formal com o cliente (D13).** Status: proposta. Motivo: a homologação, o go/no-go e o teste acompanhado dependem deles. Sem agenda reservada, o ciclo do squad aumenta e o paralelismo se perde.
- **D12 — Etapa 0 de habilitação por base, executada pela plataforma central antes do kick-off do squad.** Status: proposta. Entrada: motor escolhido (D20) e, na rota tradops, a decisão da D22 tomada (ela define o motor); sem essas definições, a habilitação da base não começa. Escopo: RDS e DMS provisionados, regras da conta transitiva, rota AWS → on-prem (porta 1433) para o CDC reverso, segredos e acessos, inventário de consumidores ocultos e janelas de mudança em prod reservadas. Motivo: esses lead times não estão no modelo e, se caírem no ciclo do squad, invalidam os prazos e a capacidade.
- **D13 — O apoio Itaú é um compromisso formal com o cliente.** Status: decidida. A agenda do ponto focal e o prazo de aceite são premissa do plano de entrega. Se a espera passar do prazo de aceite, o prazo da base fica suspenso e o impacto é registrado no calendário de viradas. Motivo: os prazos das etapas 3 e 4 dependem desse apoio.
- **D14 — Triagem automatizada do portfólio no T1/2027, em paralelo ao piloto SQNPRD.** Status: decidida. Coleta por script dos metadados das bases (GB, procs, acessos dos últimos 90 dias, linked servers, SSIS) para enquadrar cada base em P/M/G, identificar as bases a aposentar ou consolidar e montar o grafo de dependências. Inclui a captura do SQL embutido das apps legadas como insumo da D22 (hipótese: extração do código ou captura em execução por trace/Extended Events). Hipótese: a triagem indica a rota pelo estado da app (app modernizada → devops; app legada → tradops), com decisão final do cliente ou da sigla, como na D20.
- **D15 — Unidade de virada e plano de ondas.** Status: proposta. Como app e base viram juntas (guardrail), o grafo app × base × consumidores ocultos define os grupos que viram juntos. As ondas começam pelos grupos P isolados. Hipótese: na rota tradops, a unidade de virada é a base, e o grafo só restringe pelo que não alcança a tradops (linked servers, SSIS, consumidores fora do DX).
- **D16 — Janela de fallback por porte (ex.: 1 semana no P; 2–4 semanas no M e no G).** Status: proposta.
- **D17 — O nível de investimento (quantos squads alocar) é decisão do cliente.** Status: decidida. A rampa parte do time base atual, já montado.
- **D18 — Modelo de apoio por sigla, com dois formatos.** Status: decidida. (1) A sigla tem capacidade (ex.: ff6): fazemos o hands-on do schema-forge (a ferramenta de conversão) e apoiamos o time da sigla, que converte, testa e valida em dev e hml, e atuamos junto na virada. (2) A sigla não tem capacidade: alocamos um squad base (D9) para executar a migração.
- **D19 — Portão único antes da virada, igual para todas as siglas.** Status: proposta. Critérios de aceite padronizados, reconciliação automática de dados (contagem e checksum) e checklist de go/no-go.
- **D20 — O motor do destino (RDS for PostgreSQL ou Aurora PostgreSQL) é escolhido caso a caso, por decisão do cliente ou da sigla.** Status: decidida.
- **D21 — A conta tradops (AWS) é destino intermediário opcional para as bases cujas apps ainda não modernizaram.** Status: decidida. Na tradops, só a base vira e as apps on-prem passam a acessar o RDS via DX. As bases com app modernizada seguem a rota devops (virada conjunta app + base). O SQNPRD segue a rota devops. Motivo: migrar a base sem esperar a modernização da app.
- **D22 — Na rota tradops, a escolha entre Babelfish e conversão da app legada para PostgreSQL é feita por base, na triagem (D14), antes da Etapa 0 (D12).** Status: decidida. O Babelfish só existe no Aurora PostgreSQL: uma base com Babelfish tem Aurora como motor, e a D20 fica restrita nesse caso. Motivo: a escolha define o motor da base.

## Riscos e restrições

| # | Risco | Mitigação |
|---|---|---|
| R1 | Diferenças de comportamento SQL Server × PostgreSQL (collation, maiúsculas, fuso, SQL embutido) | Testes de equivalência durante a replicação contínua |
| R2 | O DMS não migra o valor das sequences | `setval` na virada |
| R3 | O slot do CDC reverso retém WAL | Alarmes em `OldestReplicationSlotLag` e `TransactionLogsDiskUsage`; remover o slot no fim da janela; no Aurora, confirmar as métricas de slot e WAL, já que lá o WAL retido vira custo de storage, e não disco cheio |
| R4 | Os triggers do legado disparam com as gravações do DMS | Desligar os triggers durante o fallback |
| R5 | Consumidores ocultos (jobs, relatórios, integrações, linked servers, SSIS) | Inventário antes da virada (Etapa 0, D12), que também é entrada do plano de ondas (D14, D15) |
| R6 | Transformações estruturais fora do alcance do DMS | Scripts + STM + `map_pessoa` |
| R7 | O modelo de estimativa não cobre bases com dependências cross-db (critério que só existia no GG) | Levantar as queries, procs e views cross-db ao enquadrar cada base e estimar esse esforço fora do modelo |
| R8 | O total do modelo vai até a virada e o hypercare termina antes da janela de fallback (2–4 semanas). Ficam fora da estimativa o desligamento do on-prem, a normalização (D3) e a passagem tradops → devops (PostgreSQL → PostgreSQL + normalização) | Planejar o desligamento ao fim da janela de fallback, e não ao fim do hypercare |
| R9 | O DBA / eng. de dados é o gargalo: atua nas etapas 1 e 4 e fica preso 2–4 semanas por base na janela de fallback, o que quase dobra o ciclo de um squad em bases M | Célula de fallback (D10) assume as bases após o hypercare; não sobrepor viradas de bases sob o mesmo DBA |
| R10 | O paralelismo de squads esbarra em recursos compartilhados: banda do DX e instâncias DMS em full loads simultâneos, conta transitiva, viradas concorrentes e capacidade da célula de fallback. Na escala do portfólio, há muitas tasks DMS simultâneas, de replicação e de CDC reverso | Escalonar full loads e viradas em calendário; dimensionar a plataforma central e a célula pelo nº de squads ativos; confirmar as cotas de DMS e a banda do DX antes do início da escala |
| R11 | A disponibilidade dos analistas Itaú limita as etapas 3 e 4 e o nº de squads em paralelo. Esperar pela homologação ou pelo aceite aumenta o ciclo do squad além do modelo | Compromisso formal com o cliente (D13), com suspensão do prazo da base quando a espera passar do prazo de aceite; agenda reservada no kick-off da base (D11); roteiro e critérios de aceite prontos antes da etapa 3; homologação em lotes; prazo de aceite (hipótese: 2 dias úteis; 1 dia no porte P, cuja etapa 3 dura 2–3 dias); analistas Itaú no calendário de viradas |
| R12 | A etapa 4 (1–2 dias) só cobre a virada. Se o full load de prod, a recuperação do atraso e a validação ficarem para a etapa 4, o prazo estoura | Iniciar a task DMS de prod (full load + CDC) durante a etapa 3, com a replicação contínua validada antes do go/no-go |
| R13 | Lead times de provisionamento, acessos, regras de firewall e janelas de mudança em prod não estão no modelo | Etapa 0 de habilitação pela plataforma central (D12) concluída antes do kick-off do squad |
| R14 | A primeira base de cada rota (SQNPRD na rota devops; a primeira base da rota tradops) amadurece o schema-forge e os pipelines e tende a levar mais que o modelo | Tratar cada uma como piloto, fora do modelo, e recalibrar os prazos P/M/G com o resultado |
| R15 | Nas siglas com capacidade, a migração concorre com a fila de trabalho da própria sigla e pode parar | Compromisso formal da sigla nos moldes da D13; portão único (D19); squad base de reserva |
| R16 | Com o nosso time atuando em todas as viradas, a virada vira o gargalo | Escalonar as viradas em calendário; dimensionar o time e a célula de fallback pelo nº de viradas por semana |
| R17 | O Itaú pode não homologar o Babelfish | Confirmar antes do piloto da rota tradops; sem homologação, converter a app (D22) |
| R18 | Suporte do DMS ao Babelfish, em especial o CDC reverso a partir do Aurora com Babelfish | Confirmar no piloto da rota tradops |
| R19 | Latência via DX para apps on-prem muito "conversadoras" com a base na tradops | Avaliar no piloto da rota tradops |
| R20 | Linked servers e SSIS falam SQL Server e bloqueiam a virada isolada da base na tradops | Identificar na triagem (D14) e tratar no grafo de dependências (D15) |

## Entrega e migração

### Modelo de estimativa

| Etapa | P (até 10 GB, 10 procs, 1–2 apps) | M (40 GB, 30 procs, 3 apps) | G (70 GB, 100 procs, 6 apps) |
|---|---|---|---|
| 0. Habilitação (D12, plataforma central) | Fora do ciclo do squad | Fora do ciclo do squad | Fora do ciclo do squad |
| 1. Migração da base | 1–2 dias | 3–4 dias | 5–7 dias |
| 2. Conversão das apps | 1–2 dias | 3–5 dias | 6–9 dias |
| 3. Testes em dev e hml | 2–3 dias | 5–7 dias | 7–10 dias |
| 4. Migração completa em prod | 1 dia | 1–2 dias | 2 dias |
| Hypercare | 2–3 dias | 1 semana | 1–2 semanas |
| **Total até a virada** | ~1–1,5 semana | ~2,5–3,5 semanas | ~4–5,5 semanas |

- **Hipóteses:** dias úteis, semana de 5 dias e hypercare fora do total. Os totais correspondem a 5–8 dias úteis (P), 12–18 (M) e 20–28 (G). As etapas são somadas em sequência; a sobreposição parcial das etapas 1 e 2 (perfis diferentes) é folga.
- **Hipótese:** os prazos pressupõem o time base (D9) dedicado a uma base. No porte G, os prazos da etapa 1 já consideram +1 DBA / eng. de dados (squad de 4). Com 1 só, a etapa 1 de G vai para ~10–12 dias.
- **Hipótese:** a etapa 4 é só a virada. A task DMS de prod (full load + CDC) é iniciada durante a etapa 3 e fica em replicação contínua até a virada (R12).
- **Hipótese:** os prazos das etapas 3 e 4 pressupõem o apoio Itaú com agenda reservada (D11, D13), sem espera por homologação ou aceite.
- **Hipótese:** o prazo de aceite (R11) está contido na etapa 3. No porte P, isso deixa 1–2 dias de teste acompanhado, e os totais não mudam.
- **Hipótese:** o modelo vale para bases depois do piloto de cada rota (R14).
- **Hipótese:** o modelo vale também para a rota tradops. Nela, a etapa 2 é a conversão ou o ajuste da app legada (D22), e a etapa 4 é a virada só da base.
- **Premissa:** o modelo vale com a D3 (conversão 1:1). Com a D4, as etapas 1 e 3 crescem.
- Cada base é enquadrada em P, M ou G no momento da sua migração (D7).

### Modelo de apoio às siglas (D18)

| Situação da sigla | Nossa atuação |
|---|---|
| Sigla com capacidade | Hands-on do schema-forge + apoio ao time da sigla + virada em conjunto |
| Sigla sem capacidade | Alocação de squad base (D9) |

### Time base (D9)

| Perfil | Responsabilidades | Etapas |
|---|---|---|
| DBA ou engenheiro de dados | Passo 1 (schema-forge, STM, procs), passo 2 (task DMS, full load, CDC), sequences e CDC reverso | 1 e 4 |
| Dev da app | SQL embutido, anticorruption layer (D3), integração com os toggles | 2 e 4 |
| QA | Testes de equivalência (collation, maiúsculas, fuso) em dev e hml | 3 |

### Apoio Itaú por base (D11, D13)

| Etapa | Papel do analista Itaú | Dedicação |
|---|---|---|
| 1. Migração da base | Validar regras de negócio do STM e de `map_pessoa` | Pontual |
| 2. Conversão das apps | Tirar dúvidas de regra da app | Pontual |
| 3. Testes em dev e hml | Homologação e testes acompanhados em hml; aceite formal | Parcial e contínua |
| 4. Migração em prod | Go/no-go, teste acompanhado em prod, validação pós-virada | Integral na janela |
| Hypercare | Validar dados e acionar o critério de rollback ("dado errado", D5) | Parcial |
| Janela de fallback | Validar dados e acionar o critério de rollback ("dado errado", D5) | Pontual |

- Um ponto focal Itaú por base, com agenda reservada no kick-off da base.
- O esforço do apoio Itaú é estimado em linha própria e não entra no esforço do squad.

**Esforço Itaú por base (dias-pessoa)**

| Etapa | P | M | G |
|---|---|---|---|
| 1–2. Migração da base e conversão das apps | ~0,5 d | ~1 d | ~1 d |
| 3. Testes em dev e hml | 1–1,5 d | 2,5–3,5 d | 3,5–5 d |
| 4. Migração em prod | 1 d | 1–2 d | 2 d |
| Hypercare | ~0,5–1 d | ~1,25 d | ~1,25–2,5 d |
| **Total** | ~3–4 d | ~6–8 d | ~8–10,5 d |

- **Hipótese:** dedicação do analista de ~0,5–1 dia nas etapas 1 e 2, 50% na etapa 3, 100% na etapa 4 e 25% no hypercare.
- **Hipótese:** na janela de fallback, ~0,5 dia por semana, ou seja ~1–2 dias-pessoa em uma janela de 2–4 semanas, fora do total acima.

### Clusters de profissionais (D10)

| Cluster | Composição | Escala com | Papel na escala |
|---|---|---|---|
| **Plataforma central** | schema-forge e agentes, data-devops-pipelines, templates de criação da base para os dois motores (D20; no Aurora, parameter group de cluster com logical replication ligada para o CDC reverso), templates de DMS, monitoramento e alarmes, rede DX e conta transitiva, Portal Manager, mapeamento de particularidades, Etapa 0 de habilitação (D12) | nº de squads ativos | Único e compartilhado; não se replica por base |
| **Squads de migração** | 1 time base (D9) por base, nas siglas sem capacidade (D18) | nº de bases em paralelo | Unidade de escala: +1 squad = +1 base em paralelo |
| **Célula de fallback** | DBA(s) de plantão: CDC reverso, slot de WAL, rollback | nº de bases em janela de fallback | Assume a base ao fim do hypercare e libera o squad para a próxima |

### Reforço dentro do squad (bases maiores)

| Etapa | Cresce com | Como reforçar |
|---|---|---|
| 1. Migração da base | nº de procs | +1 DBA / eng. de dados no porte G (já considerado nos prazos de G); a conversão das procs pode ser dividida, o full load não acelera |
| 2. Conversão das apps | nº de apps | +1 dev a cada ~2–3 apps (hipótese); acelera além do modelo, que considera 1 dev (~1–1,5 dia por app) |
| 3. Testes em dev e hml | nº de apps | +QA junto com os devs adicionados |
| 4. Migração em prod | — | Não escala com pessoas |
| Hypercare / fallback | tempo de calendário | Transferir para a célula de fallback ao fim do hypercare |

### Capacidade por squad

| Porte | Ciclo do squad (até o fim do hypercare) | Bases por squad por trimestre (13 semanas) |
|---|---|---|
| P | ~1,5–2 semanas | ~6–9 |
| M | ~3,5–4,5 semanas | ~3 |
| G | ~5–7,5 semanas | ~2 |

- **Hipótese:** o squad inicia a próxima base ao fim do hypercare, e a célula de fallback assume a janela de fallback.
- **Hipótese:** a Etapa 0 (D12) da próxima base termina antes de o squad ficar livre.
- **Hipótese:** cada analista Itaú acompanha no máximo ~2 bases ao mesmo tempo na etapa 3 e não atende duas viradas sobrepostas, porque a etapa 4 exige dedicação integral.
- Capacidade total ≈ nº de squads × bases por squad, limitada pelos recursos compartilhados (R10) e pela disponibilidade dos analistas Itaú (R11).
- Cada base fica 2–4 semanas na célula. Com N squads em bases M, a célula sustenta ~N/2 a N bases em fallback ao mesmo tempo.

### Fluxo da migração
1. **Início:** o DMS marca a posição atual do log do SQL Server.
2. **Cópia inicial (full load):** as apps seguem gravando no legado e o DMS guarda as mudanças.
3. **Recuperar o atraso:** o DMS aplica no RDS o que mudou durante a cópia.
4. **Replicação contínua:** atraso de segundos, por dias ou semanas, enquanto a app nova é testada.
5. **Virada:**
   - congelar a escrita no legado (toggle das apps on-prem);
   - conferir que o atraso está zerado;
   - ajustar as sequences;
   - ligar o CDC reverso;
   - ativar a stack AWS (toggles do Portal Manager e/ou DNS).

Em prod, os passos 1–4 acontecem durante a etapa 3 do modelo; a etapa 4 corresponde ao passo 5.

### Exceções
- **6.1:** breaking glass e correções na AWS.
- **6.2:** rollback com CDC reverso.

### Janela de fallback e rollback
- **Janela de fallback:** por exemplo, 2–4 semanas. Depois dela, o on-prem é desligado e começa a normalização na cloud (D3).
- **Passos do rollback**:
  1. bloquear a escrita na AWS;
  2. gravar um registro marcador e conferir sua chegada no SQL Server;
  3. desligar a cópia;
  4. religar jobs e triggers do legado;
  5. devolver o tráfego ao on-prem (DNS/toggle).

## Custos

_A definir._

## Perguntas em aberto

- Quantos analistas Itaú estão disponíveis para o apoio, e cada um atende uma app/base ou várias? A resposta define quantos squads podem estar ao mesmo tempo nas etapas 3 e 4.
- Quantos squads o cliente vai investir (D17)?
- Quais siglas têm capacidade própria para migrar (D18)?
- O Itaú homologa o Babelfish?
- Como e com que custo será feita a passagem tradops → devops?
