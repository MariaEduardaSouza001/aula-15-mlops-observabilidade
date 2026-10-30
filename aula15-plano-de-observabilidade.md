# Plano de observabilidade — agente de trocas

Antes de tudo, uma decisão que usamos no plano inteiro: cada conversa de troca é uma execução, então cada conversa vira um *trace*, com o `invoke_agent` na raiz. Todas as métricas abaixo são "por conversa". Com umas 500 conversas por dia, uma janela de 28 dias dá algo perto de 14 mil conversas.

---

## 1. Perguntas

| Interessado | Pergunta | Decisão que a resposta muda |
|---|---|---|
| o negócio | Quantas trocas o agente resolve sozinho (aprovando ou recusando), sem passar para o atendente? Estamos perto dos 60% prometidos? | Se ficar bem abaixo, a loja precisa rever o case: ver se o agente está encaminhando demais ou se a política tem casos que ele não consegue decidir. |
| a operação | Quantas conversas pararam no meio na última hora, por estourar os 6 passos ou por erro? E o cliente está esperando mais de 10 segundos pela resposta? | Voltar para a versão anterior, corrigir a integração que está falhando ou investigar lentidão do provedor. |
| quem paga | Quanto custa uma conversa, e o que pesa mais nesse custo? Tem algum usuário gastando muito mais tokens que o normal? | Encurtar o que as ferramentas devolvem, testar um modelo mais barato, mexer no limite de passos ou bloquear quem está abusando. |
| quem responde pelo sistema | Alguma troca foi registrada fora da política, ou o `registrar_troca` foi chamado sem o agente ter consultado o pedido e a política antes? | Parar a aprovação automática e exigir que um atendente confirme antes de registrar, até achar a causa. |

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

Na raiz ficam `gen_ai.agent.name = agente-de-trocas` e `gen_ai.provider.name`. Em cada `chat` ficam o modelo, os tokens de entrada e saída e o `finish_reasons`; em cada `execute_tool`, o nome da ferramenta e o `gen_ai.tool.call.id`. Se a troca é recusada ou encaminhada, o `registrar_troca` não aparece. Se o orçamento acaba, aparecem 6 `chat` e a conversa vai para o atendente.

A duração do `invoke_agent` é o tempo de resposta que o cliente sente, do envio da mensagem até a resposta do agente. É essa duração que usamos no SLO de 10 segundos. A duração de cada `chat` e de cada `execute_tool` mostra onde o tempo foi gasto.

**Atributos próprios na raiz:**

| Atributo | Valores possíveis | Para que serve |
|---|---|---|
| `troca.decisao` | aprovada, recusada, encaminhada, ausente | É com ele que dá para medir a promessa: resolvido sem atendente = aprovada + recusada. O "ausente" pega os casos em que o agente respondeu, mas sem decisão clara. |
| `agente.termino` | respondeu, orcamento_esgotado, limite_tokens, erro_fatal | Separa as conversas que terminaram bem das que pararam no meio. Junto com a decisão, mostra se um encaminhamento foi porque o agente ficou em dúvida (ok) ou porque acabaram os passos ou os tokens do usuário (problema). |
| `troca.dentro_do_prazo` | sim, nao, nao_se_aplica | A própria aplicação calcula, comparando a data da compra com o prazo da categoria. Não deixamos isso com o modelo. Assim dá para checar o prazo em todas as trocas aprovadas, sem precisar de juiz. |

Esses três atributos têm poucos valores, então podem ser usados para cortar as métricas. Já o número do pedido e o identificador do cliente ficam só no *span*, porque como dimensão de métrica iam explodir a cardinalidade.

---

## 3. Modos de falha

