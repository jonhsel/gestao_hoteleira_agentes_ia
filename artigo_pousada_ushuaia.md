# Da Recepção ao Banco de Dados: Como Automatizamos a Gestão Completa de uma Pousada com IA, N8N e Supabase

**por Jonh Selmo — Engenheiro de Automação e Sistemas de IA**

---

A hospitalidade é, por natureza, um negócio de pessoas. Mas por trás de cada hóspede bem acolhido, há uma cadeia de processos operacionais que, quando feita manualmente, consome tempo, gera erros e sobrecarrega equipes pequenas. Este artigo descreve o projeto que desenvolvi para a **Pousada Ushuaia, localizada em Morros, Maranhão** — uma solução completa de automação de gestão hoteleira, da atendimento ao hóspede até a manutenção do banco de dados, construída sobre N8N, Dify, Supabase e a API do WhatsApp.

---

## O Negócio: Uma Pousada Charmosa com Desafios Operacionais Reais

A Pousada Ushuaia é um destino de turismo regional que oferece chalés com e sem cozinha, casas para grupos maiores e o serviço de Day Use — acesso às áreas de lazer sem pernoite. O público é predominantemente familiar e de casais, com picos de ocupação em feriados prolongados como Carnaval e Semana Santa, quando a pousada opera com **pacotes fechados** de diárias mínimas.

O cenário operacional antes do projeto era típico de pequenos meios de hospedagem: atendimentos via WhatsApp feitos manualmente pela equipe, planilhas para controle de reservas, comunicação de check-in e check-out por mensagens de voz ou texto, e nenhuma automação de relatórios. Tudo funcionava — mas escalava mal. Em períodos de alta temporada, a equipe ficava sobrecarregada com tarefas repetitivas enquanto precisava, ao mesmo tempo, garantir a qualidade do atendimento presencial.

O desafio central era claro: **como manter o calor humano no atendimento sem depender exclusivamente de pessoas para tarefas que podem ser automatizadas?**

---

## A Arquitetura: Um Ecossistema de Ferramentas Integradas

A solução foi construída como um sistema modular, onde cada camada é responsável por uma classe de problemas específica. As tecnologias escolhidas foram selecionadas deliberadamente para maximizar controle, flexibilidade e custo-benefício.

### Camada de Inteligência Conversacional: Dify + LLM

O coração do atendimento ao hóspede é **Maria**, uma assistente virtual com persona definida — calorosa, proativa e profissional — construída no **Dify**, uma plataforma open-source para orquestração de aplicações baseadas em LLMs. Maria opera como agente conversacional no WhatsApp e é capaz de:

- Identificar automaticamente o tipo de serviço desejado pelo hóspede (hospedagem, Day Use ou dúvida geral);
- Conduzir o fluxo de coleta de dados de forma natural — nome, datas, número de pessoas, tipo de acomodação;
- Aplicar silenciosamente políticas internas, como diárias mínimas em finais de semana, sem revelar as regras ao cliente;
- Detectar e apresentar pacotes especiais em datas comemorativas (Carnaval, Semana Santa) com tabelas de preços dinâmicas;
- Distinguir entre interesse ("quero uma reserva") e confirmação explícita ("sim, confirmo"), evitando pré-reservas prematuras;
- Manter **estado persistente entre mensagens** — um dos avanços mais importantes do projeto, que eliminou o problema clássico de assistentes que repetem perguntas já respondidas.

O prompt da Maria evoluiu por seis versões (V1 a V6), cada uma adicionando uma camada de sofisticação: memória de estado, validação de tipo de acomodação, tratamento de pacotes fechados e diferenciação semântica entre interesse e confirmação.

### Camada de Orquestração de Fluxos: N8N

O **N8N** funciona como o sistema nervoso da automação — é ele que conecta Maria ao banco de dados, processa decisões de negócio complexas e aciona as notificações corretas nos momentos certos. Os principais fluxos desenvolvidos foram:

**Fluxo de Reservas**: Após Maria coletar e confirmar todos os dados com o hóspede, o N8N assume e executa a lógica transacional completa. Ele verifica disponibilidade em tempo real, aplica as regras de validação de período, cria a pré-reserva no banco de dados com status "pendente", gera um código único de reserva e dispara automaticamente as instruções de pagamento via PIX para o cliente.

