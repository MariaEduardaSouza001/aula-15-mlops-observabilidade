# Plano de observabilidade — agente de trocas

Antes de tudo, duas decisões que usamos no plano inteiro.

A primeira é a unidade de trabalho. Na conversa típica o cliente manda o número do pedido e o motivo na mesma mensagem, e o agente responde com a decisão. Nesse caso, a conversa é uma execução e vira um *trace*, com o `invoke_agent` na raiz, como pede o enunciado. Quando o agente precisa perguntar alguma coisa de volta, cada turno vira um *trace* separado, e todos são ligados pelo `gen_ai.conversation.id`. Assim o tempo que o cliente leva digitando não entra no tempo de resposta do agente. O tempo de resposta é medido por turno; a decisão e a promessa são medidas por conversa.

A segunda é o volume: umas 500 conversas por dia, o que dá algo perto de 14 mil conversas numa janela de 28 dias.

---

## 1. Perguntas

| Interessado | Pergunta | Decisão que a resposta muda |
|---|---|---|
| o negócio | Quantas trocas o agente resolve sozinho, aprovando ou recusando, sem passar para o atendente? | Se ficar abaixo dos 60% prometidos, a loja revê o case: o agente está encaminhando demais, ou a política tem casos que ele não consegue decidir? |
| a operação | Quantas conversas pararam no meio na última hora, por estourar os 6 passos ou por erro? | Voltar para a versão anterior ou corrigir a integração que está falhando. |
| quem paga | Quanto custa uma conversa? | Aplicar uma alavanca de redução: encurtar o que as ferramentas devolvem, testar um modelo mais barato ou mexer no limite de passos. |
| quem responde pelo sistema | Alguma troca foi registrada fora da política, ou sem o agente ter consultado o pedido e a política antes? | Parar a aprovação automática e exigir que um atendente confirme antes de registrar, até achar a causa. |

Duas perguntas a mais, que entram por causa do tempo de resposta e do limite de tokens (anexos A e B):

| Interessado | Pergunta | Decisão que a resposta muda |
|---|---|---|
| a operação | O cliente está esperando mais de 10 segundos pela resposta? | Investigar se a demora está no provedor do modelo ou numa ferramenta, e trocar para o modelo de reserva se for o provedor. |
| quem paga | Algum usuário está gastando muito mais tokens do que uma pessoa normal gastaria? | Bloquear o usuário no agente (com encaminhamento para o atendente) ou rever o limite, se ele estiver pegando gente normal. |

Pensamos também em perguntar quantas vezes por dia o `consultar_politica` é chamado, mas tiramos, porque a resposta não muda nada que alguém faria.

---

## 2. O *trace* de uma conversa

**Árvore de *spans* de uma conversa típica:**

Um caso normal de troca aprovada usa 4 dos 6 passos:

```text
invoke_agent  agente-de-trocas
├── chat mistral-small-latest        (modelo pede consultar_pedido)
├── execute_tool consultar_pedido
├── chat mistral-small-latest        (modelo pede consultar_politica)
├── execute_tool consultar_politica
├── chat mistral-small-latest        (modelo decide aprovar, pede registrar_troca)
├── execute_tool registrar_troca
└── chat mistral-small-latest        (modelo responde o cliente)
```

Na raiz ficam `gen_ai.agent.name = agente-de-trocas`, `gen_ai.provider.name` e `gen_ai.conversation.id`. Em cada `chat` ficam o modelo, os tokens de entrada e saída e o `finish_reasons`; em cada `execute_tool`, o nome da ferramenta e o `gen_ai.tool.call.id`. Se a troca é recusada ou encaminhada, o `registrar_troca` não aparece. Se o orçamento acaba, aparecem 6 `chat` e a conversa vai para o atendente.

A duração do `invoke_agent` é o tempo de resposta do turno. A duração de cada `chat` e de cada `execute_tool` mostra onde o tempo foi gasto.

**Atributos próprios na raiz:**

