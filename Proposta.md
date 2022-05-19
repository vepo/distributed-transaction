# Proposta de Serviço com _Two-Phase Commit_

**UNIVERSIDADE TECNOLÓGICA FEDERAL DO PARANÁ**

**Aluno**: Victor Emanuel Perticarrari Osório

**Disciplina**: Sistemas Distribuídos

**Professora**: Ana Cristina Barreiras Kochem Vendramin


## Descrição da Solução

Aplicação de compras online em que o sistema de pagamentos e o estoque são separados. Temos então ao menos 3 sistemas envolvidos na efetivação de uma compra: o carinho que irá receber todas as informações da compra, o estoque que precisa validar se os itens adquiridos tem em quantidades satisfatórias e o sistema de pagamento.

Assim, nosso sistema será composto pelos microsserviços **Carrinho** (coordenador da transação), **Estoque** e **Pagamentos**. Par que uma compra possa ser efetivada com sucesso os seguintes passos devem ser executados:

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

<style>
blockquote {
  border: solid 1px #888;
  margin: 5px;
  padding: 15px 25px;
  background: #EEE;
}

blockquote p:first-child {
  font-size: 1.3em;
}
</style>
> **Convenções**
>
> Usaremos os termos **operações** e **passos**. Operações irá se referir a uma interação completa entre os serviços de Carrinho e Estoque/Pagamento e deverá estar com seu identificador entre parenteses. Já um passo é a requisição ou a resposta de uma operação e deverá ter seu identificador entre colchetes.

Assim podemos falar que deverá existir uma transação para operação de **Efetivar Compra** (1) no carrinho e que ela deverá conter as operações de **Reservar** (2) e **Despachar** (5) no Estoque e as operações de **Reservar** (3) e **Cobrar** (4) no Pagamentos.

* Se as operações (2) ou (3) falharem, as operações (4) e (5) não podem ser efetuadas.
* Os recursos reservados das operações (2) e (3) ficam reservados até elas serem confirmadas ou canceladas
* Toda consulta ao estoque a ao pagamento deverá considerar o Saldo/Quantidade e Reservas
* Se o sistema de Carrinho falhar enquanto estiver realizando a operação (1), ele deverá ler todas as entradas de log para decidir se a operação pode ser concluída ou não.
* As operações (2) e (3) são operações as _prepare_ do **_Two Phase Commit_**, enquanto as operações (4) e (5) são operações de _commit_.

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
* Se um sistema ficar indisponível, outros sistemas podem continuar a operação sem depender dele. Ele deve depois garantir que todas as operações estão sincronizadas.

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

SE (resposta.estado = SEM_ESTOQUE OU
    resposta.estado = FALHA) ENTÃO
    carrinho.estado ← INVÁLIDO
    Estoque.CancelarReserva(transação)
    transação.Rollback()
SENÃO
    carrinho.estado ← PRODUTOS_RESERVADOS

transação.checkPoint(2)
resposta ← Pagamentos.Reservar(transação, carrinho.items)        // Encapsula os passos [4] e [5]

SE (resposta.estado = SEM_SALDO OU
    resposta.estado = FALHA) ENTÃO
    carrinho.estado ← INVÁLIDO
    Pagamentos.CancelarReserva(transação)
    Estoque.CancelarReserva(transação)
    transação.Rollback()
SENÃO
    carrinho.estado ← SALDO_RESERVADO

transação.checkPoint(3)
resposta ← Pagamentos.Cobrar(transação)                          // Encapsula os passos [6] e [7]

SE (resposta.estado = FALHA) ENTÃO
    carrinho.estado = INVÁLIDO
    Pagamentos.CancelarReserva(transação)
    Estoque.CancelarReserva(transação)
    transação.Rollback()
SENÃO
    carrinho.estado ← PAGO

transação.checkPoint(4)
resposta ← Estoque.Despachar(transação)                          // Encapsula os passos [8] e [9]

SE (resposta.estado = SUCESSO) ENTÃO
    carrinho.estado = FECHADO                                    // Passo [10]
    transação.Commit()
SENÃO
    transação.retentar()
