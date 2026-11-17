# Backup de banco de dados

Texto de apoio de Gerência de Redes 2026.2. Os exemplos usam PostgreSQL, que é o
banco das atividades da disciplina, mas os conceitos valem para qualquer banco.

## Para que serve um backup

Um backup é uma cópia dos dados guardada de modo que seja possível voltar a um
estado anterior. Ele protege contra perdas que a redundância do servidor não
cobre:

- um `DELETE` sem `WHERE` digitado por uma pessoa;
- uma migração com defeito que corrompe uma tabela;
- um disco que falha;
- um programa malicioso que cifra os dados e pede resgate;
- um servidor perdido inteiro, por incêndio, roubo ou conta encerrada.

Replicação para outro servidor cobre o disco que falha e o servidor perdido. O
`DELETE`, a migração com defeito e a cifragem chegam à réplica alguns
milissegundos depois do original, e ela passa a ter o mesmo problema.

## RPO e RTO

O RPO e o RTO definem se um backup atende à necessidade de um sistema.

O **RPO** (*recovery point objective*) é quanto dado pode ser perdido, medido em
tempo. Com um backup por hora, o pior caso perde os 59 minutos de alterações
feitas depois da última cópia. Com um backup a cada 15 minutos, perde até 15.

O **RTO** (*recovery time objective*) é quanto tempo o sistema pode ficar fora do
ar até voltar a funcionar. Ele inclui perceber o problema, achar o backup certo,
copiá-lo de volta, restaurar e conferir.

O RPO é decidido pela frequência do backup. O RTO é decidido pelo procedimento de
restauração e só se conhece medindo.

## Backup lógico e backup físico

O PostgreSQL tem backup lógico e backup físico.

O **backup lógico** gera os comandos que recriam o banco: `CREATE TABLE`, os dados
das tabelas e os índices. A ferramenta é o `pg_dump`. Ele lê o banco por uma
transação, então a cópia é consistente mesmo com o banco recebendo escrita
durante a execução, e não bloqueia as outras sessões.

O **backup físico** copia os arquivos do diretório de dados do servidor. A
ferramenta é o `pg_basebackup`. Ele copia o cluster inteiro, com todos os bancos,
e só restaura na mesma versão principal do PostgreSQL.

| | Lógico (`pg_dump`) | Físico (`pg_basebackup`) |
|---|---|---|
| O que copia | um banco | o cluster inteiro |
| Restaura em outra versão | sim | não |
| Restaura uma tabela só | sim | não |
| Velocidade em banco grande | menor | maior |
| Recuperação a um instante qualquer | não | sim, com arquivamento de WAL |

Para bancos pequenos e médios, o `pg_dump` resolve. O backup físico com
arquivamento de WAL entra quando o RPO precisa ser de segundos. O WAL é o
registro de cada alteração feita no banco, e arquivá-lo é guardar esse registro
fora do servidor, de modo contínuo; a seção "Recuperação a um instante qualquer"
explica o mecanismo.

Copiar a pasta de dados com `cp` ou `tar` enquanto o banco está rodando produz um
backup que pode não abrir. Os arquivos mudam durante a cópia, e o resultado mistura
páginas de instantes diferentes. A cópia de arquivos só é segura com o banco
parado, com um *snapshot* do sistema de arquivos, ou pelo `pg_basebackup`.

## Uso do pg_dump

O `pg_dump` é um programa cliente que vem com o PostgreSQL. Ele se conecta ao
servidor como qualquer cliente, com servidor, porta, usuário e senha, lê um banco
inteiro e grava num arquivo tudo o que é preciso para recriá-lo: as definições das
tabelas, dos índices e das restrições, e o conteúdo de cada tabela.

Como trabalha pela conexão, o `pg_dump` roda de qualquer máquina que alcance o
servidor pela rede, sem acesso aos arquivos do servidor. O usuário da conexão
precisa ter permissão de leitura nas tabelas que vai copiar.

A leitura acontece dentro de uma única transação. O arquivo gerado mostra o banco
como ele estava no instante em que o `pg_dump` começou, mesmo que outras sessões
continuem inserindo e alterando dados durante a cópia, e essas sessões seguem
trabalhando sem esperar por ele.

