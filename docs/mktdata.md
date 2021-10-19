# Market Data API

## Instrumento

O `InstrumentRegister` representa o objeto pelo qual são disponibilizados o book e o buffer trades.

### Assinatura

Para assinar um instrumento use a função add especifique o nome do ativo, o tamanho do buffer de trades e as callbacks de mercado:

|Função|Descrição|
|---|---|
|`add(<symbol>, book_callback=on_data, trade_callback=on_data, trade_buffer_size=64)`|Adiciona um instrumento, usando como argumentos:<ul><li>symbol: string com o nome do ativo a ser assinado, ex. 'WINQ19'</li><li>book_callback: callback a ser disparada na notificação de mudança de book</li><li>trade_callback: callback a ser disparada na notificação de trade</li><li>trade_buffer_size: quantidade máxima de trades contidos no buffer a cada notificação</ul>Um objeto do tipo InstrumentRegister é devolvido no caso de sucesso. Caso o símbolo não seja válido ou o tamanho do buffer ultrapasse 64 elementos, `None` é devolvido.|

</br>
Por exemplo:

```python
from neutrino import market

# Uso de duas callbacks customizadas
self.win = market(self).add(
    'WINQ19',
    book_callback=on_book,
    trade_callback=on_trade,
    trade_buffer_size=10)

# Cancelada callback de book e uso de callback default on_data para trades
self.wdo = market(self).add('WDOQ19', book_callback=None, trade_buffer_size=50)
```


A callback default é chamada *on_data* e precisa ser implementada pelo
usuário para ser notificado sobre as mudanças de book e trades.

As assinaturas estão baseadas em ativos podendo ser feitas em qualquer
momento da operação da estratégia. A assinatura de um ativo gera
automaticamente estruturas de dados preenchidas com informações do book
e os trades que acontecem nesse momento. Vale lembrar que os trades do
começo do dia não estão nesta lista.


### Acesso


Caso o instrumento não tenha sido salvo no momento da assinatura, é
possível utilizar a função *get* para recuperá-lo:

| **Função**      | **Descrição**                                                                                                                            |
|-----------------|-------------------------------------------------------------------------------------------------------------------|
| `get(<symbol>)` | Devolve um instrumento já cadastrado usando como argumento a string com o nome do símbolo. Caso o nome seja inválido *None* é devolvido. |

<br>
Por exemplo:

```python
self.instrument = market(self).get('WINQ19')
```


### Desassinatura


Para remover um instrumento utilize o objeto obtido pela criação do
mesmo:

```python
self.instrument = neutrino.market(self).get('WINDOM19')
if neutrino.market(self).remove(self.instrument):
    print('success')
```

A função *remove *é parte do módulo *market:*

| **Função**      | **Descrição**                                                                                                                            |
|-----------------|------------------------------------------------------------------------------------------------------------------------------------------|
| `remove(<instrument>)` |<p>Remove um instrumento. O <em>instrument</em> deve ser um objeto válido devolvido pela assinatura ou obtido pelo método <em>get. </em>A função retorna <em>True </em>em caso se sucesso, <em>False </em>caso contrário.</p><p>O objeto <em>instrument </em>passado como parâmetro é invalidado no sucesso da operação.</p>|

<br>
Na remoção de um instrumento os candles relacionado a ele não são
removidos de maneira automática, eles continuam sendo atualizados.



### Callbacks

Ao assinar determinada estrutura de análise é possível configurar as
futuras callbacks, podendo mudar a callback default on_data para outra
função, por exemplo:

```python
def initialize(self, symbols):
    market(self).add('WINQ19', book_callback=on_win_callback, trade_callback=None)
    market(self).add('WDOM19', book_callback=None, trade_callback=on_wdo_callback)

def on_win_callback(self, update):
    print('WINQ19 bar received')

def on_wdo_callback(self, update):
    print('WDOQ19 bar received')
```

Inclusive é possivel cancelar a callback para determinada assinatura. No
exemplo a seguir, o usuário cancela a callback do book, porém mantem as
notificações dos trades por meio da callback default on_data:

```python
def initialize(self, symbols):
    market(self).add('WINQ19', book_callback=None)

def on_data(self, update):
    print('WINQ19 trade')
```

### Atributos e métodos

Os atributos a seguir estão disponível para uma instancia do
*InstrumentRegister*:

-   name: nome do símbolo

-   book: book de ofertas

-   trades: buffer circular de trades

-   min_order_qty: lote mínimo

-   price_increment: tick mínimo

-   ready(): testa se o estado é diferente de OFFLINE


#### Book

-   bid: conjunto de ordens do lado da compra

-   ask: conjunto de ordens do lado da venda

-   state: enumerado com o estado do book, segundo a página 64
    [desta](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=2&ved=2ahUKEwit49OZkLDlAhXTCtQKHUlTDRkQFjABegQIAhAC&url=http%3A%2F%2Fwww.b3.com.br%2Fdata%2Ffiles%2F76%2F42%2F6A%2F9B%2F4DA15610BE423F46AC094EA8%2FUMDF_MarketDataSpecification_v2.1.5.pdf&usg=AOvVaw0-Vp2bWvXwst0nsz-7nW6O)
    documentação.