**Check-in Digital**: A equipe da pousada pode registrar o check-in de um hóspede enviando um simples comando via WhatsApp (`/checkin RSV123`). O N8N valida o horário, busca a reserva no Supabase, verifica elegibilidade, atualiza o status no banco e notifica tanto o cliente quanto a equipe administrativa em paralelo.

**Check-out Digital**: Seguindo lógica análoga ao check-in, o fluxo de check-out inclui uma etapa adicional de avaliação do estado do quarto — a equipe registra notas de limpeza, conservação e eventuais danos, informações que alimentam automaticamente a fila de tarefas de manutenção.

**Relatórios Automáticos Diários**: Todo dia, o N8N consulta o Supabase e gera um relatório consolidado com check-ins e check-outs do dia, reservas pendentes de confirmação e ocupação geral — entregue diretamente no WhatsApp da gestão.

**Limpeza Automática de Quartos**: Um workflow separado gerencia a fila de limpeza com base nos check-outs registrados, garantindo que a equipe de arrumação receba a lista de tarefas sem precisar de comunicação manual.

### Camada de Dados: Supabase com PostgreSQL

O **Supabase** — plataforma open-source construída sobre PostgreSQL — foi escolhido como backend de dados pela combinação de robustez relacional, API REST nativa e suporte a lógica transacional complexa via stored procedures.

O modelo de dados foi projetado para suportar o ciclo completo de uma reserva:

- `reservas_confirmadas`: tabela central com status do ciclo de vida (pendente → confirmada → check_in → check_out → expirada);
- `disponibilidade_quartos`: controle de disponibilidade por data, tipo de quarto e referência de reserva, com suporte a **bloqueios temporários** de 30 minutos para reservas em processo de confirmação;
- `historico_status_reserva`: auditoria completa de todas as transições de estado, com registro de responsável e motivo.

A lógica de disponibilidade merece destaque especial. Para evitar overbooking em cenários de alta concorrência — dois clientes reservando o mesmo quarto simultaneamente — foi implementado um sistema de **bloqueio temporário otimista**: ao iniciar uma reserva, o sistema bloqueia o quarto por 30 minutos. Se o pagamento não for confirmado nesse período, jobs automáticos (`pg_cron`) liberam o bloqueio e expiram a reserva. Quando o pagamento é confirmado, os bloqueios temporários são substituídos por bloqueios definitivos.

Os jobs de limpeza automática rodam em três frequências: a cada 10 minutos para bloqueios temporários expirados, a cada hora para reservas pendentes vencidas, e uma varredura profunda diária às 3h da manhã.

### Camada de Comunicação: Evolution API + WhatsApp

Toda a comunicação com hóspedes e equipe trafega via **Evolution API**, uma implementação open-source do protocolo do WhatsApp Business. Isso permite que o sistema envie e receba mensagens estruturadas sem depender dos custos e limitações da API oficial do Meta, mantendo toda a comunicação dentro do ecossistema controlado da solução.

---

## Maria Como Agente de IA: Além do Chatbot Convencional

Um dos aspectos mais relevantes — e menos óbvios — deste projeto é que a Maria não é um simples chatbot com respostas pré-programadas. Ela é um **agente de IA**: um sistema autônomo que percebe o contexto, raciocina sobre ele, decide qual ação tomar e executa essa ação com consequências reais no mundo.

Essa distinção não é apenas semântica. Ela define a arquitetura, o nível de complexidade e o tipo de valor que o sistema entrega.

### O que diferencia um Agente de IA de um chatbot tradicional

Um chatbot convencional opera com lógica de decisão determinística: se o usuário digitar X, o sistema responde Y. O mapeamento é estático, frágil a variações de linguagem e incapaz de lidar com situações não previstas explicitamente. O comportamento é inteiramente definido por quem o programou.

Um agente de IA, por outro lado, opera sobre um modelo de linguagem capaz de compreender intenção, interpretar contexto ambíguo e gerar respostas originais. Mas o que o transforma em *agente* — e não apenas em um LLM respondendo perguntas — é a combinação de três capacidades:

- **Percepção de estado**: a Maria lê variáveis de contexto injetadas a cada turno (datas coletadas, tipo de acomodação, se há confirmação explícita, histórico recente) e usa esse estado para determinar *onde está* na conversa antes de decidir o que fazer;
- **Raciocínio sobre objetivos**: o prompt instrui a Maria não apenas sobre o *que* dizer, mas sobre *por que* cada etapa existe — ela entende que seu objetivo é criar uma pré-reserva válida, e toma decisões intermediárias para chegar lá;
- **Execução de ações com efeitos externos**: ao confirmar os dados com o hóspede, a Maria não apenas responde — ela aciona um webhook que desencadeia uma cadeia de operações reais: verificação de disponibilidade, escrita no banco de dados, geração de código de reserva e envio de instruções de pagamento.

É essa última capacidade — agir sobre o mundo externo, não apenas conversar — que caracteriza o paradigma agêntico.

### A Arquitetura Agêntica: Dify + N8N como Cérebro e Braços

No projeto da Pousada Ushuaia, a arquitetura agêntica foi implementada com uma separação clara de responsabilidades entre duas plataformas:

O **Dify** funciona como o *cérebro cognitivo* do agente: é onde reside o modelo de linguagem, o prompt de sistema, a lógica de persona e as instruções de comportamento. O Dify recebe a mensagem do hóspede junto com todas as variáveis de estado e produz duas saídas: a resposta para o usuário e metadados estruturados sobre o estado atual da conversa (tipo de serviço identificado, dados coletados, fluxo a seguir).

O **N8N** funciona como o *sistema motor* do agente: ele processa os metadados retornados pelo Dify, executa as verificações de negócio (disponibilidade, diárias mínimas, períodos de pacote), escreve e lê do banco de dados, e decide se a conversa deve continuar em modo de coleta ou acionar a criação da reserva. O N8N também mantém o estado da conversa entre mensagens, alimentando o Dify com o contexto atualizado a cada turno.

Esse modelo — LLM como módulo de raciocínio linguístico, orquestrador de workflows como módulo de execução e gestão de estado — é uma implementação prática do padrão **ReAct** (*Reasoning + Acting*), amplamente discutido na literatura de agentes de IA. A diferença é que aqui ele foi construído com ferramentas acessíveis e sem uma linha de código Python de orquestração.

### Agentes com Memória: o Problema do Estado

Um dos desafios centrais no desenvolvimento de agentes conversacionais é que LLMs são, por natureza, *stateless*: cada chamada à API do modelo é independente, sem memória das interações anteriores. Para um chatbot simples, isso não é problema. Para um agente que precisa conduzir uma negociação de múltiplos turnos — coletando dados, validando, aguardando confirmação, processando — é uma limitação crítica.

A solução implementada foi o que se pode chamar de **memória externalizada com injeção de contexto**: o N8N mantém um objeto de estado estruturado com todos os dados coletados ao longo da conversa e o injeta no contexto do prompt a cada nova mensagem. Para o modelo, cada turno parece uma conversa completa com todo o histórico relevante disponível. Para o sistema, o estado é sempre consistente e auditável, independente de falhas ou reinicializações.

Essa abordagem é mais robusta do que depender de janelas de contexto longas (que aumentam custo e latência) e mais flexível do que soluções de memória vetorial (adequadas para recuperação de conhecimento, mas não para gestão de estado transacional).

### Tomada de Decisão em Múltiplas Camadas

A Maria não toma decisões em uma única camada. O sistema implementa uma hierarquia de decisão com três níveis:

No **nível do LLM** (Dify), as decisões são linguísticas e intencionais: o modelo identifica se o cliente quer hospedagem ou Day Use, se está confirmando ou apenas demonstrando interesse, se a mensagem contém dados de datas ou pessoas, e qual é o próximo passo natural da conversa.

No **nível do orquestrador** (N8N), as decisões são de negócio e validação: o sistema verifica se as datas atendem às políticas de diárias mínimas, se o período é um pacote fechado, se há disponibilidade real no banco, e se todos os dados obrigatórios foram coletados antes de acionar a criação da reserva.