A versão do `pg_dump` precisa ser igual ou mais nova que a versão do servidor. Um
`pg_dump` 15 recusa copiar um servidor 17, com a mensagem `server version
mismatch`. O contrário funciona: um `pg_dump` 17 copia um servidor 15.

### Formatos de saída

Dos formatos do `pg_dump`, o texto e o personalizado são os mais usados.

O formato texto (`-Fp`, o padrão) gera um arquivo `.sql` legível, que se restaura
com `psql`:

```bash
pg_dump -h localhost -U app -d loja -f loja.sql
psql -h localhost -U app -d loja_nova -f loja.sql
```

O formato personalizado (`-Fc`) gera um arquivo binário, já comprimido, que se
restaura com `pg_restore`:

```bash
pg_dump -h localhost -U app -Fc -d loja -f loja.dump
pg_restore -h localhost -U app -d loja_nova loja.dump
```

O formato personalizado permite restaurar só parte do conteúdo e listar o que
está dentro do arquivo antes de restaurar:

```bash
pg_restore --list loja.dump            # índice do conteúdo
pg_restore -d loja_nova -t pedidos loja.dump   # só a tabela pedidos
```

### Exemplo medido

Num banco de teste com uma tabela de 500 mil pedidos, 51 MB no disco, em
PostgreSQL 17.11, os números foram estes:

| Formato | Tamanho | Tempo do `pg_dump` |
|---|---|---|
| texto (`.sql`) | 29 MB | 0,6 s |
| texto comprimido com `gzip` | 4,0 MB | 0,6 s mais a compressão |
| personalizado (`-Fc`) | 4,0 MB | 1,1 s |

A restauração do formato personalizado num banco vazio levou 1,8 s, e a contagem
conferiu 500 mil linhas. Os tempos e a compressão variam com a máquina e com o
conteúdo. Neste exemplo, o formato personalizado ocupou perto de um sétimo do
texto puro.

### Senha sem digitar

Um backup automático não pode parar pedindo senha. O PostgreSQL lê a senha do
arquivo `~/.pgpass`, uma linha por conexão, no formato
`servidor:porta:banco:usuario:senha`:

```
localhost:5432:loja:app:senha-do-banco
```

O arquivo precisa estar em modo 600. Com permissão mais aberta, o cliente ignora
o arquivo e volta a pedir a senha.

```bash
chmod 600 ~/.pgpass
```

### O que o pg_dump não copia

O `pg_dump` copia um banco. Usuários (*roles*) e permissões globais ficam fora,
porque pertencem ao cluster. Eles se copiam com:

```bash
pg_dumpall --globals-only -f globais.sql
```

## Tipos de backup pela quantidade copiada

Um **backup completo** copia tudo a cada execução. É o mais simples de restaurar,
porque um único arquivo basta.

Um **backup incremental** copia só o que mudou desde o backup anterior, de
qualquer tipo. Ocupa pouco espaço, mas a restauração precisa do último completo e
de todos os incrementais depois dele, em ordem. Se um elo da corrente estiver
estragado, os seguintes não servem.

Um **backup diferencial** copia o que mudou desde o último completo. Cresce ao
longo da semana, e a restauração precisa só do último completo e do último
diferencial.

| Restaurar a quinta-feira | Arquivos necessários |
|---|---|
| completo todo dia | o de quinta |
| completo no domingo, incremental nos outros dias | domingo, segunda, terça, quarta e quinta |
| completo no domingo, diferencial nos outros dias | domingo e quinta |

O `pg_dump` sempre gera um backup completo. Ferramentas como o `restic` e o
`borg` guardam os arquivos em blocos e só enviam os blocos novos, o que dá o
espaço de um incremental com a restauração de um completo.

## Retenção

Retenção é a regra que decide quantos backups ficam guardados e por quanto
tempo. Sem ela, o disco enche. Com retenção curta demais, o problema que
apareceu há três dias já não tem cópia limpa.

O esquema mais usado guarda muitos backups recentes e poucos antigos. Um exemplo:
os 24 últimos backups de hora em hora, um por dia nos últimos 7 dias e um por
semana nas últimas 4 semanas. Ele se chama avô, pai e filho (GFS, em inglês).

Num script, a retenção mais simples apaga os arquivos mais velhos que um número
de dias:

