Avaliação (valor 2,5)
Transações

Descreva uma aplicação simples (cliente-servidor ou de processos pares) que tenha duas operações (exceto operações bancárias) que precisam ser executadas dentro de uma transação. Lembrando que duas operações precisam estar dentro de uma transação quando é necessária garantir que essas operações sejam executadas com sucesso ou que nenhuma seja executada (propriedade de atomicidade). Por exemplo, uma transferência bancária entre duas contas é composta por duas operações: (1) retirada de um valor de uma conta bancária; (2) inserção desse valor na outra conta bancária. Essas operações devem ser inseridas em uma transação para que não ocorra o seguinte: (i) a operação (1) ser executada com sucesso e a operação (2) falhar; ou (ii) a operação (1) falhar e a operação (2) ser executada com sucesso **(valor 0,5)**. 

Implemente essa aplicação ou apresente um pseudocódigo explicando **(valor 2,0)**: 

* o funcionamento do protocolo two-phase commit para comunicação entre os processos, explicando quem são os participantes e o coordenador. Detalhe os métodos necessários para funcionamento do protocolo **(valor 0,4)**;
* como a propriedade de Atomicidade é garantida **(valor 0,4)**;
* como a propriedade de Isolamento é garantida **(valor 0,4)**;
* como a propriedade de Durabilidade é garantida **(valor 0,4)**;
* como a propriedade de Consistência é garantida. Lembrar aqui que processos podem falhar no meio de transações **(valor 0,4)**.

Obs.: a aplicação não pode utilizar banco de dados e nem APIs para gerenciar transações.