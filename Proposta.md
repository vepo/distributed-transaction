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
> Usaremos os termos **operações** e **passos**. Operações irá se referir a uma interação completa entre os serviços de Carrinho e Estoque/Pagamento e deverá estar com seu identificado entre parenteses. Já um passo é a requisição e a resposta de uma operação e deverá ter seu identificador entre colchetes.


Assim podemos falar que deverá existir uma transação para operação de **Efetivar Compra** no carrinho e que ela deverá conter as operações de **Reservar** (1) e **Despachar** (4) no Estoque e as operações de **Reservar** (2) e **Cobrar** (3) no Pagamentos.

* Se as operações (1) ou (2) falharem, as operações (3) e (4) não podem ser efetuadas.
* Os recursos reservados das operações (1) e (2) ficam reservados até elas serem confirmadas ou canceladas
* Toda consulta ao estoque a ao pagamento deverá considerar o Saldo/Quantidade e Reservas.

## Limitações Arquiteturais

* O sistema não tem uma base de dados SQL permitindo ACID
* Toda operação envolve a escrita de uma escrita em um Write-Ahead Log
* Toda comunicação é feita através de HTTP REST
* Toda transação tem um timeout

## Tratamento de Erros

* Se o sistema falhar, o Write-Ahead Log deve ser verificado para tentar recupera o estado da transação.
* A transação só pode ser recuperada se finalizou o passo [5] con sucesso, caso contrário deverá executar o processo de rollback.
 