```bash
find /backup -name 'loja-*.dump' -mtime +7 -delete
```

Com o `restic`, a mesma regra do exemplo acima fica numa linha:

```bash
restic forget --keep-hourly 24 --keep-daily 7 --keep-weekly 4 --prune
```

## A regra 3-2-1 e a cópia imutável

A regra 3-2-1 resume onde guardar:

- **3** cópias dos dados, contando a original;
- **2** tipos de armazenamento diferentes;
- **1** cópia fora do lugar onde está o servidor.

A versão atual, 3-2-1-1-0, acrescenta:

- **1** cópia imutável, que ninguém consegue alterar nem apagar durante um prazo;
- **0** erros na verificação, o que significa testar a restauração.

A cópia imutável existe por causa do sequestro de dados. Um programa que cifra os
arquivos do servidor alcança tudo o que o servidor alcança: o banco, o backup na
mesma máquina e o backup remoto, se a credencial para apagar o remoto estiver
guardada no servidor. O programa apaga ou cifra qualquer backup que o servidor
comprometido tenha permissão de alterar.

A **trava de retenção no armazenamento** bloqueia a exclusão e a sobrescrita de
um objeto até uma data. Na Cloudflare R2 isso se chama *bucket lock*; na
Backblaze B2 e na AWS S3, *Object Lock*. Na B2, o Object Lock é ligado na criação
do bucket e não pode ser desligado depois.

A **credencial sem permissão de apagar** deixa o servidor enviar arquivos novos e
nada mais. Na Backblaze B2, uma chave de aplicação pode ser criada com a
permissão `writeFiles` e sem `deleteFiles`.

O **repositório só de acréscimo** aceita dados novos e recusa exclusões. No
`borg`, isso se configura do lado do servidor de backup, prendendo a chave SSH a
um comando:

```
command="borg serve --append-only --restrict-to-path /backup/repo" ssh-ed25519 AAAA...
```

Essas proteções valem contra um cliente comprometido. Quem obtém acesso de
administrador ao próprio armazenamento ainda consegue apagar.

## Onde guardar fora do servidor

Vários provedores de armazenamento de objetos oferecem uma faixa gratuita que
cabe nos backups de uma atividade. Os dois primeiros abaixo foram conferidos nas
páginas oficiais em 16/11/2026; limites e condições mudam, então confira a página
do provedor antes de criar a conta.

| Provedor | Faixa gratuita | Trava de retenção |
|---|---|---|
| Cloudflare R2 | 10 GB por mês, sem cobrança de saída de dados | sim, *bucket lock* |
| Backblaze B2 | os primeiros 10 GB | sim, Object Lock, ligado na criação do bucket |
| IDrive e2 | camada gratuita anunciada; confira o limite | confira |
| Oracle Cloud (Always Free) | armazenamento de objetos incluso; confira o limite | confira |

Todos falam o protocolo da AWS S3, então as mesmas ferramentas servem para
qualquer um: `restic`, `rclone` ou o `aws` com o endereço do provedor.

Google Drive, Dropbox e OneDrive têm espaço gratuito maior, mas nenhum oferece
trava de retenção, e a automação depende de um fluxo de autorização pensado para
pessoas. O `rclone` funciona com eles; a cópia fica sujeita a ser apagada pela
mesma credencial que a enviou.

## Ferramentas para enviar o backup

### rclone

O `rclone` copia arquivos entre o disco local e dezenas de serviços. Depois de
configurar o destino com `rclone config`, o envio de um dump é uma linha:

```bash
rclone copy /backup/loja-2026-11-19-1400.dump r2:meu-bucket/loja/
```

Para a Cloudflare R2, o endereço do serviço é
`https://<ID-DA-CONTA>.r2.cloudflarestorage.com` e a região é `auto`.

### restic

O `restic` guarda os backups num repositório cifrado, sem repetir os blocos que
já foram enviados, e controla a retenção. O repositório pode ficar num disco
local, num servidor por SFTP ou num armazenamento S3.

