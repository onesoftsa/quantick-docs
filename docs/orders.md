# Order Routing API

O controle de ordens está estruturado nos módulos a seguir:

-   *oms*: contem as funcionalidades para gerenciamento de ordens

-   *position*: gerencia a posição do usuário

-   *risk*: apresenta o risco configurado

## OMS

O módulo *oms* contem todas as transações (de todos os ativos e tipos) e
os respectivos assessores para sua manipulação. Esse objeto será o
responsável por guardar os estados das transações de ordens e sinalizar
o algoritmo à cada mudança.

O objeto que representa uma ordem é denominado OrderEntry. Um objeto
deste tipo contém todas as informações que descrevem o estado atual da
ordem e suas propriedades. Do mesmo modo, fornece certos métodos para
facilitar a sua análise e modificacação

#### OrderEntry

Uma ordem está definida pelo objeto *OrderEntry* o qual possui as
seguintes propriedades e funções:

##### Propriedades

| **Nome da propriedade**  | **Descrição**                                                                                                                                                                                                                                  |
|--------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| unique_id                | Número inteiro identificador único da ordem gerado pelo neutrino para auxiliar o programador. Esse id se mantém o mesmo durante toda a vida da ordem e não faz referência ao "OrderID" gerado pela bolsa                                       |
| side                     | Enumerado do tipo *Side* (Será descrito abaixo) e indica se é uma ordem de compra ou venda.                                                                                                                                                    |
| type                     | Enumerado do tipo *OrderType* (Será descrito abaixo) que indica o tipo da ordem (*Limite, Mercado* ou *Start/Stop).*                                                                                                                           |
| time_in_force            | Enumerado do tipo *TimeInForce* (Será descrito abaixo) que representa o Campo 59 do FIX (a validade da ordem - Dia, VAC ou GTD).                                                                                                               |
| status                   | Enumerado do tipo *OrderStatus* (será descrito abaixo) que é a combinação entre os campos 39 e 150 do FIX.                                                                                                                                     |
| price                    | Valor numérico que representa o preço Limite da ordem.                                                                                                                                                                                         |
| last_price               | Valor numérico que representa o preço da última execução.                                                                                                                                                                                      |
| trigger_price            | Valor numérico que representa o preço de gatilho da ordem (caso a ordem não seja de gatilho, será zero).                                                                                                                                       |
| quantity                 | Valor numérico que representa o quantidade original da ordem.                                                                                                                                                                                  |
| last_quantity            | Valor numérico que representa o quantidade executada na transação atual.                                                                                                                                                                       |
| filled_quantity          | Valor numérico que representa o quantidade total executada.                                                                                                                                                                                    |
| leaves_quantity          | Valor numérico que representa o quantidade ainda disponível no book.                                                                                                                                                                           |
| transact_time            | Valor numérico que representa o timestamp da interação de ordem.                                                                                                                                                                               |
| account                  | String contendo a conta para a qual a estratégia está operando.                                                                                                                                                                                |
| symbol                   | String com o nome do contrato.                                                                                                                                                                                                                 |
| client_order_id          | String com o campo 11 do FIX que representa o *id* da transação. Esse *id* será gerenciado pelo Neutrino. \*\*                                                                                                                                 |
| original_client_order_id | String com o campo 41 do FIX que representa o *client_order_id* da transação anterior. Também será gerenciado pelo Neutrino.                                                                                                                   |
| order_id                 | String com o campo 37 do FIX que representa o *id* único da ordem atribuído pela *Bolsa,* e por isso mesmo só será preenchido em ordens que foram confirmadas, caso contrário será vazio.                                                      |
| secondary_order_id       | String com o *id* da oferta no book de ofertas. Quando presente, esse *id* vem pelo *marketdata* mas não é garantido que ele seja disponibilizado por todos os *feeders* de *marketdata.* Será branco caso não o *feeder* não o disponibilize. |