| Componente | Falha | Ruidosa ou silenciosa | Sinal que a torna visível |
|---|---|---|---|
| modelo + `registrar_troca` | Aprova e registra uma troca que estava fora da política (prazo vencido, categoria sem troca). Ou chama a ferramenta quando não devia: sem consultar a política antes, para outro pedido, ou duas vezes na mesma conversa. | Silenciosa. A troca entra no sistema normalmente e ninguém reclama, pelo menos o cliente não. | `troca.dentro_do_prazo = nao` em troca aprovada; `registrar_troca` sem `consultar_politica` antes no mesmo *trace*; mais de um `registrar_troca` por *trace*. As outras condições da política (além do prazo) só aparecem na avaliação por amostragem. |
| laço do agente | Fica repetindo consulta, não chega a uma decisão e estoura os 6 passos. | Silenciosa. O cliente é encaminhado e não aparece erro nenhum, mas a taxa de resolução cai. | `agente.termino = orcamento_esgotado` |
| modelo | Começa a encaminhar casos simples que ele conseguiria resolver, por exemplo depois de alguém mudar o prompt ou de o provedor trocar o modelo. | Silenciosa, e vai piorando aos poucos. Latência, custo e erro continuam iguais. | Aumento de `troca.decisao = encaminhada` com `agente.termino = respondeu`, olhando por versão. |
| `consultar_pedido` | O sistema de pedidos da loja fica fora do ar ou o número do pedido não existe. | Ruidosa, a ferramenta devolve erro. | `execute_tool` com status ERROR, separado por `gen_ai.tool.name`; `agente.termino = erro_fatal` quando o agente não consegue seguir. |
| provedor do modelo | A API fica lenta: a resposta chega certa, mas demora 15, 20 segundos. | Silenciosa para a taxa de erro (não dá erro), mas o cliente sente na hora e pode desistir do chat. | Duração do `invoke_agent` acima de 10 s (log WARN), e a duração dos `chat` mostra que o tempo foi gasto no modelo e não nas ferramentas. |

Seguindo a dica do enunciado, achamos mais um caso: se a categoria não estiver cadastrada, o `consultar_politica` pode devolver o prazo padrão de 30 dias, e o agente decide em cima de uma regra errada. Hoje não temos sinal para isso. Para ter, a ferramenta precisaria registrar no *span* se usou a política da categoria ou a padrão.

---

## 4. Objetivos

Primeiro, a diferença entre os três termos, do jeito que usamos aqui:

- **SLI** é o que a gente mede (por exemplo, a % de respostas em até 10 s).
- **SLO** é a meta interna para esse número, numa janela de tempo. É o que o time técnico persegue.
- **SLA** é o acordo formal com quem usa o serviço, com consequência se não for cumprido. No nosso caso, é o combinado entre o time do agente e a área de atendimento da loja. O SLA é sempre mais folgado que o SLO, para dar tempo de corrigir antes de quebrar o acordo.

| Tipo | Indicador (SLI) | Objetivo (SLO) | Janela |
|---|---|---|---|
| operacional (tempo de resposta) | % de conversas em que o agente responde em até 10 s (duração do `invoke_agent`) | pelo menos 95% | 28 dias |
| operacional (término) | % de conversas com `agente.termino = respondeu` | pelo menos 95% (sobram uns 5%, umas 700 conversas no período, de margem) | 28 dias |
| a promessa da loja | % de conversas resolvidas sem atendente (decisão aprovada ou recusada, término respondeu) | pelo menos 60% | 28 dias |
| custo | custo médio por conversa | até R$ 0,01 | 28 dias |

A promessa tem duas partes, e não adianta cumprir os 60% aprovando troca que não devia. Por isso acompanhamos também a parte de conformidade: das trocas aprovadas, menos de 1% fora da política, em 28 dias. Aqui contam as aprovadas com `troca.dentro_do_prazo = nao` mais as que o juiz reprovar na amostra.

**SLA combinado com a área de atendimento (mensal):**

| Item | SLA | Se não cumprir |
|---|---|---|
| tempo de resposta | 90% das respostas em até 10 s | o time técnico apresenta a causa e um plano de correção; se repetir no mês seguinte, a loja pode voltar a mandar as trocas direto para o atendente |
| disponibilidade | agente respondendo em 99% do tempo | durante a queda, todas as conversas vão para o atendente humano automaticamente |
| trocas fora da política | menos de 1% das aprovadas | a aprovação automática é suspensa e toda troca passa por confirmação humana até a correção |

Por que 10 segundos: numa conversa normal o modelo é chamado 4 vezes, cada chamada leva por volta de 1 a 2 s, e as ferramentas somam menos de 1 s. Isso dá uns 5 a 7 s. O limite de 10 s deixa uma folga, mas pega quando o provedor ou alguma ferramenta começa a travar.

---

## 5. Alertas