-   sequence: sequencial do book. Id único da atualização mais recente
    do book de ofertas

-   name: nome do símbolo

Cada item do bid e ask tem os seguintes atributos:

-   price: preço da ordem

-   quantity: quantidade da ordem

-   detail: corretora

-   order_id: SecondaryOrderID, definido pela Bolsa

-   virtual_md_id: ID atribuído à ordem pelo neutrino

Para imprimir o topo do book é possível:

```python
print(instrument.bid[0].price)
print(instrument.ask[0].price)
```

#### Trades

O acesso é feito por meio de indices. O valor máximo é determinado pelo
parâmetro trade_buffer_size. Os atributos de cada item são:

-   trade_id: Id do trade na B3

-   datetime: horário local quando o negócio foi executado

-   price: preço do negócio

-   quantity: quantidade do negócio

-   buyer: contraparte compradora

-   seller: contraparte vendedora

-   status: é o agressor indicator enviado pela bolsa. Pode ser R(trade
    de RLP), X(direto), -(trade iniciado pelo vendedor) e +(iniciado
    pelo comprador)


## Candles

O CandleRegister representa a estrutura que contem os candles
para determinado ativo e as propriedades para sua construção como intervalo, tick, quantidade inicial de barras, entre outros.
Estão disponíveis os candles do tipo stick e renko. A sua assinatura pode ser feita da maneira a seguir:


### Assinatura

A assinatura de um ativo pode ser feita por meio da funçao *add_bar:*

<table>
<thead>
<tr class="header">
<th><strong>Função</strong></th>
<th><strong>Descrição</strong></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><code>add_bar(&lt;symbol&gt;,bar_count=100, interval=1)</code></td>
<td><p>Adiciona um novo candle do tipo stick, utilizando como argumentos:</p>
<ul>
<li><p>symbol: string com o nome do ativo a ser assinado, ex. 'WINQ19'</p></li>
<li><p>bar_count: número de candles, default é 100.</p></li>
<li><p>interval: intervalo para geração dos candles, default é 1 minuto. </p></li>
</ul>
<p>Devolve um objeto do tipo <em>CandleRegister.</em></p></td>
</tr>
<tr class="even">
<td><code>add_interday_bar(&lt;symbol&gt;,bar_count=100, interval='D')</code></td>
<td><p>Adiciona um novo candle do tipo <em>interday</em>, utilizando como argumentos:</p>
<ul>
<li><p>symbol: string com o nome do ativo a ser assinado, ex. 'WINQ19'</p></li>
<li><p>bar_count: número de candles, default é 100</p></li>
<li><p>interval: intervalos possíveis são dia 'D' (default), semana 'W' e mês 'M'</p></li>
</ul>
<p>Devolve um objeto do tipo <em>CandleRegister.</em></p></td>
</tr>
<tr class="odd">
<td><code>add_renko(&lt;symbol&gt;,tick_count=2)</code></td>
<td><p>Adiciona um novo candle do tipo renko, utilizando como argumentos:</p>
<ul>
<li><p>symbol: string com o nome do ativo a ser assinado, ex. 'WINQ19'</p></li>
<li><p>tick_count: níveis de preço para cada tijolo, default é 2</p></li>
<p>A quantidade de dias considerados para o histórico é 15.</p>
</ul>
<p>Devolve um objeto do tipo <em>CandleRegister.</em></p></td>
</tr>
</tbody>
</table>

<br>
Por exemplo:

```python
from neutrino import market

bar = market(self).add_bar(symbol, bar_count=100, interval=1)
interday_bar = market(self).add_interday_bar(symbol, bar_count=10, interval='M')
renko = market(self).add_bar(symbol, tick_count=5)
```

A assinatura de um instrumento e de barras são **independentes**, sendo
que o usuário pode solicitar os candles de um determinado ativo usando a
função `add_bar`, `add_interday_bar` ou `add_renko` sem ter adicionado o instrumento por meio do `add`.
As atualizações de candle chegarão através da callback especificada para
o `trade_callback` daquele instrumento ou no `on_data`, se nenhuma callback
tiver sido especificado.

### Acesso

O armazenamento do candle é responsabilidade do usuário, porém caso o
objeto não tenha sido salvo, é possível recuperá-lo, utilizando a
função `get_bar`, `get_interday_bar` ou `get_renko`:

<table>
<thead>
<tr class="header">
<th><strong>Função</strong></th>
<th><strong>Descrição</strong></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><code>get_bar(&lt;symbol&gt;,bar_count=100, interval=1)</code></p>
<p><code>get_interday_bar(&lt;symbol&gt;,bar_count=100, interval='D')</code></p>
<p><code>get_renko(&lt;symbol&gt;,bar_count=2)</code></p></td>
<td><p>Recupera um candle previamente cadastrado usando como argumentos:</p>
<ul>
<li><p>symbol: string com o nome do ativo a ser assinado.</p></li>
<li><p>bar_count: número de candles, default é 100.</p></li>
<li><p>tick_count: níveis de preço para cada tijolo, default é 2.</p></li>
<li><p>interval: intervalo para geração dos candles, default é 1 minuto ou 'D'.</p></li>
</ul>
<p>Caso não exista um candle usando a combinação dos argumentos anteriores <em>None</em> é devolvido<em>, </em>caso contrário o objeto do tipo <em>CandleRegister</em></p></td>
</tr>
</tbody>
</table>

<br>
Por exemplo:

```python
self.bar_win = market(self).get_bar('WINQ19')
self.bar_win_count_1000 = market(self).get_bar('WINQ19', bar_count=1000)
self.bar_win_interval_5 = market(self).get_bar('WINQ19', interval=5)
self.bar_win_interval_day = market(self).get_interday_bar('WINQ19', interval='D')
self.renko_win_10 = market(self).get_renko('WINQ19', tick_count=10)
```

### Atributos e métodos

-   open

-   high

-   low

-   close

-   timestamps

-   quantity

-   quantity_buy

-   quantity_sell

-   volume

-   quantity_accumulated

-   quantity_buy_accumulated

-   quantity_sell_accumulated

-   num_trades

-   last_id

-   ready()

-   properties: objeto do tipo CandleProperties, que por sua vez possui:

    -   tick_count: níveis de preço para cada tijolo

    -   bar_count: número de barras solicitadas

    -   interval: periodo do candle em minutos ou especificação 'D', 'W', 'M' para interday

    -   symbol: nome do ativo

### Desassinatura

Para remover um candle utilize o objeto `CandleRegister` devolvido na
criação do mesmo usando a função `remove_bar`, `remove_interday_bar` ou `remove_renko`:

<table>
<thead>
<tr class="header">
<th><strong>Função</strong></th>
<th><strong>Descrição</strong></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><code>remove_bar(&lt;bar&gt;)</code></p>
<p><code>remove_interday_bar(&lt;bar&gt;)</code></p>
<p><code>remove_renko(&lt;bar&gt;)</code></p></td>
<td><p>Remove um candle. O <em>bar</em> deve ser um objeto válido devolvido pela assinatura de um candle ou obtido pelo método <em>get_bar. </em>A função retorna <em>True </em>em caso se sucesso, <em>False </em>caso contrário.</p>
<p>O objeto <em>bar </em>passado como parâmetro é invalidado no sucesso da operação.</p></td>
</tr>
</tbody>
</table>

<br>
Por exemplo:

```python
from utils import market

bar = market(self).add_bar('PETR4')
if market(self).remove_bar(bar):
    print('success')
```

Ao remover a assinatura de um candle todos os indicadores atrelados a
ele também são removidos.


## Indicadores

Os indicadores estão naturalmente vinculados a um candle. Sendo que, a
sua assinatura deve usar um objeto candle:

```python
self.bar_winq19 = neutrino.market(self).add_bar("WINQ19")

if self.bar_winq19.ready():

self.sma_winq19 = self.bar_winq19.add_sma(bar_count=10,
source=neutrino.IndicatorSource.OPEN)

print(self.bar_winq19.ready(), self.sma_winq19.ready())
# >> True, True
```

Caso usuário não reserve uma variável para salvar o indicador construído
é fornecido o seguinte mecanismo de acesso:

```python
for indicator in self.bar_winq19.get_indicators():  # retorna iterador
    print("Indicator " + indicator.name + " value: " + str(indicator.values[-1]))
```

### Indicadores disponíveis

O framework fornece os indicadores a seguir:

<table>
<thead>
<tr class="header">
<th><strong>Nome</strong></th>
<th><strong>Assinatura</strong></th>
<th><strong>Parâmetros</strong></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>SMA</td>
<td><code>add_sma(bar_count=.., source=...)</code></td>
<td><ul>
<li><p>bar_count: quantidade de barras</p></li>
<li><p>source: tipo de entrada (conferir seçao Source)</p></li>
</ul></td>
</tr>
<tr class="even">
<td>EMA</td>
<td><code>add_ema(bar_count=.., source=...)</code></td>
<td><ul>
<li><p>bar_count: quantidade de barras</p></li>
<li><p>source: tipo de entrada</p></li>
</ul></td>
</tr>
<tr class="odd">
<td>MOM</td>
<td><code>add_mom(bar_count=.., source=...)</code></td>
<td><ul>
<li><p>bar_count: quantidade de barras</p></li>
<li><p>source: tipo de entrada</p></li>
</ul></td>
</tr>
<tr class="even">
<td>SAMOM</td>
<td><code>add_samom(bar_count=.., source=..., sa_bar_count=...)</code></td>
<td><ul>
<li><p>bar_count: quantidade de barras</p></li>
<li><p>source: tipo de entrada</p></li>
<li><p>sa_bar_count: quantidade de barras utilizadas na média</p></li>
</ul></td>
</tr>
<tr class="odd">
<td>TRANGE*</td>
<td><code>add_trange()</code></td>
<td></td>
</tr>
<tr class="even">
<td>SATR*</td>
<td><code>add_satr(sa_bar_count=...)</code></td>
<td><ul>
<li><p>sa_bar_count: quantidade de barras utilizadas na média</p></li>
</ul></td>
</tr>
<tr class="odd">
<td>ATR*</td>
<td><code>add_atr(bar_count=...)</code></td>
<td><ul>
<li><p>bar_count: quantidade de barras</p></li>
</ul></td>
</tr>
<tr class="even">
<td>ADX*</td>
<td><code>add_atr(bar_count=...)</code></td>
<td><ul>
<li><p>bar_count: quantidade de barras</p></li>
</ul></td>
</tr>
<tr class="odd">
<td>SAADX*</td>
<td><code>add_atr(bar_count=..., sa_bar_count=...)</code></td>
<td><ul>
<li><p>bar_count: quantidade de barras</p></li>
<li><p>sa_bar_count: quantidade de barras utilizadas na média</p></li>
</ul></td>
</tr>
<tr class="even">
<td>PLUS_DI*</td>
<td><code>add_plus_di(bar_count=...)</code></td>
<td><ul>
<li><p>bar_count: quantidade de barras</p></li>
</ul></td>
</tr>
<tr class="odd">
<td>MINUS_DI*</td>
<td><code>add_minus_di(bar_count=...)</code></td>
<td><ul>
<li><p>bar_count: quantidade de barras</p></li>
</ul></td>
</tr>
<tr class="even">
<td>BBANDS**</td>
<td><code>add_bbands(bar_count=..., deviation_up=..., deviation_down=..., average=...)</code></td>
<td><ul>
<li><p>bar_count: quantidade de barras</p></li>
<li><p>deviation_up:</p></li>
<li><p>deviation_down:</p></li>
<li><p>average: tipo de média utilizada (conferir seção Average)</p></li>
</ul></td>
</tr>
<tr class="odd">
<td>SABBANDS**</td>
<td><code>add_sabbands(bar_count=..., deviation_up=..., deviation_down=..., average=..., sa_bar_count=...)</code></td>
<td><ul>
<li><p>bar_count: quantidade de barras</p></li>
<li><p>deviation_up:</p></li>
<li><p>deviation_down:</p></li>
<li><p>average: tipo de média utilizada (conferir seção Average)</p></li>
<li><p>sa_bar_count: quantidade de barras utilizadas na média</p></li>
</ul></td>
</tr>
<tr class="even">
<td>STDDEV</td>
<td><code>add_stddev(bar_count=.., source=..., deviation_count=...)</code></td>
<td><ul>
<li><p>bar_count: quantidade de barras</p></li>
<li><p>source: tipo de entrada</p></li>
<li><p>deviation_count:</p></li>
</ul></td>
</tr>
<tr class="odd">
<td>RSI</td>
<td><code>add_rsi(bar_count=..., source=...)</code></td>
<td><ul>
<li><p>bar_count: quantidade de barras</p></li>
<li><p>source: tipo de entrada</p></li>
</ul></td>
</tr>
<tr class="odd">
<td>SAR</td>
<td><code>add_sar(acceleration=..., maximum=...)</code></td>
<td><ul>
<li><p>acceleration: </p></li>
<li><p>maximum: </p></li>
</ul></td>
</tr>
<tr class="odd">
<td>MACD</td>
<td><code>add_macd(fast_ma_type=..., fast_ma_period=..., slow_ma_type=..., slow_ma_period=..., signal_ma_type=..., signal_ma_period=...)</code></td>
<td><ul>
<li><p>fast_ma_type: tipo de média utilizada (conferir seção Average) para 'fast'</p></li>
<li><p>fast_ma_period: periodo para a média 'fast'</p></li>
<li><p>slow_ma_type: tipo de média utilizada (conferir seção Average) para 'slow'</p></li>
<li><p>slow_ma_period: periodo para a média 'slow'</p></li>
<li><p>signal_ma_type: tipo de média utilizada (conferir seção Average) para 'signal'</p></li>
<li><p>signal_ma_period: periodo para a média 'signal'</p></li>
</ul></td>
</tr>
<tr class="odd">
<td>STOCH</td>
<td><code>add_stoch(fast_k_ma_period=..., slow_k_ma_type=..., slow_k_ma_period=..., slow_d_ma_type=..., slow_d_ma_period=...)</code></td>
<td><ul>
<li><p>fast_ma_type: tipo de média utilizada (conferir seção Average) para 'fast'</p></li>
<li><p>fast_ma_period: periodo para a média 'fast'</p></li>
<li><p>slow_k_ma_type: tipo de média utilizada (conferir seção Average) para 'slow_k'</p></li>
<li><p>slow_k_ma_period: periodo para a média 'slow_k'</p></li>
<li><p>slow_d_ma_type: tipo de média utilizada (conferir seção Average) para 'slow_d'</p></li>
<li><p>slow_d_ma_period: periodo para a média 'slow_d'</p></li>
</ul></td>
</tr>
<tr class="odd">
<td>STOCHF</td>
<td><code>add_stochf(fast_k_ma_period=..., fast_d_ma_type=..., fast_d_ma_period=...)</code></td>
<td><ul>
<li><p>fast_k_ma_type: tipo de média utilizada (conferir seção Average) para 'fast_k'</p></li>
<li><p>fast_d_ma_type: tipo de média utilizada (conferir seção Average) para 'fast_d'</p></li>
<li><p>fast_d_ma_period: periodo para a média 'fast_d'</p></li>
</ul></td>
</tr>
<tr class="odd">
<td>OBV</td>
<td><code>add_obv(source=...)</code></td>
<td><ul>
<li><p>source: tipo de entrada</p></li>
</ul></td>
</tr>

