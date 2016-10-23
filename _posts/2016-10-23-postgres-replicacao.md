---
layout: post
title: "PostgreSQL Replicação, balanceamento de carga e alta disponibilidade"
date: 2016-10-23
categories: PostgreSQL
permalink: archive/postgres-replicacao
---
Este artigo faz parte de um série de dois artigos sobre o PostgreSQL que escrevi enquanto estudava para um concurso que prestei em agosto.
Os artigos abordam os assuntos de [backup](postgres-backup) e [replicação](postgres-replicacao) no SGBDR.

Servidores de base de dados podem trabalhar juntos permitindo um segundo servidor assumir rapidamente se o servidor primário falhar (alta disponibilidade), ou permitindo diversos compudares servirem os mesmos dados (balanceamento de carga).

Idealmente, os Servidores de base de dados podem trabalhar juntos perfeitamente.
Servidores *web* servindo páginas estáticas podem ser combinados facilmente simplesmente realizando requisões a múltiplas máquinas.
De fato, base de dados somente leitura podem ser combinadas de forma relativamente simples também.
Infelizmente, muitos servidores de fazem misturas de requisições de leitura e escrita, tornando isto difícil de combinar.
Isto é por causa de que uma leitura necessita ser realizada em um servidor apenas uma vez, enquanto uma escrita precisa ser propagada a todos os servidores para futuras requisições de leitura possam retornar resultados consistentes.

Este problema de sincronização é a dificuldade fundamental para os servidores trabalharem juntos.
Por que não existe uma única solução que elimina o impacto dos problemas de sincronização para todos os caso, existindo múltiplas soluções.
Cada solução ataca este problema de maneira diferente, e minimiza os impactos para uma carga de trabalho específica.

Algumas soluções lidam com a sincronização permitindo apenas um servidor modificar os dados.
Servidores que podem modificaros dados são chamados de *read/write*, *mestre* ou servidor *primário*.
Servidores que podem rastrear as mudanças do *mestre* são chamados de *standby* ou *escravo*.
Um servidor *standby* que não permite conexõeas até que sejam promovidos para um servidor mestre é chamado de *warm standby*, enquanto um que aceite conexões *read-only* é chamado de *hot standby*.

Algumas soluções são síncronas, significando que os dados modificados por uma transação não são considerados *comitados* até que todos os servidores *comitaram* a transação.
Isto garante que em caso de falha, não irá perder nenhum dado e que todos os servidores do balanceamento de carga irão retornar dados consistentes, não importando qual dos servidores foi consultado.
As soluções *assíncronas* permitem algum atraso entre o tempo de commit e propagação para outros servidores.
Abrindo a possibilidade de Algumas transações serem perdidas na mudança para um servidor de backup e os servidores do balanceamento de carga poderam retornar resultados diferente.
As soluções *assíncronas* são utilizadas quando a comunicação usada quando síncrona é lenta.

As soluções podem ser categorizadas quanto a *glanularidade*.
Algumas soluções trabalham apenas com o banco de dados inteiro, enquanto outras permite o controle a *nível de tabela* e *nível de base de dados*.

O desempenho pode ser considerado em uma escolha.
Existe o *trade-off* tradicional entre funcionalidade e desempenho.
Por exemplo, uma solução completemente síncrona sobre uma conexão de rede lenta pode cortar o desempenho pela metade, enquanto uma solução assíncrona pode ter um impacto mínimo no desempenho.


## Diferentes soluções

### Disco compartilhado
Discos compartilhados evitam problemas de sincronização tendo apenas uma cópia da base de dados.
Usando um único arranjo de discos que é compartilhado entre múltiplos servidores.
Se um banco de dados principal falhar, é possível montar e inicializar a base de dados, recuperando da falha da base de dados.
Isto permite uma rápida recuperação sem perda de dados.

### Replicação do sistema de arquivos
Uma versão modificada do disco compartilhado permite a replicação do sistema de arquivos, de forma que as mudanças são espelhadas para um sistema de arquivos em outro computados.
A única restrição disto é que todos os espelhamentos necessitam ser feitos de forma que permitam os servidores *standby* possuirem uma cópia consistente do sistema de arquivos.
*DBRD* é uma solução popular de replicação para Linux.  

### Envio de log transacional
servidores *warm standby* e *hot standby* podem manter-se atualizados através da leitura de séries de registros de *write-ahead log (WAL)*
Se o servidor principal falhar, o servidor *standby* irá conter a maior parte dos dados no servidor principal e rapidamente poderá ser transformado em mestre.
Isto poder ser feito de forma síncrona ou assíncrona.

### Replicação mestre-escravo baseada em gatilho
Uma solução mestre-escravo envia todas as consultas com modificações para o servidor mestre.
O servidor mestre assíncronamente envia as mudanças de dados para o servidor *standby*.
O servidor *standby* pode receber apenas consultas somente leitura enquanto o servidor mestre estiver executando.
Este tipo de servidor *standby* é ideal para consultas de *data warehouse*.

O [Slony-I](http://slony.info/) é um exemplo deste tipo de replicação, com granularidade por tabela, suportando múltiplos servidores.
Por conta de as atualizações serem assíncronas, pode haver perda de dados em caso de uma falha.

### Replicação baseada em mediador
Nas replicações baseadas em mediador, um programa intercepta todas as consultas SQL e envia par todos os servidores.
Cada servidor opera de maneira independente.
Consultas de leitura e escrita necessitam ser enviadas a todos os servidores, de forma que todos os servidores recebam as modificações.
Porém as consultas somente leitura podem ser enviadas a apenas um servidor, permitindo que a carga de trabalho de leitura seja distribuída entre eles.
O [Pgpool-II](www.pgpool.net/) e o **Continuent Tungsten** são exemplos de replicações deste tipo.

### Replicação multi-mestre assíncrona
Para servidores que não são conectados regularmente, como laptops e servidores remotos, manter os dados consistentes entre os servidores pode ser um desafio.
Usaando a replicação multi-mestre assíncrona, cada servidor trabalha independente, e períodicamente se comunica com outros servidores para identificar transações em conflito.
Os conflitos podem ser resolvidos pelo usuário ou regras de resolução de conflito.
Um exemplo deste tipo de replicação é o [Bucardo](bucardo.org).

### Replicação multi-mestre síncrona
Na replicação multi-mestre síncrona, cada servidor aceita requisições de escrita e as modificaçãos dos dados são trasmitidas do servidor original para todos os outros servidores antes de cada transação **commitar**.
As atividades de escrita podem criar muitas travas, causando um problema de desempenho.
As transações de leitura podem ser enviadas para qualquer servidor.
Algumas implementações podem utilizar discos compartilhados para reduzir o *overhead*.
A replicação síncrona multi-mestre é melhor para muita carga de trabalho de leitura, com a principal vantagem de que qualquer servidor pode aceitar requisições de leitura.

O PostgreSQL não oferece este tipo de replicação, porém pode ser utilizado commit em duas fases (PREPARE TRANSACTION and COMMIT PREPARED) implementado na aplicação ou mediador.

### Soluções comerciais
Como o PostgreSQL é *open source* e facilmente extensível, inúmeras empresas utilizam o PostgreSQL e criam soluções comerciais fechadas com soluções únicas de falha, replicação e balanceamento de carga.

## Referências:
[PostgreSQL Documentation - Chapter 25. High Availability, Load Balancing, and Replication](https://www.postgresql.org/docs/9.4/static/high-availability.html)
