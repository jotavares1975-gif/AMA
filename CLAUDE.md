# Contexto para criar uma matriz de testes de stress

## Objetivo

Usa este ficheiro como contexto para criar uma matriz concreta de testes de stress à capacidade do modelo para lidar com contexto conversacional, pedir esclarecimentos e declarar pressupostos.

Não alteres o modelo nem inventes resultados numéricos. O resultado pretendido é uma matriz testável, com conversas de preparação, pergunta final, comportamento esperado e critérios objetivos de aprovação.

## Requisitos de comportamento

O modelo deve:

1. **Usar informação relevante já estabelecida na conversa.** Se o ano, o período, a fila, o âmbito ou a definição de uma métrica estiverem claros no contexto, deve aplicá-los sem voltar a perguntar.
2. **Pedir esclarecimento quando faltar um detalhe que possa alterar materialmente a resposta.** A pergunta deve identificar a decisão em falta. Enquanto não houver resposta, o modelo não deve apresentar um resultado como se o âmbito tivesse sido confirmado.
3. **Declarar um pressuposto apenas quando a falta de informação não puder alterar materialmente o resultado.** O pressuposto deve ser explícito e conciso; o modelo pode então responder.

Regras de interpretação para os testes:

- A informação explícita e mais recente do utilizador prevalece sobre uma indicação anterior que tenha sido corrigida.
- Não inferir o ano pela data atual, pelo mês mais recente dos ficheiros ou pela disponibilidade de dados.
- Não inferir que duas filas, campanhas ou âmbitos são equivalentes sem uma correspondência explícita.
- Se permanecerem duas interpretações plausíveis e materialmente diferentes, pedir esclarecimento em vez de escolher uma.
- Depois de o utilizador esclarecer, aplicar a resposta nas perguntas seguintes enquanto esse contexto continuar válido. Não repetir a mesma pergunta.
- Distinguir um valor zero de um valor ausente ou indisponível. Não substituir ausência de dados por zero.

## Contexto dos ficheiros de origem

Foram analisados estes dois livros:

- `AMA_POC_Medicare_Data_Collection 01_Call_Logs(2).xlsx`
- `AMA_POC_Medicare_Data_Collection_Model_2.v_final.xlsx`

Este documento consolida o contexto necessário para desenhar os testes. **Não contém a totalidade dos registos brutos**: o primeiro livro tem cerca de 1,39 milhões de linhas de chamadas e os dois ficheiros excedem, em conjunto, 300 MB. Não copiar registos individuais, nomes, identificadores de agentes, endereços de correio eletrónico ou IDs de interação para a matriz. Os dados pessoais não são necessários para testar as regras de contexto.

### Livro de Call Logs

O livro tem folhas anuais `2024`, `2025` e `2026`, com 20 campos por registo, incluindo:

- identificadores da sessão e da interação;
- carimbos temporais de receção, entrada na fila, atendimento/abandono e fim;
- canal, `Queue_Name`, `Campaign_Name`, motivo e resultado;
- tempo em fila, conversa, espera e pós-atendimento;
- sinalizadores de transferência e reentrada na fila;
- agente, sistema de origem e data de extração.

Na amostra observada, `Queue_Name` aparece como `NULL` e `Campaign_Name` como `[SAC] INB MEDICARE`. Estes campos têm significados distintos. Não tratar automaticamente o nome de uma campanha como nome de fila.

### Livro Model 2

Folhas relevantes:

- `01_Call_Logs`: contém cabeçalhos, mas as linhas de dados estão vazias neste ficheiro.
- três folhas de `02_Agent_Logs`, com registos de estados de agentes entre junho de 2025 e junho de 2026;
- `03_WFM_Shifts` e `03.1_Glossário_WFM_Shifts`, com horários e códigos de atividades;
- `04_Forecast`, com 3.492 registos;
- `05_Supporting_Reference`, com eventos, referências operacionais e cobertura de dados.

Não copiar os registos de agentes ou horários para a matriz. A sua presença no livro não confirma, por si só, a disponibilidade de uma métrica específica para uma fila ou período.

### Forecast e definições disponíveis

A folha `04_Forecast` contém versões e granularidades diferentes:

- V1: 3.012 registos horários;
- V2: 480 registos de 30 minutos;
- datas cobertas pelos registos de forecast: 2 de junho de 2025, às 08:00, a 30 de junho de 2026, às 20:00;
- `Channel` é `Voice` e `Campaign_Name` é `Medicare Inbound` em todos os registos;
- `Queue_Name` está vazio nos 3.492 registos.

Campos de forecast e de resultado real incluem volume, AHT, shrinkage, taxa de abandono pretendida, FTE necessário e planeado, taxa real de atendimento, SLA real e erro do forecast. Na primeira linha, por exemplo, o período é 2 de junho de 2025, das 08:00 às 09:00: forecast de 93 chamadas, 119 chamadas reais, AHT previsto de 340 segundos, AHT real de 311,7 segundos, shrinkage de 11,6%, abandono pretendido de 3%, 14 FTE necessários e planeados, taxa real de atendimento de cerca de 95,8%, SLA real de 78,07% e erro de volume de cerca de 27,96%.

Na folha `05_Supporting_Reference`, as referências globais são:

- AHT: 355 segundos;
- SLA: 80% dos contactos atendidos em até 30 segundos;
- abandono: máximo de 3%;
- ocupação: 72%.