| Alerta | Condição e limiar | SLO de origem | Quem é avisado | O que faz |
|---|---|---|---|---|
| custo | Custo médio por conversa na última hora acima de R$ 0,02 (o dobro da meta), ou custo total da hora acima de R$ 1,20 | custo médio de até R$ 0,01 por conversa | responsável pelo agente, com cópia para quem paga | Olha o painel por versão e por modelo. Se cada conversa ficou mais cara, compara com a versão anterior e volta se for o caso. Se o que subiu foi o número de conversas, vê se não é um robô ou abuso e confere os bloqueios por limite de tokens (seção 6). |
| degradação silenciosa | Mais de 15% das conversas terminando em `orcamento_esgotado` nas últimas 6 horas. Nesse ritmo, a margem de 5% do mês acaba em menos de 10 dias. | pelo menos 95% de conversas com término normal | responsável pelo agente (quem estiver de plantão) | Vê se a piora começou com alguma versão nova e, se sim, volta. Abre alguns *traces* que estouraram o orçamento para ver qual ferramenta está se repetindo. Se nada mudou do nosso lado, desconfia que o provedor mudou o modelo e roda a suíte de *evals* de novo. |
| tempo de resposta | Toda resposta acima de 10 s gera um log WARN e aparece no painel. O alerta para a pessoa dispara quando mais de 10% das respostas da última hora passam de 10 s, ou quando qualquer resposta passa de 30 s (aí o cliente já desistiu, e é log ERROR). | pelo menos 95% das respostas em até 10 s | responsável pelo agente (plantão) | Abre os *traces* lentos e vê qual *span* demorou. Se foi o `chat`, olha a página de status do provedor e, se continuar, troca para um modelo de reserva. Se foi uma ferramenta, aciona o time do sistema de pedidos. Se a lentidão passou a acontecer depois de uma versão nova (prompt maior, mais passos), volta a versão. |

Não colocamos o alerta para disparar a cada resposta lenta porque, com umas 40 conversas por hora, o plantão seria acordado várias vezes por casos isolados que se resolvem sozinhos. A resposta lenta isolada fica registrada no log e no painel; quem é acordado é só quem precisa agir.

No alerta de degradação usamos 6 horas e não 1 hora pelo mesmo motivo: poucos casos já fariam o alerta disparar à toa. Fica de fora do mínimo pedido, mas achamos importante: qualquer troca aprovada fora do prazo deveria avisar na hora o responsável pelo atendimento, para desfazer a troca e, se repetir, ligar a confirmação humana.

---

## 6. Limite de tokens por usuário

O limite de 6 passos já segura o custo de cada conversa, mas não segura um usuário (ou um robô) que abre dezenas de conversas seguidas. Para isso colocamos um limite de tokens por cliente, contado pela soma de `gen_ai.usage.input_tokens` e `gen_ai.usage.output_tokens` de todas as conversas dele no dia.

| Faixa | O que acontece | Log |
|---|---|---|
| até 40.000 tokens no dia | normal | INFO no fim de cada conversa, com os tokens gastos |
| 40.000 tokens (80% do limite) | o agente continua atendendo, mas fica registrado que o usuário está perto do limite | WARN `limite_tokens_proximo` |
| 50.000 tokens (limite) | o acesso do usuário ao agente é cortado até a meia-noite. Se ele estiver no meio de uma conversa, ela termina com `agente.termino = limite_tokens` e vai para o atendente humano. Nas próximas tentativas do dia, o chat já manda direto para o atendente, sem chamar o modelo. | WARN `usuario_bloqueado_tokens` |

Por que 50 mil: uma conversa normal gasta uns 8 mil tokens, e a pior possível (6 passos) uns 14 mil. Então 50 mil dá umas 6 conversas normais por dia, e ninguém precisa pedir 6 trocas no mesmo dia. No custo, 50 mil tokens dão uns R$ 0,03 por usuário por dia, então mesmo um robô fica com o prejuízo limitado.

Cortar o acesso ao agente não quer dizer deixar o cliente sem atendimento: ele sempre cai num atendente humano. Assim, um cliente legítimo que bater no limite não é prejudicado.

O que acompanhamos no painel: quantos usuários foram bloqueados por dia. Se esse número subir de repente, é sinal de abuso ou de que alguma versão nova passou a gastar mais tokens por conversa (aí o limite estaria bloqueando gente normal e precisa ser revisto). O identificador do cliente fica no *span* e no log, nunca como dimensão de métrica.

---

## 7. Níveis de log

Os logs são estruturados (em JSON) e todos levam o `trace_id`, para dar para ir do log direto para o *trace* da conversa. O texto que o cliente escreveu não vai para o log em nenhum nível, porque tem dado pessoal.