| Atributo | Valores possíveis | Para que serve |
|---|---|---|
| `troca.decisao` | aprovada, recusada, encaminhada, ausente | É com ele que dá para medir a promessa: resolvido sem atendente = aprovada + recusada. O "ausente" pega os casos em que o agente respondeu, mas sem decisão clara. |
| `agente.termino` | respondeu, orcamento_esgotado, limite_tokens, erro_fatal | Separa as conversas que terminaram bem das que pararam no meio. Junto com a decisão, mostra se um encaminhamento foi porque o agente ficou em dúvida (ok) ou porque acabaram os passos ou os tokens do usuário (problema). |
| `troca.dentro_do_prazo` | sim, nao, nao_se_aplica | A própria aplicação calcula, comparando a data da compra com o prazo da categoria. Não deixamos isso com o modelo. Assim dá para checar o prazo em todas as trocas aprovadas, sem precisar de juiz. |

Esses três atributos têm poucos valores, então podem ser usados para cortar as métricas. Já o número do pedido e o identificador do cliente ficam só no *span*, porque como dimensão de métrica iam explodir a cardinalidade.

**O carimbo de versão.** Sempre que o plano fala em "olhar por versão", são três coisas juntas, porque cada uma pode mudar sem as outras:

| O que muda | Onde fica no *trace* | Como pode mudar sozinho |
|---|---|---|
| o código | `service.version` (a mesma *tag* da imagem Docker) | *deploy* novo |
| o prompt | `agente.versao_prompt` | alguém edita o prompt no *registry*, sem *deploy* |
| o modelo | `gen_ai.request.model` em cada `chat` | troca para o modelo de reserva, ou o provedor muda o modelo por trás do mesmo nome |

Se gravássemos só a versão do código, uma mudança de prompt não apareceria em nenhum corte, e o modo de falha 3 (excesso de cautela) ficaria invisível.

---

## 3. Modos de falha

| Componente | Falha | Ruidosa ou silenciosa | Sinal que a torna visível |
|---|---|---|---|
| modelo + `registrar_troca` | Aprova e registra uma troca que estava fora da política (prazo vencido, categoria sem troca). Ou chama a ferramenta quando não devia: sem consultar a política antes, para outro pedido, ou duas vezes na mesma conversa. | Silenciosa. A troca entra no sistema normalmente e ninguém reclama, pelo menos o cliente não. | `troca.dentro_do_prazo = nao` em troca aprovada; `registrar_troca` sem `consultar_politica` antes no mesmo *trace*; mais de um `registrar_troca` por *trace*. As outras condições da política aparecem no juiz e quando um atendente desfaz uma troca aprovada pelo agente. |
| laço do agente | Fica repetindo consulta, não chega a uma decisão e estoura os 6 passos. | Silenciosa. O cliente é encaminhado e não aparece erro nenhum, mas a taxa de resolução cai. | `agente.termino = orcamento_esgotado` |
| modelo | Começa a encaminhar casos simples que conseguiria resolver, por exemplo depois de alguém mudar o prompt ou de o provedor trocar o modelo. | Silenciosa, e vai piorando aos poucos. Latência, custo e erro continuam iguais. | Aumento de `troca.decisao = encaminhada` com `agente.termino = respondeu`, cortado pelo carimbo de versão. E um rótulo implícito: o atendente aprova sem mudar nada um caso que o agente encaminhou. |
| `consultar_pedido` | O sistema de pedidos da loja fica fora do ar ou o número do pedido não existe. | Ruidosa, a ferramenta devolve erro. | `execute_tool` com status ERROR, separado por `gen_ai.tool.name`; `agente.termino = erro_fatal` quando o agente não consegue seguir. |
| provedor do modelo | A API fica fora do ar ou muito lenta (15, 20 segundos por chamada). | Ruidosa. Aparece em sinal técnico comum: erro da API no `chat` ou duração alta. | `chat` com status ERROR; duração do `invoke_agent` acima de 10 s, com a duração dos `chat` mostrando que o tempo foi gasto no modelo e não nas ferramentas. |

