# Proposta

Aplicação de compras online em que o sistema de pagamentos e o estoque são separados. Temos então ao menos 3 sistemas envolvidos na efetivação de uma compra: o carinho que irá receber todas as informações da compra, o estoque que precisa validar se os itens adquiridos tem em quantidades satisfatórias e o sistema de pagamento.

Assim, uma compra pode ser efetivada pelos seguintes passos:

```
            +---------+               +----------+              +---------+         +------------+
#ID         | Usuário |               | Carrinho |              | Estoque |         | Pagamentos |
            +---------+               +----------+              +---------+         +------------+
                 |                          |                        |                     |
                 |        Efetivar          |                        |                     |
[ 1]             |------------------------->|                        |                     |
                 |                          |                        |                     |
                 |                          |       Reservar         |                     |
[ 2]             |                          |----------------------->|                     |
                 |                          |                        |                     |
                 |                          |       Reservado        |                     |
[ 3]             |                          |<-----------------------|                     |
                 |                          |                                              |
                 |                          |                    Reservar                  |
[ 4]             |                          |--------------------------------------------->|
                 |                          |                                              |
                 |                          |                Reserva Realizada             |
[ 5]             |                          |<---------------------------------------------|
                 |                          |                                              |
                 |                          |                     Cobrar                   |
[ 6]             |                          |--------------------------------------------->|
                 |                          |                                              |
                 |                          |                Cobrança Realizada            |
[ 7]             |                          |<---------------------------------------------|
                 |                          |                                              |
                 |                          |       Despachar        |                     |
[ 8]             |                          |----------------------->|                     |
                 |                          |                        |                     |   
                 |                          |          OK            |                     |  
[ 9]             |                          |<-----------------------|                     |
                 |                          |                        |                     | 
                 |        Sucesso           |                        |                     |
[10]             |<-------------------------|                        |                     |
                 |                          |                        |                     |
```
> **Convenções**
>
> Usaremos os termos **operações** e **passos**. Operações irá se referir a uma interação completa entre os serviços de Carrinho e Estoque/Pagamento e deverá estar com seu identificador entre parenteses. Já um passo é a requisição ou a resposta de uma operação e deverá ter seu identificador entre colchetes.


Assim podemos falar que deverá existir uma transação para operação de **Efetivar Compra** (1) no carrinho e que ela deverá conter as operações de **Reservar** (2) e **Despachar** (5) no Estoque e as operações de **Reservar** (3) e **Cobrar** (4) no Pagamentos.

* Se as operações (2) ou (3) falharem, as operações (4) e (5) não podem ser efetuadas.
* Os recursos reservados das operações (2) e (3) ficam reservados até elas serem confirmadas ou canceladas
* Toda consulta ao estoque a ao pagamento deverá considerar o Saldo/Quantidade e Reservas
* Se o sistema de Carrinho falhar enquanto estiver realizando a operação (1), ele deverá ler todas as entradas de log para decidir se a operação pode ser concluída ou não.

## Limitações Arquiteturais

* O sistema não tem uma base de dados SQL permitindo ACID
* Toda operação envolve a escrita de uma escrita em um Write-Ahead Log
* Toda comunicação é feita através de HTTP REST
* Toda transação tem um timeout

## Tratamento de Erros

* Se o sistema falhar, o Write-Ahead Log deve ser verificado para tentar recupera o estado da transação.
* A transação só pode ser recuperada se finalizou o passo [5] con sucesso, caso contrário deverá executar o processo de rollback.
* O arquivo de Write-Ahead Log deve ser finalizado se nenhuma operação registrada nele continua com transação ativa ativa.
* Cada mensagem deverá conter um identificador de transação, gerado pelo Carrinho, e o estado dessa transação pode ser consultado

## Operações

### 1. [Carrinho] Efetivar Compra

```
carrinho ← (itens,valorTotal, estado)                            // Um carrinho tem uma lista de itens selecionados junto com a quantidade
                                                                 // O valorTotal é derivado dessa informação
                                                                 // O estado inicial do carrinho é ABERTO
transação ← CriarTransação()                                     // O método criar transação abre a transação gerando um id único
carrinho.transação ← transação
carrinho.estado ← FINALIZANDO

transação.checkPoint(1)
resposta ← Estoque.Reservar(transação, carrinho.items)           // Encapsula os passos [2] e [3]

SE (resposta.estado == SEM_ESTOQUE OU 
    resposta.estado = FALHA) ENTÃO
    carrinho.estado ← INVÁLIDO
    Estoque.CancelarReserva(transação)
    transação.Finalizar()
SENÃO
    carrinho.estado ← PRODUTOS_RESERVADOS

transação.checkPoint(2)
resposta ← Pagamentos.Reservar(transação, carrinho.items)        // Encapsula os passos [4] e [5]

SE (resposta.estado == SEM_SALDO OU
    resposta.estado == FALHA) ENTÃO
    carrinho.estado ← INVÁLIDO
    Pagamentos.CancelarReserva(transação)
    Estoque.CancelarReserva(transação)
    transação.Finalizar()
SENÃO
    carrinho.estado ← SALDO_RESERVADO

transação.checkPoint(3)
resposta ← Pagamentos.Cobrar(transação)

SE (resposta.estado == FALHA) ENTÃO
    carrinho.estado ← INVÁLIDO
    Pagamentos.CancelarReserva(transação)
    Estoque.CancelarReserva(transação)
    transação.Finalizar()
SENÃO
    carrinho.estado ← PAGO

transação.checkPoint(4)
resposta ← Estoque.Despachar(transação)

SE (resposta.estado == SUCESSO) ENTÃO
    carrinho.estado ← FECHADO
    transação.Finalizar()
SENÃO
    transação.retentar()
```


