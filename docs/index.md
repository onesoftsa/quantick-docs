# Neutrino API

O neutrino é uma API em Python para implementar estratégias de algotrading no
 mercado brasileiro. Ela dá acesso à uma engine desenvolvida em C++ que cuida de
 todas as interações com a B3 e está divida nos seguintes conjuntos:

- Estruturas de análise: book, trades (ticks), candles e indicadores
- Funções de configuração
- Callbacks

O acesso à estes conjuntos é feito pelos objetos `market`, `utils`, `oms` e
 `position`. A API também realiza alguns callbacks para seu código em Python
 quando certos eventos ocorrem. Portanto, você pode implementar os métodos
 abaixo:

- `initialize` (obrigatório)
- `on_data` (opcional)
- `order_update` (obrigatório se estratégia enviar ordens)
- `order_filled` (opcional)
- `set_parameters` (obrigatório se rodar pelo Quantick)
- `get_parameters` (obrigatório se rodar pelo Quantick)
- `finalize` (opcional)


## Preparando o ambiente local

Para preparar seu ambiente para testar o neutrino localmente você precisa
 clonar [este](https://github.com/onesoftsa/neutrino-devel) repositório,
 instalar o Docker e o Make na sua máquina (se estiver usando windows, pode
 querer fazer isso usando o [chocolatery](https://chocolatey.org/packages/make)).
 Depois, preencha o arquivo `scripts/credentials.yml` seguindo o arquivo
 exemplo disponível na mesma pasta. Finalmente, inicie o Docker e use os
 comandos abaixo. Isso iniciará uma estratégia demo para testar o ambiente.
 Pressione `Ctrl+A D` para terminar.

```shell
$ make docker-build
$ make example
```

Para instalar diferetes módulos do Python para serem utilizadas dentro de suas
 estratégias, inicie a imagem base do docker e instale os pacotes desejados
 utilizando o `pip`.

```shell
$ make base-env
$ pip3.7 install [PACKAGE NAME]
```

Antes de terminar o container atual, inicie outro terminal e siga [estas](https://phoenixnap.com/kb/how-to-commit-changes-to-docker-image)
 instruções para comitar as mudanças que você fez  como uma nova imagem. Assim
 você pode voltar a utilizá-la mais tarde, sem precisar reinstalar os módulos
 novamente.

```shell
$ docker ps -a
$ docker commit [CONTAINER_ID] neutrino-1
$ doker image
```
Finalmente, termine o ambiente que utilizou para fazer as instalações utilizando
 `Ctrl+A D` e inicie um novo container para testar suas estratégias.

```shell
$ make neutrino-env
```

## Arquivos de configuração

### Market Data

O arquivo de configuração determina os parâmetros para execução do Neutrino.
 O formato do arquivo é JSON, confira um exemplo:

```json
{
   "Algorithm" : "python",
   "CPUMask" : 15,
   "ExecutionClient" : "demo.conf",
   "MDChannel" : { "<ALIAS>": "<IP>:<PORTA>"},
   "AlgoId" : 0,
   "AlgoClusterId" : 0,
   "Parent" : 0,
   "FrontendIdleTimeout": 60,
   "pos": {
      "DI1F21": {
         "net": -5,
         "net_price": 0.025
      }
   },
   "Symbols" : [ "DOLM18" ],
   "class" : "SimpleCorpOrder",
   "import" : "algo_modules",
   "StrictRisk": false
}
```

Onde os parâmetros disponíveis são:

| **Nome**     | **Descrição**                                                                                                                       | **Valores** |
|--------------|-------------------------------------------------------------------------------------------------------------------------------------|-------------|
| Algorithm    | Tipo de algoritmo                                                                                                          | "python", "corporate"|
| Symbol       | Lista dos nomes dos ativos                                                                                                     | lista de strings |
| StrictRisk   | Se e somente se o simbolo de uma ordem não tenha uma configuração de risco, esse parâmetro configura se tal ordem será bloqueada ou não. Se "true": bloqueia (strict). Se "false": aceita.|bool|
| pos          | A chave é o nome do ativo e o valor é um dict composto por `net` e `net_price`, onde o primeiro termo se refere à posição atual no instrumento e net_price o seu preço médio.|dict|
| CPUMask      | Máscara usada pelo taskset para determinar alocação de CPU pelo sistema operacional                                                  | int        |
| AlgoClusterId | Process ID atribuido pelo sistema operacional e preenchido pelo AlgoMan                                                             | int        |
| AlgoId       | Process ID atribuido pelo neutrino e preenchido pelo AlgoMan                                                                         | int        |
| FrontendIdleTimeout | imeout para interação com o frontend. Sem comunicação com o frontend além do timeout a estrategia é encerrada. Se o valor igual a zero, esta validação não é feita|int|
| ExecutionClient | Caminho para o arquivo com a configuração do cliente de ordens                                                                    | string      |
| class        | Nome do classe do algoritmo                                                                                                          | string      |
| import       | Pasta com os algoritmos                                                                                                              | string      |
| MDChannel    | Dicionário com IP e porta dos Relays utilizados                                                                                      | dict de strings, no formato `<IP>:<porta>` |

<br/>
Para que este arquivo seja gerado corretamente, a estratégia deve ser cadastrada através do Admin com o atributo Classe no formato python:`<import>:<class>` e o atributo name como `<class>`.

### Order Routing

O arquivo de configuração determina os parâmetros para execução do
Neutrino relacionados ao envio de ordens. Segue o exemplo:

```shell
port = 14006
user = neutrino
password = neutrino
account = DEMO
exchange = XBMF
prefix = XBMF.
report-delay = 500
```

Onde os parâmetros disponíveis são:

| **Nome**     | **Descrição**                                                                                                                       | **Valores** |
|--------------|-------------------------------------------------------------------------------------------------------------------------------------|-------------|
| port         | porta DTC do OMS                                                                                                                    | int         |
| user         | nome de usuário para conexão com o OMS                                                                                              | string      |
| password     | senha do usuário para conexão com o OMS                                                                                             | string      |
| account      | conta para envio de ordens                                                                                                          | string      |
| exchange     |                                                                                                                                     | string      |
| prefix       | prefixo utilizado no client order id das ordens enviadas                                                                            | string      |
| report-delay | Indica qual deve ser o tempo de delay da simulação. Caso seja colocado um valor negativo o código interpreta como valor de delay 0. | int         |

<br>

## Rodando estratégias no Docker

No repositório [neutrino-devel](https://github.com/onesoftsa/neutrino-devel),
 inclua a pasta com os arquivos relacionados à estratégia que quiser testar na
 pasta `strats/` e preencha o arquivo `scripts/confs/strastlist.yml` com um
 alias para sua estratégia (que utilizará com outro script), o nome da pasta
 onde os arquivos estão e o nome da classe que será importada pelo neutrino,
 conforme modelo na pasta `scripts/confs/`. Depois disso, navegue para a pasta
 raiz do repositório (onde fica o README) e rode:

```shell
$ python scripts/create_algo.py -a <ALIAS> -id <ID para usar na pasta> -i <INSTRUMENTO PARA TESTAR>
$ make neutrino-env
```

Já dentro do docker, rode:

```shell
$ cd ~/onesoft/algos/algo<ID>
$ neutrinov4 -c <ARQUIVO DA PASTA>.conf
```

Isso iniciará o `neutrino` em modo de simulação, utilizando dados em tempo
 real, e todo evento de book ou negócio chamará os callbacks para sua
 estratégia. Para testar os comandos que implementar, digite em um novo console:

```shell
 $ controller <algo-id> <unix-domain-socket-path>
 $ set <NOME ESTRATÉGIA> comandos-em-formato-json-aqui
```