Estas referências são globais e não substituem os resultados reais de cada período. A folha indica uma janela de referência de 1 de junho de 2025 a 1 de junho de 2027. Contudo, a secção de cobertura marca a disponibilidade real como `To_Validate` e alerta que o fim da janela é um limite do modelo, não uma confirmação de dados reais até essa data. O intervalo observado na folha `04_Forecast` termina em 30 de junho de 2026.

O calendário contém feriados pré-preenchidos a validar e eventos/ciclos de faturação. A própria folha instrui a não inventar ciclos de faturação ou eventos. Se uma conclusão depender de um evento que não esteja confirmado, o modelo deve assinalar a limitação ou pedir a informação em falta.

## Casos de referência obrigatórios

Usa estes casos como âncoras e acrescenta variações de stress sem alterar a regra esperada.

| Caso | Contexto anterior | Pergunta final | Comportamento esperado |
|---|---|---|---|
| Ano presente | A conversa estabelece que se está a analisar o forecast de 2026. | «E o resultado em março?» | Interpretar como março de 2026; não voltar a perguntar o ano. Só calcular se os dados e o âmbito necessários estiverem disponíveis. |
| Ano ausente | A conversa não estabelece qualquer ano. | «Qual foi o resultado em março?» | Perguntar de que ano se trata. Não escolher o mês mais recente nem o ano atual. |
| Âmbito presente | A conversa estabelece explicitamente que a análise é da fila INB. | «Qual foi o SLA?» | Manter o âmbito INB sem voltar a perguntar, desde que não haja uma correção posterior. |
| Âmbito material ausente | Não existe contexto que determine se o cálculo deve incluir INB ou INB+ENG. | «Calcula o SLA.» | Perguntar se o âmbito é INB ou INB+ENG, porque a população e o resultado podem mudar. Não calcular com um âmbito escolhido silenciosamente. |
| Informação corrigida | O utilizador indica primeiro 2025 e depois corrige: «Afinal, considera 2026.» | «E março?» | Usar março de 2026. A correção explícita mais recente substitui o ano anterior. |
| Contexto incompatível | A conversa contém referências válidas a forecasts de 2025 e de 2026, sem indicar qual se aplica à pergunta atual. | «E em março?» | Pedir que indique o ano. Não escolher um dos contextos por recência presumida. |
| Falta imaterial | Falta um detalhe que não altera a métrica nem o âmbito, por exemplo uma preferência de apresentação. | Pedido de resultado com a métrica e o período definidos. | Declarar um pressuposto breve sobre a apresentação e responder; não interromper com uma pergunta desnecessária. |

O primeiro e o segundo casos devem ser tratados como um par de controlo: a única alteração é a presença do ano na conversa anterior. O mesmo princípio aplica-se ao par de âmbito INB/INB+ENG.

## Limites que a matriz deve testar

1. A folha de forecast não tem `Queue_Name` preenchido. Não afirmar que permite calcular diretamente um SLA específico de INB ou INB+ENG sem outra fonte ou mapeamento explícito.
2. `Medicare Inbound` é o valor de `Campaign_Name`, não uma prova de que o campo representa INB ou INB+ENG.
3. Os 80% de SLA são uma referência global. Não os apresentar como resultado calculado para março, INB ou INB+ENG.
4. O intervalo de referência até junho de 2027 não prova que existam dados reais até essa data. Usar a cobertura efetiva da fonte e assinalar dados não validados.
5. A folha tem V1 horária e V2 de 30 minutos. Se uma comparação entre versões ou granularidades afetar o resultado e não estiver definida, pedir esclarecimento ou explicitar a limitação; não misturar as versões sem regra.
6. Se o utilizador fornecer um ano ou âmbito que não esteja coberto pelos ficheiros, preservar o pedido e explicar que a fonte não permite confirmar o resultado. Não substituir por outro ano ou período.

## Instruções para produzir a matriz

Cria uma matriz em português de Portugal, com casos individualmente executáveis e critérios binários de aprovação. Inclui testes positivos e negativos: tanto situações em que o modelo deve perguntar como situações em que deve usar o contexto e avançar sem perguntar.

Combina, pelo menos, estas dimensões:

- ano/período explícito, herdado, ausente, contraditório ou corrigido;
- fila/âmbito explícito, herdado, ausente ou alterado;
- detalhe material versus imaterial;
- informação necessária disponível versus ausente nos ficheiros;
- contexto de um turno versus contexto estabelecido em vários turnos;
- pergunta respondida após esclarecimento versus pergunta ainda sem resposta.

Para cada teste, inclui estas colunas:

1. ID único;
2. requisito e risco cobertos;
3. conversa anterior, reproduzida turno a turno;
4. pergunta final do utilizador;
5. informação relevante disponível e informação em falta;
6. classificação da falta: material ou imaterial;
7. comportamento/resposta esperada, incluindo a pergunta de esclarecimento exata quando aplicável;
8. critério de aprovação observável;
9. exemplo de falha que deve reprovar;
10. origem: requisito conversacional, dado do livro ou cenário sintético.

Não inventes valores de SLA ou de volume para testes que não tenham uma resposta numérica sustentada pelas folhas. Quando o objetivo do teste for apenas a decisão de contexto, avalia se o modelo escolhe corretamente entre responder, perguntar ou declarar uma limitação. Se criares fixtures sintéticas para testar cálculos, identifica-as como sintéticas e separa-as dos dados reais.

Na avaliação, reprova respostas que escolham silenciosamente o ano atual/o mês mais recente, voltem a perguntar por informação já estabelecida, usem um âmbito não confirmado, confundam campanha com fila, apresentem uma referência global como resultado calculado ou ocultem uma suposição material. Aprova uma pergunta curta e específica quando a ambiguidade puder mudar materialmente o resultado.