Seguindo a dica do enunciado, achamos mais um caso: se a categoria não estiver cadastrada, o `consultar_politica` pode devolver o prazo padrão de 30 dias, e o agente decide em cima de uma regra errada. Hoje não temos sinal para isso. Para ter, a ferramenta precisaria registrar no *span* se usou a política da categoria ou a padrão.

---

## 4. Objetivos

| Tipo | Indicador | Objetivo | Janela |
|---|---|---|---|
| operacional | % de conversas com `agente.termino = respondeu` | pelo menos 95% (sobram uns 5%, umas 700 conversas no período, de margem) | 28 dias |
| a promessa da loja | % de conversas resolvidas sem atendente (decisão aprovada ou recusada, término respondeu) | pelo menos 60% | 28 dias |

**O denominador da promessa.** Contamos todas as conversas de troca, **menos** as encerradas por `limite_tokens`. Essas conversas são de robô ou de abuso, e não são pedidos de troca de verdade; se entrassem, um ataque derrubaria a taxa de resolução sem o agente ter piorado em nada. Mesmo fora da conta, elas são acompanhadas à parte (anexo B), para saber se o limite está pegando cliente normal.

A promessa tem duas partes, e não adianta cumprir os 60% aprovando troca que não devia. Por isso acompanhamos também a conformidade: das trocas aprovadas, menos de 1% fora da política, em 28 dias. Aqui contam as aprovadas com `troca.dentro_do_prazo = nao` mais as que o juiz reprovar na amostra.

Dois objetivos de apoio, usados pelos alertas da seção 5:

| Tipo | Indicador | Objetivo | Janela |
|---|---|---|---|
| tempo de resposta | % de turnos em que o agente responde em até 10 s (duração do `invoke_agent`) | pelo menos 95% | 28 dias |
| custo | custo médio por conversa | até R$ 0,01 | 28 dias |

Por que 10 segundos: numa conversa normal o modelo é chamado 4 vezes, cada chamada leva por volta de 1 a 2 s, e as ferramentas somam menos de 1 s. Isso dá uns 5 a 7 s. O limite de 10 s deixa uma folga, mas pega quando o provedor ou alguma ferramenta começa a travar. A conta do custo está no anexo D.

---

## 5. Alertas

| Alerta | Condição e limiar | SLO de origem | Quem é avisado | O que faz |
|---|---|---|---|---|
| custo | Custo médio por conversa na última hora acima de R$ 0,02 (o dobro da meta), ou custo total da hora acima de R$ 1,20 | custo médio de até R$ 0,01 por conversa | responsável pelo agente, com cópia para quem paga | Olha o painel pelo carimbo de versão. Se cada conversa ficou mais cara, compara com a versão anterior e volta se for o caso. Se o que subiu foi o número de conversas, vê se não é robô ou abuso e confere os bloqueios por limite de tokens. |
| degradação silenciosa | Mais de 15% das conversas terminando em `orcamento_esgotado` nas últimas 6 horas. Nesse ritmo, a margem de 5% do mês acaba em menos de 10 dias. | pelo menos 95% de conversas com término normal | responsável pelo agente (quem estiver de plantão) | Vê se a piora começou com alguma mudança de código, prompt ou modelo e, se sim, volta. Abre alguns *traces* que estouraram o orçamento para ver qual ferramenta está se repetindo. Se nada mudou do nosso lado, desconfia que o provedor mudou o modelo e roda a suíte de *evals* de novo. |
| tempo de resposta | Mais de 10% dos turnos acima de 10 s nas últimas 3 horas, contando só se houver pelo menos 100 turnos na janela. Nesse ritmo, a margem de 5% do mês acaba em 14 dias. À parte, 3 ou mais respostas acima de 30 s em 15 minutos indicam provedor fora do ar. | pelo menos 95% dos turnos em até 10 s | responsável pelo agente (plantão) | Abre os *traces* lentos e vê qual *span* demorou. Se foi o `chat`, olha a página de status do provedor e, se continuar, troca para o modelo de reserva. Se foi uma ferramenta, aciona o time do sistema de pedidos. Se a lentidão começou depois de uma versão nova, volta a versão. |

