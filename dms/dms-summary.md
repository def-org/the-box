# DMS

## Visão geral
O projeto migra a base SQL Server on-premises **SQNPRD** (bancos `db1` e `db2`, schema `dbo`), hospedada no itaú-datacenter, para o **Amazon RDS for PostgreSQL Multi-AZ** (schema `schemaapp`). A replicação é feita com **AWS DMS** (full load + CDC). A nova base atende a versão modernizada da aplicação, que já roda na AWS. O plano inclui um período de convivência, uma virada coordenada da app e da base, uma janela de fallback com CDC reverso e, por fim, o desligamento do on-prem e a adoção do modelo de dados corporativo.

## Caso de negócio
- A licença de SQL Server deixa de existir.
- Backup, alta disponibilidade e patch passam a ser responsabilidade da AWS, como serviço gerenciado.
- A base fica alinhada à aplicação modernizada que já roda na AWS e ao modelo de dados corporativo.
- Os custos ainda não foram quantificados ("a definir").

## Objetivos de negócio
- Migrar o schema e os dados de SQNPRD para o RDS for PostgreSQL.
- Manter a replicação contínua com atraso de segundos, por dias ou semanas, enquanto a app nova é testada.
- Fazer a virada com atraso zerado e a possibilidade de rollback por CDC reverso.
- Desligar o on-prem ao fim da janela de fallback (por exemplo, 2–4 semanas).
- Normalizar para o modelo corporativo (exemplo: `cliente_contato` → `pessoa` + `contato`; `id_cliente_contato` → `id_pessoa`).
- Não há metas numéricas além do atraso de segundos (ver Perguntas em aberto).

## Características
- **Origem:** SQNPRD em SQL Server (rotulado "sql server 2027" no desenho), bancos `db1` e `db2`, schema `dbo`.
- **Destino:** RDS for PostgreSQL Multi-AZ, schema `schemaapp` (`db1`, `db2`).
- **Conectividade:** Direct Connect (DX) entre o itaú-datacenter e a AWS, passando por uma conta transitiva com regras de firewall e de data-security.
- **Ferramental:**
  - **schema-forge** (passo 1): gera o STM com apoio de agentes de IA e automação;
  - **data-devops-pipelines** (passo 2): fluxo DMS em 8 etapas;
  - **breaking glass:** acesso direto para ajustes críticos.
- **Controle de tráfego:** Portal Manager com os toggles `read-cloud` / `write-cloud`, além de DNS e da entrada de tráfego.
- **Integrações:** kaas-convivencia, API Gateway + Lambda `middleware-dt-handler`, `middleware-consumer`, e `batch-democratizacao` → datamesh (S3, camadas sor/sot/spec).
- **Restrições:**
  - Guardrail: as apps on-prem não acessam a base na AWS, por isso a app e a base viram juntas.
  - O DMS só faz cópia tabela → tabela, coluna → coluna e linha a linha. Divisões, pivôs, deduplicação, lookups e joins exigem scripts, o STM e a tabela `map_pessoa`.
- **Volumes:** o documento não traz volumes de dados nem de transações.

## Situação atual e desafios
- APIs, batch, integrações e o `middleware-consumer` on-prem leem e gravam diretamente no SQNPRD.
- Existem possíveis consumidores ocultos ainda sem inventário: jobs, relatórios, integrações, linked servers e SSIS.
- O SQL Server e o PostgreSQL se comportam de forma diferente em collation, maiúsculas/minúsculas, fuso horário e SQL embutido nas apps.
- As transformações estruturais exigidas pelo modelo corporativo ficam fora do alcance do DMS.

## Solução proposta
- Uma task AWS DMS `full-load-and-cdc` lê as tabelas e o log do SQL Server via DX e a conta transitiva, e grava o full load e o CDC no RDS.
- **Fluxo:**
  1. marcar a posição do log;
  2. fazer o full load com as apps gravando no legado;
  3. recuperar o atraso;
  4. manter a replicação contínua durante os testes;
  5. fazer a virada: congelar a escrita no legado, conferir o atraso zerado, ajustar as sequences, ligar o CDC reverso e ativar a stack AWS (Portal Manager e/ou DNS).