</tbody>
</table>

*\* source utilizado: high, low, close*

*\*\* source utilizado: close*


### Acceso aos valores

Os valores dos indicadores pode ser recuperado acessando o vetor de
valores *values:*

```python
# Acessando o valor do indicador mais recente
candle =  market(self).get_bar('WINQ19', interval=5)
self.sma = candle.add_sma(10)
print(self.sma.values[-1])
```

No caso do BBANDS e SABBANDS os valores devem ser acessados pelo
vetor *bands*:

```python
# Acessando o valor do indicador mais recente
candle =  market(self).get_bar('WINQ19', interval=5)
self.sabbands = candle.add_bbands(bar_count=10, deviation_up=5,
deviation_down=5, average=IndicatorAverage.SMA, sa_bar_count=5)

print(self.sabbands.bands[0][-1])
print(self.sabbands.bands[1][-1]
print(self.sabbands.bands[2][-1]
```

### Atributos

Um indicador ao ser assinado ou ser recuperado usando a função
get_indicators, possui a seguinte lista de atributos:

-   values: lista com os valores do indicador

-   bands: lista de 3 vetores com os valores para o BBANDS e SABBANDS

-   last_id: id do valor de indicador atualizado mais recentemente

-   name: nome do indicador (confira a coluna Nome na lista de
    Indicadores disponíveis)

-   properties: itens da coluna Parâmetros na lista de Indicadores
    disponíveis

    -   source

    -   bar_count

    -   sa_bar_count

    -   deviation_count

    -   deviation_up

    -   deviation_down

    -   average

Todos os indicadores tem a lista anterior de atributos disponíveis
independente do tipo de indicador.

#### Source

O argumento source para os indicadores é determinado pelo enumerado
neutrino.IndicatorSource, podendo ter os valores:

-   NONE

-   OPEN

-   HIGH

-   LOW

-   CLOSE

-   QUANTITY

-   QUANTITY_SELL

-   QUANTITY_BUY

-   VOLUME

-   QUANTITY_ACCUMULATED

-   QUANTITY_SELL_ACCUMULATED

-   QUANTITY_BUY_ACCUMULATED

#### Average

O argumento average para os indicadores BBANDS e SABBANDS é determinado
pelo enumador neutrino.IndicatorAverage, podendo ter os valores: 

-   SMA

-   EMA

-   WMA

### Desassinatura

Para remover um indicador previamente criado é necessário ter acesso ao
objeto a ser removido. Use a função *remove_indicator* que é um método
do *CandleRegister.*

<table>
<thead>
<tr class="header">
<th><strong>Função</strong></th>
<th><strong>Descrição</strong></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><code>remove_indicator(&lt;indicator&gt;)</code></td>
<td><p>Remove um indicador contido num candle. O <em>indicator</em> deve ser um objeto válido devolvido pela assinatura de um indicador ou obtido pelo método <em>get_indicators. </em>A função retorna <em>True </em>em caso se sucesso, <em>False </em>caso contrário.</p>
<p>O objeto <em>indicator </em>passado como parâmetro é invalidado no sucesso da operação.</p></td>
</tr>
</tbody>
</table>

<br>
No exemplo a seguir o objeto do indicador SMA é salvo e utilizado para
sua remoção:

```python
from utils import market

bar = market(self).add_bar('PETR4')
sma = bar.add_sma(10)
if bar.remove_indicator(sma):
    print('success')
```


## Callbacks

### Callbacks inicio e fim

A primeira callback da estrategia é initialize. Esta função é chamada
pelo neutrino após contrastar os ativos listados no arquivo de
configuração da estrategia com o security list recebido pelo marketdata.
De modo que, toda estrategia deve iniciar o seu processamento com
instruções dentro da callback initialize. O parâmetro symbols indica a
lista de ativos dispníveis para uso. Este ativos correspondem ao mesmos
especificados no arquivo de configuração da estrategia:

```json
# quantick.conf:

"Symbols" : [ "DI1F21" , "DI1F19" , "DI1F23"],
...
```

Considerando o arquivo anterior:

```python
def initialize(self, symbols)
    print([s for s in symbols])
```
```python
# >> ['DI1F21', 'DI1F19', 'DI1F23']
```

A API também permite implementar a callback <em>finalize</em> (não obrigatória)
que é chamada quando a estratégia é terminada por qualquer motivo:

```python
def initialize(self, symbols):
    neutrino.utils(self).quit()

def finalize(self, reason):
    print("finalize:" + str(reason))
```

#### QuitReason

O neutrino sinaliza que a estrategia vai encerrar chamando a callback
finalize. O parâmetro reason indica o motivo do encerramento. Os valores
para o QuitReason são:

-   USER_QUIT: o usuário usou a chamada neutrino.utils(self).quit() ou usando
    CTRL+C

-   ALGOMAN_QUIT: o Algoman enviou o comando Abort

-   NO_MD_CONNECTION: perda de conexão com a fonte de dados de mercado
    (Relay)

-   NO_FRONTEND_CONNECTION: perda de conexão com o frontend

-   BAD_FD: falha na comunicação do neutrino com OMS ou Algoman ou Relay

-   INVALID_PROTOCOL: pacote de dados recebido é inválido

-   OUT_OF_SYNC: a estrategia não consegue sincronizar após 10
    tentativas

-   NO_OMS_CONNECTION: sem conexão com o OMS

-   NO_ALGOMAN_CONNECTION: sem conexão com o Algoman

### Callback default, book, trade e candles

O controle do volume de notificações recebidas é feito em parte pelo
próprio usuário. Basicamente a função on_data será disparada pelo
Neutrino pela mudanca de book ou trades:

-   No caso do book duas opções de atualização podem ser notificadas:
    bid e ask.

-   No caso de trades, entende-se que os candles e indicadores também
    tem sido atualizados

-   Quando acontecer a virada do periodo sem ter acontecido algum
    negocio o Neutrino notifica atualização dos candles/indicadores

A callback default on_data tem como parâmetro a estrutura update, tendo
como membros os campos a seguir:

-   symbol: nome do ativo atualizado e motivo da chamada da callback

-   reason: vetor com a lista de acontecimentos motivo da chamada da
    callback

-   bid_count: número de atualizações por parte do bid

-   ask_count: número de atualizações por parte do ask

-   trade_count: número de trades

-   status_changed:

O campo reason é um vetor ordenado levando em consideração como maior
prioridade o acontecimento de um trade e a seguir o lado do book com
maior quantidade de atualizações. Por exemplo se existiram negocios e
além disso bid_count \> ask_count, então:

```python
def on_data(self, update)
    print(update.symbol + ' - ' + ','.join(str(r) for r in update.reason))
```
```python
# >> WINQ19 - TRADES, BID_SIDE, ASK_SIDE
```

No exemplo a seguir o usuário assina book, trades, candle e indicadores
para dois ativos diferentes. As mudanças são recebidas pelo on_data e o
próprio usuário é responsável por determinar o tratamento de cada uma
das opções possíveis:

```python
from neutrino import *

def initalize(self)
    market(self).add_book("WINQ19")
    market(self).add_book("PETR4")
    self.winq_book = market(self).get_book("WINQ19")
    self.petr_book = market(self).get_book("PETR4")
    self.winq_candle = market(self).add_bar("WINQ19", interval=1)
    self.petr_candle = market(self).add_bar("PETR4", interval=5)
    self.winq_candle.add_sma(bar_count=5)
    self.petr_candle.add_sma(bar_count=10)
    self.petr_candle.add_adx()

def on_data(self, update):
    if "PETR4" in update.symbol:
        self.on_petr_data(update)
    if "WINQ19" in update.symbol:
        self.on_winq_data(update)

def on_petr_data(self, update):
    if UpdateReason.BID_SIDE in update.reason:
        self.on_petr_bid(update)
    if UpdateReason.ASK_SIDE in update.reason:
        self.on_petr_ask(update)
    if UpdateReason.TRADE in update.reason or \
        UpdateReason.NEW_BAR in update.reason:
        self.on_petr_candle(update)

def on_winq_data(self, update):
    if UpdateReason.BID_SIDE in update.reason:
        self.on_winq_bid()
    if UpdateReason.ASK_SIDE in update.reason:
        self.on_winq_ask()
    if UpdateReason.TRADES in update.reason or \
        UpdateReason.NEW_BAR in update.reason:
        self.on_winq_candle()
```

O framework permite também excluir a callback default de modo que as
estruturas assinadas são atualizadas, porém não existe notificação deste
acontecimento. Por exemplo, podem ser assinados book e trades porém o
book será só consultado quando um trade acontecer. Para ter este efeito
é possível anular a callback do book:

```python
def initalize(self)
    market(self).add("PETR4", book_callback=None)
    self.petr_book = market(self).get("PETR4").book
    self.petr_trades = market(self).get("PETR4").trades

def on_data(self, update):
    # Imprime o último trade e o topo do book a cada novo negócio
    print(self.petr_trades[-1])
    print(self.petr_book.bid[0])
    print(self.petr_book.ask[0])
```

#### Candle Vazio

Quando um candle de um determinado intervalo for inicializado pela
passagem do tempo, antes de acontecer algum negócio naquele período, a
callback de candles é chamada usando o UpdateReason.NEW_BAR, contendo um
candle com os valores OLHC do candle anterior. Neste caso a callback de
trade não é ativada.

#### Direto

No caso de sair um 'direto' a callback de book não é ativada somente a
callback de trade e candle.


## Utilitários

Funções não relacionadas diretamente à assinaturas de candle/instrumento
ou controle de ordens e posição ficam dentro do módulo *utils*. Neste
módulo existem as funções a seguir:

<table>
<thead>
<tr class="header">
<th><strong>Função</strong></th>
<th><strong>Descrição</strong></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><code>notify(&lt;text&gt;)</code></td>
<td><p>Envia uma mensagem de no máximo 200 caracteres para o AlgoMan o qual deverá notificar o frontend. A função retorna False se a string de entrada ultrapassa a quantidade de caracteres permitida ou o envio é falho, retorna True no caso sucedido.</p>
<p>text: string com até 200 caracteres. </p></td>
</tr>
<tr class="even">
<td><code>now()</code></td>
<td>Devolve o UNIX timestamp</td>
</tr>
<tr class="odd">
<td><code>quit()</code></td>
<td>Finaliza a estrategia chamando a callback de <em>finalize</em> com o valor <em>reason=USER_QUIT</em></td>
</tr>
<tr class="even">
<td><code>turn_off(&lt;text&gt;)</code></td>
<td><p>Envia mensagem para usuário, retira estratégia do scheduller do site e termina estratégia.</p></td>
</tr>
<tr class="odd">
<td><code>by_price(side=&lt;book_side&gt;, depth=&lt;max_rows&gt;)</code></td>
<td>Agrupa o book por preço de usando como entrada o <em>lado</em> (ask ou bid) do book passado como argumento <em>(book_side).</em> Se <em>depth é </em>0 o book inteiro é agrupado, caso contrário o book é agrupado até gerar no máximo <em>max_rows</em> como saída.</td>
</tr>
</tbody>
</table>

<br>

## Scheduler

O usuário tem a possibilidade de cadastrar uma função para ser executada
de acordo com um horário específico ou a cada certo intervalo. Estas
funções encontram-se dentro do módulo *utils*.

| **Função**                                  | **Descrição**                                                                                                                                                                                                                                                    |
|---------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `at(function=..., hour=.., minute=...)`       | Agenda a função *function *para ser executada a determinada hora, com precisão de hora e minute. No caso de parâmetros inválidos de hora (\[0-23\]) e minuto (\[0-59\]) a função retorna None, caso contrário um objeto do tipo *ScheduledFunction* é devolvido. |
| `every(function=...,interval=\<seconds.ms\>)` | Agenda a função *function *para ser executada com determinado intervalo. Intervalo mínimo de 0.250 s. No caso de parâmetros inválidos de intervalo (\< 0.250 s) a função retorna None, caso contrário um objeto do tipo *ScheduledFunction* é devolvido.         |
| `get_functions()`                             | Retorna uma lista com todas as funções agendadas                                                                                                                                                                                                                 |
| `remove_function(function)`                   | Remove a função *function. *Caso *function* não exista a função retorna *False,* *True* caso contrário.                                                                                                                                                          |

<br>
No exemplo a seguir, a funçao *opening *será executada as 10:00h e a
função *check* será executada a cada 0.5s:

```python
from neutrino import utils
opening_event = utils(self).at(self.opening, 10, 00)
check_event = utils(self).every(self.check, 0.5)

```

Também é possível recuperar e remover as funções agendadas, assim como
os outros callbacks registrados pelo usuário. Neste exemplo, o usuário
remove os eventos agendados um a um:

```python
from neutrino import utils

function = utils(self).get_functions()
for function in functions:
    utils(self).remove_function(function)

```
Lembre-se que:

-   ao adicionar um evento no scheduler a callback default *on_data*
    assim como todas as callbacks cadastradas pelo usuário continuam
    sendo executadas normalmente;

-   as funções agendadas pelo utils(self).at são excluídas logo após serem
    executadas.

-   no caso de tentar inserir um agendamento repetido, isto é o mesmo
    para horario/intervalo - função, o objeto ja existente é devolvido.

-   o agendamento não suporta funções sobrecarregadas, pois o nome da
    função é usado para indexar os agendamentos internamente. 

### Atributos

Ao agendar uma função um objeto do tipo *ScheduledFunction* é devolvido.
Ele possui os seguintes atributos:

-   function: objeto python apontando para a função que será executada

-   hour: hora do agendamento no formato 24h

-   minute: minuto do agendamento

-   interval: intervalo em segundos com precisao de 3 casas decimais.

Caso o agendamento seja feito usando *at, *o campo *interval *é igual a
zero. No caso de ter usado a *every* os campos *hour* e *minute* são
iguais a zero.




## SummaryLine

O `SummaryLine` é uma estrutura que contém dados operacionais sobre determinado ativo. Por meio dele é possível consultar o topo do book, último trade, estatísticas, entre outros. Este conjunto de informações visa reduzir o tráfego para o neutrino evitando, por exemplo, assinar o book de certo ativo para conhecer o último preço de negociação. O SummaryLine é enviado ao neutrino a cada 200 ms. A tabela a seguir detalha a os campos desta estrutura:

|Campo|Descrição|
|---|---|
|`symbol`|Nome do símbolo assinado|
|`bid`|Estrutura `BookEntry` que indica o topo do lado da compra|
|`ask`|Estrutura `BookEntry` que indica o topo do lado da venda|
|`last_trade`|Estrutura `TradeEntry` com informações do último negocio executado|
|`stats`|Estruturas `SummaryLineStats` com informações estatísticas:<ul><li>trade_volume</li><li>high</li><li>low</li><li>vwap</li><li>opening</li><li>closing</li><li>theo</li><li>settlement</li><li>imbalance</li><li>last</li></ul>|
|`tunnels`|Estrutura `SummaryLineTunnels` com informações sobre tunnels:<ul><li>hard_limit</li><li>auction_limit</li><li>rejection_band</li><li>static_limit</li></ul>|
|`status`|Estrutura `StatusEntry` com informações sobre o estado do book|

</br>

|Campo de BookEntry|Descrição|
|---|---|
|price||
|quantity||
|detail||
|order_id||

</br>

|Campo de TradeEntry|Descrição|
|---|---|
|price||
|quantity||
|buyer||
|seller||
|datetime||
|status|'+' compra, '-' venda, 'x' cross|
|trade_id||

</br>

|Campo de StatisticsEntry|Descrição|
|---|---|
|price||
|quantity||
|longnum||

</br>

|Campo de TunnelEntry|Descrição|
|---|---|
|low_price||
|high_price||

</br>

|Campo de StatusEntry|Descrição|
|---|---|
|status|17: open|
|open_trade_time||

</br>


### Assinatura

Para assinar um `SummaryLine` use a função add_summary e especifique o nome do ativo e callback opcionalmente. A assinatura de um `SummaryLine` não está vinculada à assinatura de um `InstrumentRegister`, de modo que é possível assinar o SummaryLine de múltiplos ativos sem sequer ter assinado o book de algum deles.

|Função|Descrição|
|---|---|
|`add_summary(<symbol>, summary_callback=on_data)`|Adiciona um SummaryLine, usando como argumentos:<ul><li>symbol: string com o nome do ativo a ser assinado, ex. 'WINQ19'</li><li>summary_callback: callback a ser disparada na notificação de recepção de SummaryLine</li>ntidos no buffer a cada notificação</ul>Um objeto do tipo SummaryLine é devolvido no caso de sucesso. Caso o símbolo não seja válido, `None` é devolvido.|

O argumento `summary_callback` por padrão é a callback `on_data` porém pode ser customizada. Em ambos casos, a callback recebe como argumento um objeto do tipo `Update`. Se a callback para o `SummaryLine` é chamada, o vetor `reason` deste objeto contém um elemento com valor `BookUpdateReason.SUMMARY_LINE`.

</br>
Por exemplo:

```python
from neutrino import market

# Uso de callback customizadas
self.win_summary = market(self).add_summary(
    'WINQ19',
    summary_callback=on_summary)
```

### Acesso

Para recuperar uma objeto `SummaryLine` use a funcao `get_summary`:

|Função|Descrição|
|---|---|
|`get_summary(<symbol>)`|Use como argumento a string com o nome do ativo a ser assinado, ex. 'WINQ19'. Uma objeto do tipo `SummaryLine` é devolvido ou `None` caso o símbolo seja inválido ou não possua assinatura|


</br>
Por exemplo:

```python
from neutrino import market

self.win_summary = market(self).get_summary('WINQ19')
print(self.win_summary.stats.high.price)
```

### Desassinatura

Para desassinar o `SummaryLine` use a funcao `remove_summary`:

|Função|Descrição|
|---|---|
|`remove_summary(summary)`|Use como argumento um objeto `SummaryLine` existente obtido pelo retorno das funções `add_summary` ou `get_summary`. Se objeto fornecido como parâmetro é inválido a função retorna `False`, `True` caso sucesso|


</br>
Por exemplo:

```python
from neutrino import market

self.win_summary = market(self).add_summary('WINQ19')
success = market(self).remove_summary(self.win_summary)
```