<sub>
** Observação: A string gerada pelo neutrino como o ClOrdID (client_order_id) contém o tranche_id do algoritmo codificado nela. Para recuperar o valor original, no python, pode-se rodar `int(<ClOrdID>.split('.')[1][2:8], 16)`, onde <ClOrdID> é uma string. O valor obtido deve corresponder ao PID presente no arquivo de configuração da estratégia.
</sub>

##### Enumerados

<table>
<thead>
<tr class="header">
<th><strong>Nome</strong></th>
<th><strong>Descrição</strong></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><code>OrdeSide</code></td>
<td><p> BID: identifica uma ordem de compra</p>
<p> ASK: identifica uma ordem de venda</p></td>
</tr>
<tr class="even">
<td><code>OrderType</code></td>
<td> LIMIT, MARKET, STOP, STOP_LIMIT</td>
</tr>
<tr class="odd">
<td><code>TimeInForce</code></td>
<td> DAY, FAK, FOK</td>
</tr>
<tr class="even">
<td><code>OrderStatus</code></td>
<td>
<ul>
<li><p>WAIT: estado inicial de toda ordem antes da confirmação ou rejeição</p></li>
<li><p>WAIT_CANCEL: estado após solicitação de cancelamento e antes da confirmação/rejeição</p></li>
<li><p>ACTIVE: ordem ativa aceita pela bolsa e apresentada no book</p></li>
<li><p>WAIT_REPLACE: estado retornar após solicitação de alteração da ordem</p></li>
<li><p>REPLACED: estado de confirmação de alteração da ordem, a qual é apresentada no book com as mudanças solicitadas</p></li>
<li><p>FILLED: ordem executada</p></li>
<li><p>PARTIAL_FILLED: ordem parcialmente executada, na qual a quantidade solicitada não foi atingida porém somente parte dela.</p></li>
<li><p>CANCELED: ordem cancelada à solicitação do usuário</p></li>
<li><p>REJECTED: ordem rejeitada</p></li>
</ul>
</td>
</tr>
</tbody>
</table>

<br>

##### Métodos

| **Nome do método**   | **Descrição**                                                                                                                                                        |
|----------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `is_execution()`     | Retorna *True* caso a ordem represente uma execução, *False* caso contrário                                                                                          |
| `is_dead()`          | Retorna *True* caso a ordem esteja em um estado final irreversível (cancelada, executada ou rejeitada caso a ordem nunca tenha sido aceita), *False* caso contrário. |
| `is_alive()`         | Retorna *True* caso a ordem esteja num estado que pode ser alterado por ação do algoritmo ou do mercado (ativa, modificada), *False* caso contrário.                 |
| `is_pending()`       | Retorna *True* caso a ordem esteja em *inflight* (wait, wait cancel ou wait replace), *False* caso contrário.                                                        |
| `replace()`          |                                                                                                                                                                      |
| `cancel()`           |                                                                                                                                                                      |

<br>

#### Envio de ordens

Cada função de envio de ordens retonar um número que se for positivo
representará o id único da ordem que foi criada (apenas no caso de ordem
nova, já que esse id nunca mudará mesmo que a ordem seja
modificada/cancelada), este identificador é denominado *unique_id.*

Para envio, mudança ou cancelamento de ordens podem ser utilizadas as
funções a seguir:

<table>
<thead>
<tr class="header">
<th><strong>Nome da função</strong></th>
<th><strong>Descrição</strong></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><code>send(symbol=, side=, price=, quantity=, time_in_force=TimeInForce.DAY)</code></td>
<td><p>Envia uma ordem de tipo limite, usando como argumentos:</p>
<ul>
<li><p><em>symbol: </em>nome do ativo</p></li>
<li><p><em>side:</em> OrderSide::BID ou OrderSide::ASK</p></li>
<li><p><em>price</em>: valor do preço</p></li>
<li><p><em>quantity</em>: quantidade inteira múltipla do lote mínimo</p></li>
<li><p><em>time_in_force</em>: validade da ordem determinado pelo enumerado <em>TimeInForce, </em>default DAY<em>.</em></p></li>
</ul>
<p>Retorna <em>unique_id (</em>inteiro &gt; 0) no caso de sucesso, ou código de erro caso contrário.</p></td>
</tr>
<tr class="even">
<td><code>send(symbol=, side=, trigger_price=, price=, quantity=, time_in_force=TimeInForce.DAY)</code></td>
<td><p>Envia uma ordem de tipo gatilho, usando como argumentos:</p>
<ul>
<li><p><em>symbol: </em>nome do ativo</p></li>
<li><p><em>side:</em> OrderSide::BID ou OrderSide::ASK</p></li>
<li><p><em>price</em>: valor do preço</p></li>
<li><p><em>trigger_price</em>: valor do preço gatilho</p></li>
<li><p><em>quantity</em>: quantidade inteira múltipla do lote mínimo</p></li>
<li><p><em>time_in_force</em>: validade da ordem determinado pelo enumerado <em>TimeInForce, </em>default DAY.</p></li>
</ul>
<p>Retorna <em>unique_id (</em>inteiro &gt; 0) no caso de sucesso, ou código de erro caso contrário.</p></td>
</tr>
<tr class="odd">
<td><code>send(symbol=, side=, quantity=)</code></td>
<td><p>Envia uma ordem de tipo mercado, usando como argumentos:</p>
<ul>
<li><p><em>symbol: </em>nome do ativo</p></li>
<li><p><em>side:</em> OrderSide::BID ou OrderSide::ASK</p></li>
<li><p><em>quantity</em>: quantidade inteira múltipla do lote mínimo</p></li>
</ul>
<p>Retorna <em>unique_id (</em>inteiro &gt; 0) no caso de sucesso, ou código de erro caso contrário.</p></td>
</tr>
<tr class="even">
<td><code>is_dead()</code></td>
<td>Retorna <em>True</em> caso a ordem esteja em um estado final irreversível (CANCELED, FILLED, REJECTED ou REJECTED caso a ordem nunca tenha sido aceita), <em>False</em> caso contrário.</td>
</tr>
<tr class="odd">
<td><code>is_alive()</code></td>
<td>Retorna <em>True</em> caso a ordem esteja num estado que pode ser alterado por ação do algoritmo ou do mercado (ACTIVE, REPLACED), <em>False</em> caso contrário.</td>
</tr>
<tr class="even">
<td><code>is_pending()</code></td>
<td>Retorna <em>True</em> caso a ordem esteja em <em>inflight</em> (WAIT, WAIT_CANCEL ou WAIT_REPLACE), <em>False</em> caso contrário.</td>
</tr>
<tr class="odd">
<td><p><code>replace(order=, price=, quantity=, time_in_force=)</code></p>
<p><code>replace(order=, price=, quantity=)</code></p></td>
<td><p>Modifica uma ordem de tipo limite, usando como argumentos:</p>
<ul>
<li><p><em>order</em>: objeto do tipo <em>OrderEntry</em> podendo ser obtido pelas funções de consulta ou pela callback <em>order_update</em></p></li>
<li><p><em>price</em>: valor do preço</p></li>
<li><p><em>quantity</em>: quantidade inteira múltipla do lote mínimo</p></li>
<li><p><em>time_in_force</em>: validade da ordem determinado pelo enumerado <em>TimeInForce, </em>default é o valor do TIF da ordem em questão.</p></li>
</ul>
<p>Retorna <em>unique_id (</em>inteiro &gt; 0) no caso de sucesso, ou código de erro caso contrário.</p></td>
</tr>
<tr class="even">
<td><p><code>replace(symbol=, trigger_price=, price=, quantity=, time_in_force=)</code></p>
<p><code>replace(symbol=, trigger_price=, price=, quantity=)</code></p></td>
<td><p>Modifica uma ordem de tipo gatilho, usando como argumentos:</p>
<ul>
<li><p><em>order</em>: objeto do tipo <em>OrderEntry</em> podendo ser obtido pelas funções de consulta ou pela callback <em>order_update</em></p></li>
<li><p><em>price</em>: valor do preço</p></li>
<li><p><em>trigger_price</em>: valor do preço gatilho</p></li>
<li><p><em>quantity</em>: quantidade inteira múltipla do lote mínimo</p></li>
<li><p><em>time_in_force</em>: validade da ordem determinado pelo enumerado <em>TimeInForce, </em>default é o valor do TIF da ordem em questão.</p></li>
</ul>
<p>Retorna <em>unique_id (</em>inteiro &gt; 0) no caso de sucesso, ou código de erro caso contrário.</p></td>
</tr>
<tr class="odd">
<td><code>cancel(&lt;order&gt;)</code></td>
<td><p>Cancela determinada ordem, sendo que o cancelamento é comum à todos os tipos de ordens. Usa como argumento:</p>
<ul>
<li><p><em>order</em>: objeto do tipo <em>OrderEntry</em> podendo ser obtido pelas funções de consulta ou pela callback <em>order_update</em></p></li>
</ul>
<p>Retorna <em>unique_id (</em>inteiro &gt; 0) no caso de sucesso, ou código de erro caso contrário.</p></td>
</tr>
<tr class="even">
<td><code>cancel_all(&lt;symbol&gt;, &lt;side&gt;=NONE_SIDE, &lt;price&gt;=NONE_)</code></td>
<td><p>Cancela o grupo de ordens do ativo <em>symbol</em>, sendo possível filtrar pelo lado e/ou preço.</p>
<ul>
<li><p><em>symbol</em>: nome do ativo</p></li>
<li><p><em>side</em>: OrderSide::BID ou OrderSide::ASK</p></li>
<li><p><em>price:</em> valor do preço</p></li>
</ul>
<p>Retorna <em>unique_id (</em>inteiro &gt; 0) no caso de sucesso, ou código de erro caso contrário.</p>
</tr>
</tbody>
</table>