**Por que essas janelas.** Com umas 40 conversas por hora, uma janela de 1 hora teria poucos casos, e 4 ou 5 respostas ruins já disparariam o alerta à toa. Por isso usamos 6 horas no alerta de degradação e 3 horas, com mínimo de 100 turnos, no de tempo. Cada resposta lenta ou cada orçamento esgotado continua registrado no log (anexo C) e no painel; o alerta só chama uma pessoa quando o padrão se repete o bastante para ameaçar o SLO.

**O modelo de reserva** já passou pela suíte de *evals* antes de ser cadastrado, e o custo dele é conhecido. A troca aparece no `gen_ai.request.model`, então o painel separa as conversas atendidas por ele.

Fica de fora do mínimo pedido, mas achamos importante: qualquer troca aprovada fora do prazo avisa na hora o responsável pelo atendimento, para desfazer a troca e, se repetir, ligar a confirmação humana.

---

## 6. Matriz de rastreabilidade

Liga cada pergunta da seção 1 ao sinal que a responde. Lida ao contrário, ela também mostra que todo sinal do plano tem uma pergunta por trás.

| Pergunta | Sinal | Atributos | Quem consome |
|---|---|---|---|
| quantas trocas o agente resolve sozinho? | contagem de conversas por decisão | `troca.decisao`, `agente.termino`, carimbo de versão | painel semanal do SLO da promessa |
| quantas conversas pararam no meio? | contagem de conversas por término | `agente.termino`, carimbo de versão | alerta de degradação silenciosa |
| quanto custa uma conversa? | soma dos tokens do *trace* × preço do modelo | `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.request.model` | painel e alerta de custo |
| alguma troca foi registrada fora da política? | contagem de aprovadas fora do prazo; ordem dos `execute_tool` no *trace* | `troca.dentro_do_prazo`, `gen_ai.tool.name`, número do pedido no *span* | alerta de conformidade e consulta quando uma troca é contestada |
| o cliente está esperando mais de 10 s? | duração do `invoke_agent` e de cada filho | duração dos *spans*, carimbo de versão | alerta de tempo de resposta e painel |
| algum usuário está gastando tokens demais? | soma de tokens por cliente no dia, contada pela aplicação | identificador do cliente (no *span* e no log, nunca como dimensão de métrica) | bloqueio automático e painel de bloqueios por dia |
| as decisões estão certas? | juiz sobre uma amostra e rótulos implícitos (troca desfeita ou encaminhamento aprovado sem mudança pelo atendente) | `trace_id`, versão da rubrica do juiz | painel, ao lado das métricas de operação |

A soma de tokens por usuário fica com a aplicação, e não com a plataforma de observabilidade, porque uma métrica cortada por cliente teria milhares de valores.

---

## 7. Guardar, avaliar e o que fica fora

**Volume.** Pela conta da aula, V = conversas por dia × *spans* por conversa × tamanho do *span*: 500 × 8 × ~1 KB ≈ 4 MB por dia, uns 120 MB por mês. É tão pouco que guardamos **100% dos *traces***, sem amostragem. Se o volume crescer muito, passamos a guardar todos os que terminaram mal e uma parte dos normais.

**Retenção.**

| O quê | Por quanto tempo | Por quê |
|---|---|---|
| *traces* sem `registrar_troca` | 30 dias | diagnóstico e comparação entre versões dentro do mês |
| *traces* com `registrar_troca` | 1 ano | uma troca pode ser contestada meses depois, e o *trace* é a prova de como a decisão foi tomada (aula de governança) |
| logs INFO | 30 dias | diagnóstico do dia a dia |
| logs WARN e ERROR | 90 dias | comparar incidentes parecidos |
| logs DEBUG | 7 dias, e só quando ligados | investigação pontual |

