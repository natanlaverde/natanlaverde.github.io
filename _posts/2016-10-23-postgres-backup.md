---
layout: post
title: "PostgreSQL Backup"
date: 2016-10-23
categories: PostgreSQL
permalink: archive/postgres-backup
---
Este artigo faz parte de um série de dois artigos sobre o PostgreSQL que escrevi enquanto estudava para um concurso que prestei em agosto.
Os artigos abordam os assuntos de [backup](postgres-backup) e [replicação](postgres-replicacao) no SGBDR.

Existem 3 modos de fazer backup no PostgreSQL, o backup em [Modo Texto](#modo-texto), o backup do
[Sistema de arquivos](#sistema-de-arquivos) e a [Arquivação contínua](#arquivao-contnua).

## Modo Texto
O *backup* em formato de texto é obtido com o programa utilitário **pg_dump** a forma básica de uso é da seguinte forma:
```bash
pg_dump dbname > outfile.sql
```

O pg_dump é uma aplicação cliente do PostgreSQL e não necessita de permissões especiais, necessitando apenas de acesso a leitura de todas as tabelas nos quais se realizará o *backup*.
No geral para exportar o banco de dados inteiro, é necessário de permissão de superusuário, mas para exportar porções do banco de dados é possível usar o *pg_dump* com as cláusulas *-n esquema* ou *-t tabela*.
Por padrão o *pg_dump* tentará autenticar utilizando o usuário do sistema operacional, porém pode ser especificado outro usuário através da cláusula *-U usuario*.

Uma vantagem do *pg_dump* sobre as demais formas de *backup* é que geralmente a saída pode ser restaurada para novas versões do PostgreSQL, enquanto as demais formas são extremamente específicas de cada versão.
Também pode ser utilizado para transferir um banco de dados para máquinas de diferentes aquiteturas.

Além disto, os arquivos criados com o *pg_dump* são internamentes consistentes, representando o estado do banco de dados quando o utilitário começa a executar.
O *pg_dump* não bloqueia as demais operações no banco de dados enquanto está sendo executado, excetuando-se as operações qe necessitam de trava exclusiva.

Os arquivos criados com o *pg_dump* podem ser lidos diretamente com o cliente *psql*. o comando geral para isto é:
```bash
psql dbname < infile.sql
```

O banco de dados não é criado com o comando e necessita ser criado anteriormente com o comando:
```bash
createdb -T template0 dbname
```

Antes de restaurar o *backup*, é necessário que todos os usários que sejam donos dos objetos ou que possuam permissões nestes objetos existam. Caso eles não existam a restauração irá falhar para recriar estes objetos.
Por padrão, o script psql irá continuar a executar após um erro ser encontrado, esse comportamento pode ser alterado com a variável ON_ERROR_STOP:
```bash
psql --set ON_ERROR_STOP=on dbname < infile.sql
```

O *pg_dump* pode trabalhar com pipes, tornando possível executar um dump de um banco de dados de um servidor diretamente para outro:
```bash
pg_dump -h host1 dbname | psql -h host2 dbname
```

O pg_dump funciona com apenas um banco de dados por vez e não opera sobre as informações de *roles* e *tablespaces*.
O utilitário *pg_dumpall* realiza o *backup* de cada banco de dados em um dado *cluster* e preserva as definições de *role* e *tablespace*.
O comando geral é da seguinte forma:
```bash
pg_dumpall > outfile.sql
```

E pode ser restaurado da seguinte forma:
```bash
psql -f infile.sql postgres
```

O comando trabalha emitindo os comandos para criação de *roles*, *tablespaces*, and dos banco de dados vazios, e então invoca o comando *pg_dump* para cada banco de dados.
Desta forma, cada banco de dados será internamente consistente, porém os estados de bancos de dados diferentes não estarão sincronizados.

Alguns sistemas operacionais podem ter problemas com limites de tamanhos de arquivos. Porém como o pg_dump escreve diretamente na saída padrão, as ferramentas Unix podem ser utilizadas para solucionar possíveis problemas:
Como por exemplo o *gzip*:
```bash
pg_dump dbname | gzip > filename.gz
```

Restaurar com:
```bash
gunzip -c filename.gz | psql dbname
```

Ou:
```bash
cat filename.gz | gunzip | psql dbname
```

O comando *split* pode ser utilizado para dividir um arquivo para arquivos menores, por exemplo para criar pedaços de 1 mb:
```bash
pg_dump dbname | split -b 1m - filename
```

Restaurar com:
```bash
cat filename* | psql dbname
```

Os backups também podem ser realizados em formato customizado do *pg_dump*.
Se o postgreSQL foi compilado com a biblioteca de compressão *zlib* instalada, os dados do *dump* serão comprimidos no arquivo de saída.
O arquivo de saída será similar ao produzido pelo *gzip*, porém terá a vantagem de que as tabelas podem ser restaurades de maneira seletiva.
O seguinte comando irá cria um arquivo de saída customizado.
```bash
pg_dump -Fc dbname > filename
```

Um *dump* criado com o formato customizado necessita ser restaurado com o utilitário *pg_restore*:
```bash
pg_restore -d dbname filename
```

O comando *pg_dump* também pode ser utilizado de forma paralela, agilizando o *dump* de grandes bases de dados.
Desta forma serão exportadas diversas tabelas ao mesmo tempo.
O grau de paralelismo pode ser controlado utilizado o parâmetro *-j num* e suporta apenas o modo de arquivo diretório:
'pg_dump -j num -F d -f out.dir dbname'

Para restaurar um *dump* paralelo também é necessário a utilização do *pg_restore*.

Para segurança outros programas como o *openssl* ou *gpg* podem ser utilizados para a criação de arquivos criptografados utilizando *pipes* ou redirecionamentos.

## Sistema de Arquivos

Uma outra alternativa para realizar o backup é copiar diretamente os arquivos onde o PostgreSQL armazena o banco de dados, por exemplo:
```bash
tar -cf backup.tar /usr/local/pgsql/data
```

Porém este modo de *backup* possui algumas restrições:
* O servidor necessita ser parado para poder ser gerado um *backup* que possa ser restaurado. Isto se deve em partes ao fato da ferramenta *tar* e  outras similares não gerarem um *snapshot* atómico do estado do sistema de arquivos.
Da mesma forma, o servidor necessita ser parado antes dos dados serem restaurados.

* Se você tentar entrar nos detalhes do sistema de arquivo do banco de dados, pode ficar tentado a tentar restaurar somente algumas tabelas ou bases de dados individuais de seus respectivos arquivos ou diretórios.
Isto não irá funcionar por que as informações contidas nestes arquivos não podem ser usadas sem os arquivos de *log*, que contém os *commit status* para todas as transações.
Assim fica impossível restaurar apenas uma tabela, fazendo que os backups por sistemas de arquivos funcionem somente para a restauração da base de dados completa.

Uma abordagem alternativa é fazer um *snapshot* consistente do diretório de dados, se o sistema de arquivos suportar essa funcionalidade (e se você confiar na implementação).
O procedimento típico é fazer um *frozen snapshot* do volume que contém a base de dados, e copiar o diretório inteiro deste *snapshot* para o dispositivo de *backup* e em seguida liberar o *frozen snapshot*.
Quando no momento da inicializão do servidor para restarar os backups criados desta forma, o servidor irá imaginar que o serviço não foi corretamente desligado e irá refazer os *WAL log*.
Isto não é um problema, porém esteja certo de que os *WAL logs* estão inclusos no *backup*.
Você também pode realizar um *CHECKPOINT* antes do *snapshot* reduzindo o tempo de recuperação.

Se a base de dados est Continuous Archiving and Point-in-Time Recovery (PITR)iver distribuido sobre diversos sistemas de arquivos, pode não ser possível realizar o *frozen snapshot* simutaneamente para todos os volumes.
Caso isso não seja possível, uma opção para realizar o backup a nível de sistema de arquivos é parar o servidor.

Outra opção é utilizar o *rsync* para realizar o backup a nível de sistema de arquivos.
Isto é feito utilizando o *rsync* enquanto o servidor está sendo executado, e em seguida pará-lo enquanto executa o *rsync --checksum*.
O segundo *rsync* será mais rápido que o primeiro, por causa que há somente poucos dados para serem transferidos e o resultado final será consistente, pois o servidor está parado.
Este método permite um backup a nível de sistema de arquivos com um período mínimo de *downtime*.

Note que o backup a nível de sistema de arquivos, tipicamente será maior que o *dump* SQL.
No entanto, fazer o backup a nível de sistema de arquivos pode ser mais rápido.

## Arquivação contínua

O PostgreSQL mantém um registro prévio da escrita -- *write ahead log (WAL)* no diretório de *logs*.
Este *log* contém todas as modificações feitas nos arquivos da base de dados.
Este *log* tem como objetivo primário a recuperação de falhas.
No entando a existência do *log* permite combinar um backup baseado em nível de sistema de arquivos com os arquivos de *log*.
Se uma recuperação é necessária, o sistema de arquivos pode ser recuperado, e em seguida, a recuperação pelo *log* ser realizada.
Esta abordagem é mais complexa, porém pode trazer benefícios significativos.

Não é necessário a consistência do backup de sistema de arquivos no ponto inicial, pois qualquer inconsistência interna será corrigida pelo recuperação do *log*.
Desta forma não é necessária a capacidade de tomar um *snapshot* do sistema de arquivos, apenas uma ferramenta como o *tar*.

Desde que sejam combinados uma sequência indefinidamente long de arquivos de *WAL log*, o backup continuo pode ser arquivado apenas continuamente arquivando os arquivos de *log*.
Isto é particularmente valoroso para grandes bases de dados, onde pode não ser conveniente realizar um backup completo com frequência.

Como não é necessário realizar a execução dos comando do log de forma completa.
Pode-se parar a execução em um dado ponto e ter um *instantâneo* da base de dados em um dado tempo.
Esta técnica permite o suporte a recuperação em um dado ponto no tempo, sendo possível restaurar a base dados até um dado tempo desde que o backup da base de dados foi realizado.

Se você continuamente enviar os arquivos de *WAL* para outra máquina, carregada com o mesmo arquivo de backup, é possível manter um sistema *warm standby*, em um dado ponto é possível levantar uma segunda máquina e ter uma cópia próxima da cópia corrente da base de dados.

Para recuperar com sucesso utilizando o arquivamento contínuo, é necessário uma sequência continua dos arquivos de *log* que se estendam pelo menos até o momento que o backup foi iniciado.


## Referências:
[PostgreSQL Documentation - Chapter 24. Backup and Restore](https://www.postgresql.org/docs/9.4/static/backup.html)