<br>

No caso do método *send *recomenda-se o uso dos argumentos com os nomes
para evitar conflitos entre 

Caso o valor de retorno no envio, mudança ou cancelamento de ordens seja
um número negativo ele representa um código de falha a ser tratado pelo
programador. 

O acontecimento de qualquer falha indicará que essa ordem não existe no
mercado e por isso não haverá qualquer tipo de *report* via
*order_update(order)* para ela.

Os possíveís códigos de erro são:

| **Valor de retorno**  | **Descrição**                                          |
|-----------------------|--------------------------------------------------------|
| unique_id (\>0)       | ID único da ordem                                      |
| OK (0)                | Envio de ordem OK.                                     |
| ORDER_NOT_FOUND (-1)  | Ordem não encontrada                                   |
| RISK_INVALID_NET (-2) | Ordem não aceita pelo controle de risco                |
| INVALID_QUANTITY (-3) | Quantidade inválida (por exemplo: quantidade negativa) |
| INFLIGHT (-4)         | Ordem pendente                                         |
| UNKNOWN (-5)          |                                                        |
| OMS_DISCONNECTED (-6) | OMS desconectado                                       |
| INVALID_SYMBOL (-7)   | Símbolo inválido no contexto da estratégia             |
| INVALID_PRICE (-8)    | Preço inválido                                         |
| NOT_SUPPORTED (-9)    | Ordem não suportada                                    |

<br>

Vale ressaltar que, na modificação de ordens, mesmo que o programador
altere os campos da ordem, apenas seu preço e quantidade são passíveis
de modificações. Além disso, para modificar ou cancelar uma ordem,
deve-se usar de um objeto *OrderEntry* como argumento, o qual pode ser
recebido pelo *update_order(order)* quando encontrada via
*get_live_orders.*

#### Gerenciamento do WAIT

Ao utilizar as funções de envio, modificação ou cancelamento de ordens é
possível ou não receber o status WAIT a partir da corretora.

O neutrino opta por colocar a ordem em WAIT, WAIT_REPLACE ou WAIT_CANCEL
de maneira inmediata apenas as funções de envio, modificação ou
cancelamento sejam invocadas. Este fato acontece independentemente de
ter ou não recebido o status WAIT pela corretora.

Caso ocorra de receber o status WAIT da corretora, ele é ignorado e a
callback *order_update* é chamada notificando que existe uma ordem em
WAIT.