| Nível | Quando usar | Exemplos no agente de trocas | Em produção | Retenção |
|---|---|---|---|---|
| DEBUG | detalhe para investigar um problema específico | argumentos e retorno resumido de cada ferramenta, número do passo, tokens de cada chamada | desligado; liga só durante uma investigação, por pouco tempo | 7 dias |
| INFO | o caminho normal, o que aconteceu na conversa | conversa iniciada; ferramenta chamada; decisão tomada (`troca.decisao`); conversa finalizada com término, duração, tokens e custo | ligado | 30 dias |
| WARN | algo saiu do normal, mas o cliente foi atendido | resposta acima de 10 s; `orcamento_esgotado`; usuário em 80% do limite de tokens; usuário bloqueado; retentativa de ferramenta; política padrão de 30 dias usada | ligado | 90 dias |
| ERROR | algo falhou e precisa de alguém | ferramenta devolveu erro; `erro_fatal`; provedor fora do ar; `registrar_troca` falhou; resposta acima de 30 s; troca aprovada fora do prazo | ligado | 90 dias |

Exemplo de linha de log:

```json
{"nivel": "WARN", "evento": "resposta_lenta", "trace_id": "4bf92f3577b34da6",
 "duracao_ms": 12400, "span_mais_lento": "chat", "versao": "1.3.0"}
```

A regra que seguimos para escolher o nível: se ninguém precisa fazer nada, é INFO; se alguém deveria olhar quando tiver tempo, é WARN; se alguém precisa agir agora, é ERROR. O alerta da seção 5 é montado em cima de WARN e ERROR, nunca de INFO.

---

## 8. Custo

Juntando o custo num lugar só:

| Item | Valor estimado |
|---|---|
| tokens por conversa normal (4 chamadas) | uns 7.800 de entrada (o histórico vai sendo reenviado a cada volta) e uns 320 de saída |
| custo por conversa | perto de R$ 0,005 |
| custo por dia (500 conversas) | perto de R$ 2,50 |
| custo por mês (uns 15 mil conversas) | perto de R$ 75 |
| meta (SLO) | até R$ 0,01 por conversa |
| máximo por usuário por dia (limite de tokens) | uns R$ 0,03 |

Esses valores usam o preço de um modelo pequeno e ainda precisam ser confirmados na tabela do provedor e depois no painel, com conversas reais. O custo de cada conversa é calculado somando os tokens de todos os `chat` do *trace* e multiplicando pelo preço do modelo. Como o limite de 6 passos e o limite de tokens por usuário seguram o custo por conversa e por pessoa, se o custo total subir é quase sempre por volume (muita conversa) ou por uma versão nova que gasta mais.

---

## Algumas decisões a mais

- Guardamos todos os *traces* que terminaram mal, tiveram erro ou demoraram mais de 10 s, e só uma parte dos normais. Como a gente só sabe se a conversa terminou bem no final, a amostragem precisa ser na cauda.
- Uma amostra pequena das conversas (uns 5%) passa pelo juiz da aula de *evals*, para ter uma taxa de acerto no painel.
- O `registrar_troca` usa uma chave de idempotência montada com o número do pedido e o motivo. Assim, se a chamada se repetir, a troca não é registrada duas vezes.
- Por enquanto, tudo isso é hipótese. Só vamos saber se os sinais funcionam quando provocarmos as falhas de propósito: mandar um pedido fora do prazo, derrubar o sistema de pedidos, atrasar a resposta do modelo, gastar tokens até bater no limite.

---

## Resposta à questão

**Como saber, sem ler o código, se o agente de trocas está funcionando, e se está funcionando bem?**

Para saber se está funcionando, olhamos a porcentagem de conversas que terminam normalmente, os erros de cada ferramenta, se as respostas saem em até 10 segundos e o custo por conversa. Quase tudo isso sai dos *spans* padrão do OpenTelemetry e dos logs WARN e ERROR.

Saber se está funcionando bem é outra coisa, e taxa de erro não mostra. A maioria das falhas desse agente é silenciosa: ele responde normal, só que responde pior. Por isso a gente depende dos atributos que a própria aplicação grava: quantas trocas foram resolvidas sem atendente (meta de 60%), quantas foram aprovadas fora da política (meta abaixo de 1%) e a taxa de acerto da amostra avaliada pelo juiz. Sempre olhando por versão, para saber qual mudança causou a piora.