O limite máximo vem da LGPD: dado pessoal só pelo tempo da finalidade. Por isso o identificador do cliente vai nos *spans* e nos logs como *hash*, e não como CPF ou e-mail. Os prazos acima são uma proposta e precisam ser validados com o jurídico da loja.

**Qualidade.** Duas fontes de rótulo:

- o juiz validado na aula de *evals*, sobre uns 5% das conversas (umas 25 por dia);
- rótulos implícitos, que não custam nada e cobrem todas as conversas que passam por um atendente: troca aprovada pelo agente e depois desfeita (o agente errou), e caso encaminhado que o atendente aprova sem mudar nada (o agente podia ter resolvido).

Cada erro encontrado volta para o *golden dataset* como caso novo.

**O que fica fora.** O texto que o cliente escreve não vai para os *spans* nem para os logs, porque tem dado pessoal. O juiz precisa do conteúdo; para isso, ele fica num armazenamento separado, ligado ao *trace* pelo `trace_id`, por 30 dias.

**Idempotência.** O `registrar_troca` usa uma chave montada com o número do pedido e o motivo. Assim, se a chamada se repetir, a troca não é registrada duas vezes.

**O plano é hipótese.** Só vamos saber se os sinais funcionam quando provocarmos as falhas de propósito: mandar um pedido fora do prazo, derrubar o sistema de pedidos, atrasar a resposta do modelo, gastar tokens até bater no limite.

---

## Anexo A. SLI, SLO e SLA

- **SLI** é o que a gente mede (por exemplo, a % de respostas em até 10 s).
- **SLO** é a meta interna para esse número, numa janela de tempo. É o que o time técnico persegue.
- **SLA** é o acordo formal com quem usa o serviço, com consequência se não for cumprido. No nosso caso, é o combinado entre o time do agente e a área de atendimento da loja. O SLA é sempre mais folgado que o SLO, para dar tempo de corrigir antes de quebrar o acordo.

| Item | SLA (mensal) | Se não cumprir |
|---|---|---|
| tempo de resposta | 90% das respostas em até 10 s | o time técnico apresenta a causa e um plano de correção; se repetir no mês seguinte, a loja pode voltar a mandar as trocas direto para o atendente |
| disponibilidade | agente respondendo em 99% do tempo | durante a queda, todas as conversas vão para o atendente humano automaticamente |
| trocas fora da política | menos de 1% das aprovadas | a aprovação automática é suspensa e toda troca passa por confirmação humana até a correção |

---

## Anexo B. Limite de tokens por usuário

O limite de 6 passos já segura o custo de cada conversa, mas não segura um usuário (ou um robô) que abre dezenas de conversas seguidas. Para isso colocamos um limite de tokens por cliente: a soma de `gen_ai.usage.input_tokens` e `gen_ai.usage.output_tokens` de todas as conversas dele no dia.

| Faixa | O que acontece | Log |
|---|---|---|
| até 40.000 tokens no dia | normal | INFO no fim de cada conversa, com os tokens gastos |
| 40.000 tokens (80% do limite) | o agente continua atendendo, mas fica registrado que o usuário está perto do limite | WARN `limite_tokens_proximo` |
| 50.000 tokens (limite) | o acesso do usuário ao agente é cortado até a meia-noite. Se ele estiver no meio de uma conversa, ela termina com `agente.termino = limite_tokens` e vai para o atendente humano. Nas próximas tentativas do dia, o chat manda direto para o atendente, sem chamar o modelo. | WARN `usuario_bloqueado_tokens` |

Por que 50 mil: uma conversa normal gasta uns 8 mil tokens, e a pior possível (6 passos) uns 14 mil. Então 50 mil dá umas 6 conversas normais por dia, e ninguém precisa pedir 6 trocas no mesmo dia. No custo, 50 mil tokens dão uns R$ 0,03 por usuário por dia, então mesmo um robô fica com o prejuízo limitado.

Cortar o acesso ao agente não quer dizer deixar o cliente sem atendimento: ele sempre cai num atendente humano.