```bash
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export RESTIC_REPOSITORY=s3:https://<ID-DA-CONTA>.r2.cloudflarestorage.com/meu-bucket
export RESTIC_PASSWORD_FILE=~/.restic-senha

restic init                                   # uma vez só
restic backup --stdin-from-command \
  --stdin-filename loja.dump \
  -- pg_dump -h localhost -U app -Fc -d loja  # a cada execução
restic snapshots                              # lista o que existe
restic dump latest loja.dump > loja.dump      # recupera o último
restic check                                  # confere a integridade
```

A opção `--stdin-from-command` executa o `pg_dump` e confere o código de saída
dele. Ela existe a partir do restic 0.17; o pacote do Debian bookworm é mais
antigo, e a versão nova se baixa da página de *releases* do projeto. A forma com pipe, `pg_dump ... | restic backup --stdin`, salva o que chegou
mesmo quando o `pg_dump` falha no meio, e o snapshot fica incompleto sem aviso.

A senha do repositório cifra todos os backups. Sem ela, os dados não voltam, nem
pelo provedor. Guarde uma cópia da senha fora do servidor.

O `restic` cria e apaga arquivos de trava no repositório a cada execução. Com a
trava de retenção aplicada ao bucket inteiro, essa exclusão falha. Aplique a
regra só ao prefixo dos dados ou use a combinação de dump com `rclone`, em que cada
backup é um objeto novo que nunca é sobrescrito.

## Automatizar

A execução periódica fica com o `cron` ou com um *timer* do `systemd`, para o
backup não depender de alguém lembrar.

Um script de backup completo, com data no nome, retenção e envio:

```bash
#!/usr/bin/env bash
set -euo pipefail

DESTINO=/home/aluno/backup
ARQUIVO="$DESTINO/loja-$(date +%Y-%m-%d-%H%M).dump"

mkdir -p "$DESTINO"
pg_dump -h localhost -U app -Fc -d loja -f "$ARQUIVO"
pg_restore --list "$ARQUIVO" > /dev/null      # o arquivo abre
rclone copy "$ARQUIVO" r2:meu-bucket/loja/
find "$DESTINO" -name 'loja-*.dump' -mtime +2 -delete
echo "$(date -Is) ok $ARQUIVO"
```

O `set -euo pipefail` faz o script parar no primeiro erro, inclusive dentro de um
pipe. Sem ele, um `pg_dump` que falha deixa um arquivo vazio, e o script segue e
envia o arquivo vazio.

A linha do `cron` que roda o script a cada 15 minutos e guarda a saída:

```
*/15 * * * * /home/aluno/backup.sh >> /home/aluno/backup.log 2>&1
```

O `cron` executa com um ambiente mínimo, sem as variáveis do seu terminal. Use
caminhos absolutos e defina no próprio script o que ele precisa.

Sem registro, um backup que falha só é descoberto na hora da restauração. O log
com uma linha por execução, com data e resultado, é o mínimo. O passo
seguinte é um aviso quando a execução não acontece, por exemplo um monitor que
espera uma chamada a cada intervalo e alerta quando ela não chega.

## Testar a restauração

Um backup só está provado depois de restaurado. O teste roda num banco separado,
nunca por cima do banco em produção:

```bash
createdb -h localhost -U app loja_teste
pg_restore -h localhost -U app -d loja_teste loja-2026-11-19-1400.dump
psql -h localhost -U app -d loja_teste -c "select count(*), max(criado_em) from pedidos"
dropdb -h localhost -U app loja_teste
```

Na consulta final, a contagem mostra que os dados vieram, e a data mais recente
mostra até onde o backup chega, que é o RPO real daquela cópia.

O tempo entre o começo e o fim desses comandos, somado ao tempo de baixar o
arquivo, é uma medida do RTO. Anote.

## Recuperação a um instante qualquer

Com `pg_dump`, a restauração volta ao instante do último dump. Para voltar a um
instante qualquer, por exemplo um minuto antes de um `DELETE` errado, o
PostgreSQL combina um backup físico com o arquivamento do WAL.

### O que é o WAL

WAL é a sigla de *write-ahead log*, o registro gravado antes. Toda alteração no
banco, seja um `INSERT`, um `UPDATE`, um `DELETE` ou um `CREATE TABLE`, é escrita
primeiro nesse registro, e só depois nas páginas das tabelas. Se o servidor cair
no meio do caminho, o PostgreSQL relê o WAL na partida e refaz o que faltava,
deixando as tabelas consistentes.