- **CDC reverso (RDS → SQNPRD):** ligação direta, sem a conta transitiva. As apps legadas ficam implantadas e sem tráfego durante a janela de fallback.
- **Exceções:**
  - 6.1: breaking glass, com correções na AWS;
  - 6.2: rollback. O rollback segue estes passos: bloquear a escrita na AWS, gravar um registro marcador e conferir sua chegada, desligar a cópia, religar jobs e triggers e devolver o tráfego.
- **Depois da janela de fallback:** o on-prem é desligado e a normalização para o modelo corporativo acontece na cloud.

## Decisões-chave
| # | Decisão | Status | Motivo |
|---|---|---|---|
| D1 | Amazon RDS for PostgreSQL Multi-AZ | Decidida | Backup, HA e patch na AWS; fim da licença de SQL Server |
| D2 | AWS DMS full load + CDC na AWS, via DX/VPN | Decidida | Cópia online com atraso de segundos |
| D3 | Schema 1:1 na migração e normalização na cloud após desligar o on-prem (expand/contract), com anticorruption layer | Proposta | Caminho mais barato, com rollback simples |
| D4 | Alternativa: normalizar durante a migração (`stg` → `corp` por SQL/dbt; volta `corp` → `rev` por trigger → DMS) | Proposta | Só se a app nova falar apenas o modelo novo; mais cara e com rollback mais complexo |
| D5 | Janela de fallback com CDC reverso | Decidida | Dado errado ou causa desconhecida → rollback; erro com correção conhecida → correção na AWS |
| D6 | CDC reverso direto DMS → SQNPRD, sem a conta transitiva | Decidida | Definição do arquiteto |

## Riscos e mitigações
| # | Risco | Mitigação |
|---|---|---|
| R1 | Diferenças de comportamento SQL Server × PostgreSQL | Testes de equivalência durante a replicação contínua |
| R2 | O DMS não migra o valor das sequences | `setval` na virada |
| R3 | O slot do CDC reverso retém WAL | Alarmes em `OldestReplicationSlotLag` e `TransactionLogsDiskUsage`; remover o slot no fim da janela |
| R4 | Os triggers do legado disparam com as gravações do DMS | Desligar os triggers durante o fallback |
| R5 | Consumidores ocultos | Inventário antes da virada |
| R6 | Transformações estruturais fora do alcance do DMS | Scripts + STM + `map_pessoa` |

## Perguntas em aberto
1. A app nova consegue gravar no modelo antigo por meio de uma anticorruption layer? A resposta define D3 ou D4.
2. A virada e o rollback serão feitos por DNS/entrada de tráfego, pelos toggles do Portal Manager ou pelos dois?
3. Como os caminhos de convivência (API Gateway → `middleware-dt-handler` → RDS; kaas-convivencia → `middleware-consumer`) se conciliam com o guardrail?
4. A conta transitiva e o DX liberam a rota AWS → on-prem na porta 1433? A ligação direta do CDC reverso (D6) está liberada?
5. Qual é o prazo da janela de fallback e quais são os critérios objetivos de rollback?
6. Cada linha do STM será tratada pelo DMS ou por script?
7. Qual é o inventário de consumidores ocultos?
8. Quais serviços de monitoramento e de segurança/segredos ficam ligados ao DMS? Qual é a versão real do SQL Server?
9. Quais são os custos, o cronograma, os volumes de dados e as metas mensuráveis do projeto? O documento não os informa.

## Próximos passos
- Decidir entre D3 e D4 a partir da capacidade da app nova de gravar no modelo antigo.
- Fazer o inventário dos consumidores ocultos (R5).
- Validar a conectividade do DX e da conta transitiva (porta 1433) e a rota direta do CDC reverso.
- Classificar o STM entre DMS e script e preparar a `map_pessoa`.
- Definir o mecanismo de virada, o prazo da janela de fallback e os critérios de rollback.
- Levantar custos, cronograma e volumes.