No painel acompanhamos quantos usuários são bloqueados por dia. Se esse número subir de repente, ou é abuso, ou alguma versão nova passou a gastar mais tokens por conversa e o limite está pegando gente normal; nesse caso, o limite precisa ser revisto.

---

## Anexo C. Níveis de log

Os logs são estruturados (em JSON) e todos levam o `trace_id`, para dar para ir do log direto para o *trace* da conversa. O texto que o cliente escreveu não vai para o log em nenhum nível.

| Nível | Quando usar | Exemplos no agente de trocas | Em produção |
|---|---|---|---|
| DEBUG | detalhe para investigar um problema específico | argumentos e retorno resumido de cada ferramenta, número do passo, tokens de cada chamada | desligado; liga só durante uma investigação, por pouco tempo |
| INFO | o caminho normal, o que aconteceu na conversa | conversa iniciada; ferramenta chamada; decisão tomada; conversa finalizada com término, duração, tokens e custo | ligado |
| WARN | algo saiu do normal, mas o cliente foi atendido | resposta acima de 10 s; `orcamento_esgotado`; usuário em 80% do limite de tokens; usuário bloqueado; retentativa de ferramenta; política padrão de 30 dias usada | ligado |
| ERROR | algo falhou e precisa de alguém | ferramenta devolveu erro; `erro_fatal`; provedor fora do ar; `registrar_troca` falhou; resposta acima de 30 s; troca aprovada fora do prazo | ligado |

Exemplo de linha de log:

```json
{"nivel": "WARN", "evento": "resposta_lenta", "trace_id": "4bf92f3577b34da6",
 "duracao_ms": 12400, "span_mais_lento": "chat", "versao": "1.3.0", "versao_prompt": "p7"}
```

A regra que seguimos para escolher o nível: se ninguém precisa fazer nada, é INFO; se alguém deveria olhar quando tiver tempo, é WARN; se alguém precisa agir agora, é ERROR. Os alertas da seção 5 são montados em cima de WARN e ERROR, nunca de INFO.

---

## Anexo D. Custo

| Item | Valor estimado |
|---|---|
| tokens por conversa normal (4 chamadas) | uns 7.800 de entrada (o histórico vai sendo reenviado a cada volta) e uns 320 de saída |
| custo por conversa | perto de R$ 0,005 |
| custo por dia (500 conversas) | perto de R$ 2,50 |
| custo por mês (umas 15 mil conversas) | perto de R$ 75 |
| meta (SLO) | até R$ 0,01 por conversa |
| máximo por usuário por dia (limite de tokens) | uns R$ 0,03 |

Esses valores usam o preço de um modelo pequeno e ainda precisam ser confirmados na tabela do provedor e depois no painel, com conversas reais. O custo de cada conversa é a soma dos tokens de todos os `chat` do *trace* multiplicada pelo preço do modelo. Como o limite de 6 passos e o limite de tokens por usuário seguram o custo por conversa e por pessoa, quando o custo total sobe é quase sempre por volume (muita conversa) ou por uma versão nova que gasta mais.

---

## Resposta à questão

**Como saber, sem ler o código, se o agente de trocas está funcionando, e se está funcionando bem?**

Para saber se está funcionando, olhamos a porcentagem de conversas que terminam normalmente, os erros de cada ferramenta, se as respostas saem em até 10 segundos e o custo por conversa. Quase tudo isso sai dos *spans* padrão do OpenTelemetry e dos logs WARN e ERROR.

Saber se está funcionando bem é outra coisa, e taxa de erro não mostra. A maioria das falhas desse agente é silenciosa: ele responde normal, só que responde pior. Por isso a gente depende dos atributos que a própria aplicação grava: quantas trocas foram resolvidas sem atendente (meta de 60%), quantas foram aprovadas fora da política (meta abaixo de 1%) e se as decisões estão certas, pelo juiz e pelo que os atendentes desfazem ou aprovam. Sempre olhando pelo carimbo de versão (código, prompt e modelo), para saber qual mudança causou a piora.