Caso ocorra de não receber o status WAIT da corretora, a callback
*order_update* só será chamada ao receber da corretora o estado da ordem
em questão (ACTIVE ou REJECTED). Nesse momento, o neutrino informa via
callback num primeiro momento que existe uma ordem em WAIT e em
sequência uma nova chamada à callback *order_update* informando o
estatdo da ordem (ACTIVE ou REJECTED).


## Callback de Ordens

As repostas ao envio, modificação e cancelamento de ordens são
notificadas ao usuário utilizando a callback *order_update. *De maneira
opcional, caso o usuário precise ser notificado somente das execuções é
possível utilizar a callback *order_filled.*

<table>
<thead>
<tr class="header">
<th><strong>Nome da callback</strong></th>
<th><strong>Descrição</strong></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>order_update(&lt;order&gt;)</td>
<td><p>Por esse <em>callback</em> o algoritmo será notificado à cada mudança de estado das das ordens válidas e essa notificações dependerão da máquina de estados do protocolo usado pela corretora. (Confira a seção Gerenciamento do WAIT). O argumento utilizado é:</p>
<ul>
<li><p><em>order</em>: objeto OrderEntry atualizado com informações recebidas pela corretora</p></li>
</ul></td>
</tr>
<tr class="even">
<td>order_filled(&lt;order&gt;, &lt;price&gt;, &lt;quantity&gt;)</td>
<td><p>Callback opcional, a qual será chamada antes do <em>order_update </em>caso definida no algoritmo, reportando somente as execuções. Após a chamada desta callback, a <em>order_update </em>será invocada.</p>
<ul>
<li><p><em>order</em>: objeto OrderEntry atualizado com informações recebidas pela corretora</p></li>
<li><p><em>price: </em>preço da execução</p></li>
<li><p><em>quantity:</em> quantidade executada</p></li>
</ul></td>
</tr>
</tbody>
</table>

<br>

## Consulta de Ordens

Internamente o neutrino armazena uma lista com as ordens indicando o
último estado na qual se encontram. A partir desta lista são
disponibilizados métodos de acesso para filtrar e obter informações do
conteudo desta lista.

Os métodos de consulta encontram-se no módulo *oms*:

<table>
<thead>
<tr class="header">
<th><strong>Função</strong></th>
<th><strong>Descrição</strong></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>get_orders(&lt;symbol&gt;="", &lt;side&gt;=NONE_SIDE, &lt;price&gt;=-1)</td>
<td><p>Recupera a lista de todas as ordens criadas no seu último estado. Utiliza como filtro:</p>
<ul>
<li><p><em>symbol: </em>nome do ativo</p></li>
<li><p><em>side:</em> OrderSide::BID ou OrderSide::ASK</p></li>
<li><p><em>price</em>: valor do preço</p></li>
</ul>
<p>Se são utilizados os valores default o filtro ignora esse campo e utiliza os restantes. Caso todos sejam default, a lista completa é devolvida.</p></td>
</tr>
<tr class="even">
<td>get_live_orders(&lt;symbol&gt;="", &lt;side&gt;=NONE_SIDE, &lt;price&gt;=-1)</td>
<td><p>Recupera a lista de todas as ordens criadas nas quais o seu estado seja: ACTIVE, REPLACED ou PARTIAL_FILLED.</p>
<p>Utiliza os mesmos filtros que a função <em>get_orders.</em></p></td>
</tr>
<tr class="odd">
<td>get_total_quantity(&lt;symbol&gt;, &lt;side&gt;, &lt;status_combination&gt;=0)</td>
<td><p>Recupera o acumulado das quantidades das ordens para o atibo <em>symbol</em> e lado <em>side</em>.</p>
<p>O bitmask <em>status_combination</em> especifica o filtro utilizado para o cálculo. O filtro representa uma combinação dos possíveis estados da ordem. No caso de solicitar ordens em ativas e executas, utilize <em>status_combination = (OrderStatus::ACTIVE | OrderStatus::FILLED)</em></p></td>
</tr>
<tr class="even">
<td>get_order_by_id(&lt;unique_id&gt;)</td>
<td><p>Recupera a ordem pelo <em>unique_id</em>.</p>
<p>Retorna None caso não exista uma ordem com o <em>unique_id </em>fornecido.</p></td>
</tr>
</tbody>
</table>