No **nível do banco de dados** (Supabase/PostgreSQL), as decisões são transacionais e de consistência: as stored procedures verificam condições de corrida, gerenciam bloqueios e garantem integridade dos dados independentemente do que as camadas superiores tenham decidido.

Essa arquitetura em camadas garante que uma falha ou comportamento inesperado em qualquer nível não comprometa a integridade do sistema como um todo. O banco de dados nunca confia cegamente nas camadas acima — ele valida por si mesmo.

### O Futuro: Agentes Mais Autônomos

A versão atual da Maria opera dentro de um escopo bem definido, com fluxos pré-determinados e um conjunto fechado de ações possíveis. Esse é o padrão mais seguro e previsível para um ambiente de produção.

A evolução natural é em direção a agentes com maior autonomia: capazes de consultar disponibilidade proativamente antes mesmo de o cliente perguntar, sugerir datas alternativas com base em padrões históricos de ocupação, negociar condições especiais dentro de parâmetros autorizados, e escalar para atendimento humano apenas quando a situação genuinamente exigir julgamento além das capacidades do agente.

O paradigma agêntico não substitui a equipe da pousada — ele libera essa equipe para o que realmente importa: o acolhimento presencial, a resolução de situações excepcionais e a construção de relacionamentos que nenhum sistema automatizado consegue replicar.

---

## Fluxo Completo de uma Reserva: Do "Olá" à Confirmação

Para tornar a arquitetura mais concreta, vale descrever o caminho completo de uma reserva de hospedagem:

1. O hóspede envia uma mensagem no WhatsApp da pousada demonstrando interesse em se hospedar.
2. Maria, no Dify, identifica a intenção e inicia a coleta de dados: datas, número de pessoas e tipo de acomodação.
3. Silenciosamente, Maria verifica as políticas de diárias mínimas. Se o período solicitado for inadequado, informa genericamente que "não há disponibilidade" e sugere alternativas.
4. Se o período incluir uma data comemorativa com pacote fechado, Maria apresenta as opções do pacote com tabela de preços e aguarda a escolha do cliente.
5. Com todos os dados coletados, Maria apresenta um resumo completo e solicita confirmação explícita.
6. Ao receber o "sim" do cliente, um webhook é disparado para o N8N.
7. O N8N executa a função PostgreSQL `verificar_e_criar_reserva`, que atomicamente verifica disponibilidade, cria a reserva com status "pendente" e registra os bloqueios temporários.
8. O sistema gera o código único de reserva e envia ao cliente as instruções de pagamento via PIX.
9. Ao receber o comprovante, a equipe confirma o pagamento via comando, o N8N aciona `confirmar_reserva_transacional`, que atualiza o status para "confirmada", remove bloqueios temporários e cria os bloqueios definitivos.
10. No dia do check-in, a equipe registra a entrada via comando no WhatsApp, e o sistema notifica o hóspede com as informações de acomodação.

---

## Tempo de Desenvolvimento

O projeto foi desenvolvido de forma iterativa ao longo de **aproximadamente 3 meses**, divididos em três fases principais:

**Fase 1 — Fundação (4 semanas)**: Modelagem do banco de dados no Supabase, desenvolvimento das stored procedures transacionais e configuração da infraestrutura base (N8N, Evolution API, conexões).

**Fase 2 — Automações Operacionais (5 semanas)**: Desenvolvimento dos fluxos de N8N para check-in, check-out, relatórios e limpeza. Criação da Maria em sua versão inicial (V1 a V3) com os fluxos básicos de hospedagem e Day Use.

**Fase 3 — Refinamentos e IA Avançada (3 semanas)**: As versões V4 a V6 da Maria, incorporando o sistema de estado persistente, a lógica de distinção entre interesse e confirmação, o tratamento de pacotes especiais e a integração completa dos fluxos de coleta de dados com validações em tempo real via N8N.

O volume de iterações na camada de IA (seis versões do prompt principal, além de dezenas de ajustes nos nós de processamento do N8N) reflete uma realidade importante em projetos de sistemas conversacionais: o comportamento emergente dos LLMs exige validação contínua com casos reais. Cada nova versão da Maria surgiu de um problema identificado em produção — uma pergunta repetida desnecessariamente, uma reserva processada antes da confirmação explícita, uma política interna revelada por engano.