```

### (2). [Estoque] Reservar Produtos

O serviço de estoque funcionará como um **Countdown Latch**. Serão feitas reservas até não haver mais o produto em estoque, quando uma reserva é liberada ou efetivada a próxima na fila pode ser processado.

```
transação ← CriarTransaçãoLocal(requisição, transaçãoCoordenador)
emEstoque ← TemEstoque(requisição.itens)

SE (emEstoque = requisição.itens) ENTÃO
    reserva ← NovaReserva(transação)
    reserva.itens ← requisição.itens
    Bloqueia(reserva)                                            // Remove o item do estoque e adiciona como reserva
    reserva.estado ← RESERVADO                                   // as reservas devem ser processadas por ordem de chegada em fila
    transação.Commit()
    RETORNA PRODUTOS_RESERVADOS

emReserva ← TemReservaPendente(requisição.itens, emEstoque)      // Deve calcular só os itens pendentes

SE (emEstoque + emReserva = requisição.itens) ENTÃO
    SE EsperaReservas(emReserva) ENTÃO                           // Operação Bloqueante
        reserva ← NovaReserva(transação)
        reserva.itens ← requisição.itens
        Bloqueia(reserva)                                        // Remove o item do estoque e adiciona como reserva
        reserva.estado ← RESERVADO                               // as reservas devem ser processadas por ordem de chegada em fila
        transação.Commit()
        RETORNA PRODUTOS_RESERVADOS
    SENÃO
        transação.Rollback()
        RETORNA SEM_ESTOQUE
SENÃO
    transação.Rollback()
    RETORNA SEM_ESTOQUE
```

### (3). [Pagamentos] Reservar Saldo

O sistema de pagamentos funciona de maneira similar, ele é um gateway para o sistema bancário e reservas podem ser feitas.

```
transação ← CriarTransaçãoLocal(requisição, transaçãoCoordenador)
emConta ← TemSaldo(requisição.valorTotal)

SE (emConta = requisição.valorTotal) ENTÃO
    reserva ← NovaReserva(transação)
    reserva.valor ← requisição.valorTotal
    Bloqueia(reserva)                                            // Remove o item do estoque e adiciona como reserva
    reserva.estado ← RESERVADO                                   // as reservas devem ser processadas por ordem de chegada em fila
    transação.Commit()
    RETORNA SALDO_RESERVADO

emReserva ← TemReservaPendente(requisição.valorTotal, emConta)   // Deve calcular só os itens pendentes

SE (emConta + emReserva = requisição.valorTotal) ENTÃO
    SE EsperaReservas(emReserva) ENTÃO                           // Operação Bloqueante
        reserva ← NovaReserva(transação)
        reserva.valor ← requisição.valorTotal
        Bloqueia(reserva)                                        // Remove o item do estoque e adiciona como reserva
        reserva.estado ← RESERVADO                               // as reservas devem ser processadas por ordem de chegada em fila
        transação.Commit()
        RETORNA SALDO_RESERVADO
    SENÃO
        transação.Rollback()
        RETORNA SEM_SALDO
SENÃO
    transação.Rollback()
    RETORNA SEM_SALDO
```

### (4). [Pagamentos] Cobrar

A cobrança deve recuperar a reserva através da transação e efetivar ela.

```
transação ← CriarTransaçãoLocal(requisição, transaçãoCoordenador)
reserva ← EncontraReserva(transação)
EfetivaCobrança(reserva)
LiberaBloqueios(reserva)
transação.Commit()
RETORNA SUCESSO
```

### (5). [Estoque] Despachar Produtos


O despacho dos produtos deve recuperar a reserva através da transação e liberar para entrega.

```
transação ← CriarTransaçãoLocal(requisição, transaçãoCoordenador)
reserva ← EncontraReserva(transação)
LiberarParaEntrega(reserva)
LiberaBloqueios(reserva)
transação.Commit()
RETORNA SUCESSO
```

### [Estoque] Cancelar Transação

Para cancelar a transação, deve-se verificar se existe uma reserva e retornar os itens para o estoque.

```
transação ← CriarTransaçãoLocal(requisição, transaçãoCoordenador)
reserva ← EncontraReserva(transação)
RetornaParaEstoque(reserva)
LiberaBloqueios(reserva)
transação.Commit()
RETORNA CANCELADO
```

### [Pagamentos] Cancelar Transação

Para cancelar a transação, deve-se verificar se existe uma reserva e retorna o valor para saldo.

```
transação ← CriarTransaçãoLocal(requisição, transaçãoCoordenador)
reserva ← EncontraReserva(transação)
RetornaParaSaldo(reserva)
LiberaBloqueios(reserva)
transação.Commit()
RETORNA CANCELADO
```

### [Carrinho] Recuperar Transação

A tentativa de recuperação do estado da transação é feito sempre que o serviço de Carrinho falhar. Como as transações são salvas em um arquivo de Write-Ahead Log, o processo deve ler todas as operações existente (não arquivadas).

```
transação ← PróximaTransação(log)                                // Recupera o último estado da transação armazenado
SE (transação.checkPoint ≤ 1) ENTÃO
    Estoque.CancelarReserva(transação)
    transação.checkPoint(2)
    transação.Rollback()
    RETORNA CANCELADA                                            // Como não chegou ao ponto 2, só foi feita reserva no Estoque