<br>

## Controle de Risco

O módulo de risco tem a função de não permitir que seja ultrapassado o
limite de posição configurado no neutrino. Após atingir esse limite
apenas as ordens que diminuam a posição serão aceitas além de permitir
a *virada de mão.*

A configuração do risco é feita por meio da mensagem de controle do
frontend ControlAlgo usando o comando *Set*. A mensagem *ControlAlgo* no
seu campo  *Tunables* deve conter o *max_position.* Este campo pode ser
composto por múltiplos objetos indexados pelo nome do ativo. Cada item,
deve ter dois campos obrigatórios *limit*, o qual indica o valor máximo
a ser atingido e o campo *type_limit*, indicando se o valor refere-se a
uma quantidade ou um valor financeiro.

Neste exemplo, dois ativos são configurados. O primeiro deles utiliza um
limite em quantidade e o segundo em financeiro:

```json
{
  "Tunables": {
    "FooStrategyName": {
      "max_position": {
        "DI1F21": {
          "limit": 200,
          "type_limit": "quantity"
        },
        "DI1F22": {
          "limit": 400,
          "type_limit": "financial"
        }
      }
    }
  }
}
```

Para acessar aos valores de risco dentro da estrategia utilize o
módulo *risk, *por meio da função *get:*

| **Função**      | **Descrição**                                                                 |
|-----------------|-------------------------------------------------------------------------------|
| `get(<symbol>)` | Recupera a estrutura *RiskData* com os valores de risco para o ativo *symbol* |

</p>

A estrutura *RiskData* é composta por: 

<table>
<thead>
<tr class="header">
<th><strong>Atributo</strong></th>
<th><strong>Descrição</strong></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>limit</td>
<td>Valor máximo a ser atingido após a conclusão da operação</td>
</tr>
<tr class="even">
<td>type_limit</td>
<td><p>Tipo risco, sendo possível:</p>
<ul>
<li><p>QUANTITY</p></li>
<li><p>FINANCIAL</p></li>
</ul></td>
</tr>
</tbody>
</table>

</p>

Baseado na configuração anterior é possível utilizar o módulo de risco
da maneira a seguir:

```python
win_risk = neutrino.risk.get('DI1F21')
print(win_risk.limit)
# >> 200
print(win_risk.type_limit)
# >> QUANTITY
```

As ordens que forem rejeitadas pelo risco tem o código de
erro `RISK_INVALID_NET`. Portanto ela não notificará o
algoritmo via *order_update()*.

A rejeição pelo risco é detectada ao comparar os valores de limite e a
soma das quantidades de ordens executadas (FILLED e PARTIAL_FILLED),
ordens ativas ou pendentes (ACTIVE, REPLACED, WAIT e WAIT_CANCEL) e da
nova ordem enviada.



##Controle de Posição

O controle de posição está contido no módulo *position.* Por meio dele é
possível acessar a posição de determinado ativo:

| **Função**      | **Descrição**                                                                   |
|-----------------|---------------------------------------------------------------------------------|
| `get(<symbol>)` | Recupera a posição para o ativo *symbol, *retornando um objeto *PositionStatus* |

</p>

Um objeto da classe *PositionStatus* contem os atributos a seguir:

| **Atributo** | **Descrição**                                                      |
|--------------|--------------------------------------------------------------------|
| total        | Objeto *PositionData* contendo informações sobre a posição total   |
| partial      | Objeto *PositionData* contendo informações sobre a posição parcial |
| initial      | Objeto *PositionData* contendo informações sobre a posição inicial |

</p>

Cada objeto *PositionData* contem:

| **Atributo** | **Descrição**         |
|--------------|-----------------------|
| net          | Quantidade da posição |
| net_price    | Preço médio           |
| bid_quantity | Quantidade da compra  |
| ask_quantity | Quantidade da venda   |
| bid_volume   | Volume da compra      |
| ask_volume   | Volume da venda       |

</p>