---

## Desafios Técnicos e Soluções

**O problema do estado em conversas longas**: LLMs não têm memória nativa entre mensagens. A solução foi externalizar o estado da conversa em variáveis gerenciadas pelo N8N, injetadas no contexto do prompt da Maria a cada interação. Isso criou um sistema híbrido onde a IA é stateless, mas a conversa tem memória.

**Prevenção de overbooking em alta concorrência**: A solução de bloqueio temporário otimista com expiração automática, implementada diretamente no PostgreSQL via `pg_cron`, garantiu consistência transacional mesmo com múltiplas requisições simultâneas — algo que uma abordagem naive baseada apenas em leitura de disponibilidade seguida de escrita não conseguiria garantir.

**Diárias mínimas sem revelar a política**: O requisito do negócio era que clientes não soubessem da existência da política de estadia mínima — deveriam receber apenas uma mensagem de "indisponibilidade" com sugestão de datas alternativas. Isso foi implementado como uma camada de validação silenciosa no processamento do N8N, antes de qualquer resposta da Maria.

**Pacotes fechados em datas comemorativas**: O sistema detecta automaticamente quando uma solicitação de reserva coincide com um período de pacote e bloqueia a criação de reservas avulsas, redirecionando o cliente para a lógica de pacotes com tabela de preços dinâmica.

---

## Resultados e Impacto Operacional

A solução reduziu drasticamente o volume de tarefas manuais da equipe em horários de pico. O atendimento via WhatsApp agora acontece 24 horas por dia, 7 dias por semana, com Maria respondendo instantaneamente mesmo fora do horário comercial. As reservas chegam ao banco de dados já estruturadas, com todos os dados validados — eliminando o retrabalho de interpretar mensagens informais e transcrever informações para planilhas.

Os relatórios automáticos diários garantem que a gestão tenha visibilidade operacional completa sem precisar consultar ativamente o sistema. E o histórico de auditoria de todas as transições de status de reservas cria um registro rastreável de toda a operação — algo que planilhas nunca conseguiram oferecer.

---

## Reflexões sobre a Stack Tecnológica

A escolha de N8N, Dify e Supabase em vez de soluções SaaS proprietárias foi deliberada e reflete uma filosofia de **soberania tecnológica**: todo o sistema roda em infraestrutura própria (via Easypanel), o código é auditável e modificável, e não há dependência de contratos ou precificações de terceiros que podem mudar a qualquer momento.

O N8N, em particular, mostrou-se extraordinariamente flexível para implementar lógica de negócio complexa. A capacidade de misturar nós visuais com blocos de código JavaScript personalizado permite que automações simples e sofisticadas coexistam no mesmo fluxo — algo que plataformas puramente visuais não conseguem e que plataformas puramente de código tornam desnecessariamente verboso.

O Dify se mostrou a escolha certa para gestão de prompts complexos com múltiplas variáveis de contexto. A capacidade de versionar prompts, testar variações e injetar variáveis dinâmicas de forma estruturada é essencial para projetos onde o comportamento da IA precisa ser finamente ajustado.

---

## Conclusões e Próximos Passos

Este projeto demonstra que automação de gestão hoteleira de alta qualidade não é exclusividade de grandes redes com equipes de TI robustas. Com as ferramentas certas e uma arquitetura bem pensada, é possível dotar pequenos meios de hospedagem de capacidades operacionais que antes eram inacessíveis.

A Maria continuará evoluindo. Os próximos passos incluem a integração com sistemas de pagamento para confirmação automática de PIX, a implementação de um painel web para a gestão visualizar ocupação em tempo real, e a expansão do sistema de avaliação de quartos para alimentar um módulo de manutenção preventiva.

O código base construído aqui é também um ponto de partida replicável para outros estabelecimentos. A modularidade da arquitetura permite que os fluxos de N8N, o schema do Supabase e a persona do Dify sejam adaptados para diferentes tipos de meios de hospedagem com esforço relativamente baixo.

---

*Jonh Selmo — Engenheiro de Automação e Sistemas de IA | N8N · Dify · Supabase · LLM Applications*