SE (transação.checkPoint ≤ 2) ENTÃO
    Pagamentos.CancelarReserva(transação)
    transação.checkPoint(3)
    transação.Rollback()
    RETORNA CANCELADA                                            // Cmo não chegou ao ponto 3, rollback concluído

SE (transação.checkPoint ≤ 3) ENTÃO
    resposta ← Estoque.Despachar(transação)                      // Encapsula os passos [8] e [9]

    SE (resposta.estado = SUCESSO) ENTÃO
        carrinho.estado ← FECHADO                                // Passo [10]
        transação.checkPoint(4)
        transação.Commit()
        RETORNA SUCESSO
    SENÃO
        transação.Retenta()
        RETORNA FALHA
SENÃO
    transação.checkPoint(4)
    transação.Commit()                                           // Falhou no último passo
    RETORNA SUCESSO
```

### [Estoque/Pagamentos] Recuperar Transação

Se os serviços de Estoque ou Pagamentos ficarem indisponíveis, o Carrinho compreenderá que ouve uma falha (estado FALHA na resposta da requisição) na comunicação e concluirá a transação. Por isso é preciso uma rotina de validar as transações perdidas para liberar bloqueios. A rotina abaixo deverá ser executada constantemente para garantir que nenhuma operação fique bloqueada e, caso bloqueios surja, o sistema deve validar novamente a primeira transação da lista.

```
transação ← PróximaTransação(log)                                // Recupera o último estado da transação armazenado
resposta = VerificarEstadoTransaçãoNoCoordenador(transação)
SE (resposta.estado == ROLLBACK) ENTÃO
    LiberaRecursos(transação)
    LiberaBloqueios(transação)
    transação.Rollback()
    RETORNA CANCELADA

SE (resposta.estado == COMMIT) ENTÃO
    EfetivaRecursos(transação)
    LiberaBloqueios(transação)
    transação.Commit()
    RETORNA SUCESSO

transação.Retenta()                                              // Falha de comunicação
RETORNA FALHA
```

No código acima, os métodos `EfetivaRecursos` e `LiberaBloqueios` encapsulam a respectiva lógica em cada serviço

## Respostas

### Como a propriedade de Atomicidade é garantida?

A operação de Recuperar Transação pode retomar o ponto em que houve a falhar e decidir se faz o rollback ou finaliza a operação.

### Como a propriedade de Isolamento é garantida?

O isolamento é garantido através do bloqueio tipo **CountDown Latch**. Se há estoque, ele é reduzido e a transação segue. Mas não há estoque suficiente, o bloqueio é feito até que as operações possam ser finalizadas e a decisão se existe ou não estoque possa ser tomada.

### Como a propriedade de Durabilidade é garantida?

A Durabilidade é garantida pelo uso do Write-Ahead Log. Antes de qualquer operação, o estado é gravado, permitindo a operação recuperar estado. Se a escrita do Write-Ahead Log falhar a operação não é iniciada, o que significa que se houver uma falha,o Write-Ahead Log contém ao menos a última operação iniciada.

### Como a propriedade de Consistência é garantida?

A Consistência é garantida pela operação de Recuperar Transação associada ao **CountDown Latch**, pois o banco de dados poderá recuperar o estado anterior em caso de falha, mas evitando problemas de estoque ao não permitir que a operação continue enquanto houverem itens reservados.