O WAL fica no diretório `pg_wal`, dentro do diretório de dados, dividido em
arquivos chamados segmentos, de 16 MB cada um no padrão. Depois que as alterações
de um segmento estão gravadas nas tabelas, o servidor reaproveita ou apaga aquele
segmento. Sem arquivamento, o histórico de alterações se perde à medida que o
banco trabalha.

### O que é o arquivamento de WAL

Arquivar o WAL é copiar cada segmento para um lugar fora do servidor assim que ele
fica completo, antes que o servidor o reaproveite. O resultado é uma sequência
contínua de todas as alterações feitas no banco desde que o arquivamento começou.

Com um backup físico de base e todos os segmentos arquivados depois dele, o banco
pode ser reconstruído em qualquer instante desse intervalo. A restauração parte da
base e reaplica as alterações, em ordem, até o instante escolhido. Um exemplo:
base tirada no domingo às 2h, segmentos arquivados até quinta-feira, e um `DELETE`
errado na quinta às 14h03. A restauração parte da base de domingo e reaplica o WAL
até as 14h02.

A configuração do servidor liga o arquivamento:

```
archive_mode = on
archive_command = 'test ! -f /arquivo/%f && cp %p /arquivo/%f'
archive_timeout = 60
```

Para cada segmento completo, o servidor executa o `archive_command`, trocando `%p`
pelo caminho do segmento e `%f` pelo nome do arquivo. O segmento só é liberado
quando o comando termina com código 0. Se o comando falhar, os segmentos se
acumulam no `pg_wal` até o disco do servidor encher, então o arquivamento quebrado
derruba o banco se ninguém olhar.

Um segmento só é arquivado quando completa 16 MB, o que num banco com pouca escrita
pode levar horas. O `archive_timeout` força a troca de segmento a cada tantos
segundos. Com 60, o RPO fica em cerca de um minuto, e cada troca gera um
segmento inteiro de 16 MB no arquivo, mesmo com pouca alteração dentro dele.

### Backup de base e restauração

O backup físico de base sai do `pg_basebackup`:

```bash
pg_basebackup -h localhost -U replicador -D /backup/base -Ft -z -X stream -P
```

Na restauração, o diretório de dados volta a partir da base, o arquivo
`recovery.signal` é criado nele, e a configuração diz de onde ler o WAL e até
quando aplicar:

```
restore_command = 'cp /arquivo/%f %p'
recovery_target_time = '2026-11-19 14:02:00'
```

A visão `pg_stat_archiver` mostra se o arquivamento está funcionando e quando
falhou pela última vez. Ferramentas como o pgBackRest, o WAL-G e o Barman
cuidam dessas etapas e da retenção do WAL.

## Erros comuns

- Guardar o backup só na mesma máquina do banco.
- Guardar no servidor a credencial que apaga o backup remoto.
- Nunca testar a restauração.
- Usar pipe sem `pipefail`, o que transforma um dump com erro num arquivo vazio
  com cara de sucesso.
- Copiar a pasta de dados com o banco rodando.
- Não ter retenção, e descobrir pelo disco cheio.
- Perder a senha do repositório cifrado, o que torna todos os backups
  irrecuperáveis.
- Esquecer usuários e permissões, que o `pg_dump` não copia.

## Fontes

- [Documentação do PostgreSQL, capítulo "Backup and Restore"](https://www.postgresql.org/docs/current/backup.html)
- [Documentação do restic](https://restic.readthedocs.io/)
- [Documentação do rclone, Cloudflare R2](https://rclone.org/s3/#cloudflare-r2)
- [Cloudflare R2, preços e faixa gratuita](https://developers.cloudflare.com/r2/pricing/)
- [Cloudflare R2, bucket locks](https://developers.cloudflare.com/r2/buckets/bucket-locks/)
- [Backblaze B2, preços](https://www.backblaze.com/cloud-storage/pricing)
- [Backblaze B2, Object Lock](https://www.backblaze.com/docs/cloud-storage-object-lock)
- [ComputingForGeeks, backups imutáveis com restic e borg](https://computingforgeeks.com/immutable-backups-ransomware-proof-linux-minio-restic-borg/)
