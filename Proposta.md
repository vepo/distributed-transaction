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

### (1). [Carrinho] Efetivar Compra

Essa é a operação principal de todo processo de compra. Ela deverá chamar cada operação e registrar os passos localmente. Se houver alguma falha na operação, ela pode ser recuperada posteriormente.

```
carrinho ← (itens,valorTotal, estado)                            // Um carrinho tem uma lista de itens selecionados junto com a quantidade
                                                                 // O valorTotal é derivado dessa informação
                                                                 // O estado inicial do carrinho é ABERTO
transação ← CriarTransação()                                     // O método criar transação abre a transação gerando um id único
carrinho.transação ← transação
carrinho.estado ← FINALIZANDO

transação.checkPoint(1)
resposta ← Estoque.Reservar(transação, carrinho.itens)           // Encapsula os passos [2] e [3]

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
resposta ← Pagamentos.Cobrar(transação)                          // Encapsula os passos [6] e [7]

SE (resposta.estado == FALHA) ENTÃO
    carrinho.estado ← INVÁLIDO
    Pagamentos.CancelarReserva(transação)
    Estoque.CancelarReserva(transação)
    transação.Finalizar()
SENÃO
    carrinho.estado ← PAGO

transação.checkPoint(4)
resposta ← Estoque.Despachar(transação)                          // Encapsula os passos [8] e [9]

SE (resposta.estado == SUCESSO) ENTÃO
    carrinho.estado ← FECHADO                                    // Passo [10]
    transação.Finalizar()
SENÃO
    transação.retentar()
```

### (2). [Estoque] Reservar Produtos

O serviço de estoque funcionará como um **Countdown Latch**. Serão feitas reservas até não haver mais o produto em estoque, quando uma reserva é liberada ou efetivada a próxima na fila pode ser processado.

```
emEstoque ← TemEstoque(requisição.itens)

SE (emEstoque == requisição.itens) ENTÃO
    reserva = NovaReserva(transação)
    reserva.itens ← requisição.itens
    Bloqueia(reserva)                                            // Remove o item do estoque e adiciona como reserva
    reserva.estado = RESERVADO                                   // as reservas devem ser processadas por ordem de chegada em fila
    RETORNA PRODUTOS_RESERVADOS

emReserva ← TemReservaPendente(requisição.itens, emEstoque)      // Deve calcular só os itens pendentes

SE (emEstoque + emReserva == requisição.itens) ENTÃO
    SE EsperaReservas(emReserva) ENTÃO                           // Operação Bloqueante
        reserva = NovaReserva(transação)
        reserva.itens ← requisição.itens
        Bloqueia(reserva)                                        // Remove o item do estoque e adiciona como reserva
        reserva.estado = RESERVADO                               // as reservas devem ser processadas por ordem de chegada em fila
        RETORNA PRODUTOS_RESERVADOS
    SENÃO
        RETORNA SEM_ESTOQUE
SENÃO
    RETORNA SEM_ESTOQUE
```

### (3). [Pagamentos] Reservar Saldo

O sistema de pagamentos funciona de maneira similar, ele é um gateway para o sistema bancário e reservas podem ser feitas.

```
emConta ← TemSaldo(requisição.valorTotal)

SE (emConta == requisição.valorTotal) ENTÃO
    reserva = NovaReserva(transação)
    reserva.valor ← requisição.valorTotal
    Bloqueia(reserva)                                            // Remove o item do estoque e adiciona como reserva
    reserva.estado = RESERVADO                                   // as reservas devem ser processadas por ordem de chegada em fila
    RETORNA SALDO_RESERVADO

emReserva ← TemReservaPendente(requisição.valorTotal, emConta)   // Deve calcular só os itens pendentes

SE (emConta + emReserva == requisição.valorTotal) ENTÃO
    SE EsperaReservas(emReserva) ENTÃO                           // Operação Bloqueante
        reserva = NovaReserva(transação)
        reserva.valor ← requisição.valorTotal
        Bloqueia(reserva)                                        // Remove o item do estoque e adiciona como reserva
        reserva.estado = RESERVADO                               // as reservas devem ser processadas por ordem de chegada em fila
        RETORNA SALDO_RESERVADO
    SENÃO
        RETORNA SEM_SALDO
SENÃO
    RETORNA SEM_SALDO
```

### (4). [Pagamentos] Cobrar

A cobrança deve recuperar a reserva através da transação e efetivar ela.

```
reserva ← EncontraReserva(transação)
EfetivaCobrança(reserva)
LiberaBloqueios(reserva)
RETORNA SUCESSO
```

### (5). [Estoque] Despachar Produtos


O despacho dos produtos deve recuperar a reserva através da transação e liberar para entrega.

```
reserva ← EncontraReserva(transação)
LiberarParaEntrega(reserva)
LiberaBloqueios(reserva)
RETORNA SUCESSO
```

### [Estoque] Cancelar Transação

Para cancelar a transação, deve-se verificar se existe uma reserva e retornar os itens para o estoque.

```
reserva ← EncontraReserva(transação)
RetornaParaEstoque(reserva)
LiberaBloqueios(reserva)
RETORNA CANCELADO
```

### [Pagamentos] Cancelar Transação

Para cancelar a transação, deve-se verificar se existe uma reserva e retorna o valor para saldo.

```
reserva ← EncontraReserva(transação)
RetornaParaSaldo(reserva)
LiberaBloqueios(reserva)
RETORNA CANCELADO
```

### [Carrinho] Recuperar Transação

A tentativa de recuperação do estado da transação é feito sempre que o serviço de Carrinho falhar. Como as transações são salvas em um arquivo de Write-Ahead Log, o processo deve ler todas as operações existente (não arquivadas).

```
transação ← PróximaTransação(log)                                // Recupera o último estado da transação armazenado
SE (transação.checkPoint ≤ 2) ENTÃO
    Estoque.CancelarReserva(transação)

SE (transação.checkPoint ≤ 3) ENTÃO
    Pagamentos.CancelarReserva(transação)
    transação.Finalizar()
    RETORNA CANCELADA

SE (transação.checkPoint = 4) ENTÃO
    resposta ← Estoque.Despachar(transação)                          // Encapsula os passos [8] e [9]

    SE (resposta.estado == SUCESSO) ENTÃO
        carrinho.estado ← FECHADO                                    // Passo [10]
        transação.Finalizar()
    SENÃO
        transação.retentar()
SENÃO
    transação.Finalizar()                                            // Falhou no último passo
```