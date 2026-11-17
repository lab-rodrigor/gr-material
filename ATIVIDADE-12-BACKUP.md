# Atividade 12: backup do banco de dados

- Prazo: sexta-feira, 20/11, às 20h.
- Entrega: este repositório, com commit e push até o prazo.
- Material de apoio: [Backup de banco de dados](https://github.com/lab-rodrigor/gr-material/blob/main/BACKUP.md).

## O cenário

Você tem um PostgreSQL 15 só seu, o container `gr-<seu slug>-dados`, na rede do
laboratório. O seu slug é o começo do seu e-mail com o ponto trocado por hífen:
`maria.silva` vira `gr-maria-silva-dados`. Servidor, porta, banco, usuário e senha
estão no arquivo `/home/aluno/banco.txt` do seu laboratório.

Desde segunda-feira, 16/11, às 22h, um programa do servidor insere uma linha na
tabela `registros` do banco `escriba`, com intervalo sorteado de 1 a 10 minutos
entre uma linha e a próxima. Cada linha tem um número de sequência e o hash da
linha anterior.

Na quinta-feira, 19/11, entre 8h e 20h, num minuto sorteado para você, o banco é
sequestrado. O conteúdo de `registros` é cifrado, a tabela é renomeada para
`registros_sequestrados`, e aparece uma tabela `resgate` com um bilhete. No mesmo
minuto, toda cópia de banco guardada no seu laboratório também é cifrada e o
original é apagado: arquivos `.dump`, `.pgdump`, `.backup`, `.sql`, `.sql.gz`,
`.tar`, `.tgz` e semelhantes, e repositórios locais do restic. A chave não é
entregue. A saída é o seu backup.

Depois do ataque, o programa continua tentando inserir na tabela `registros`. Ele
volta a conseguir quando você restaurar a tabela.

## O que fazer

1. Monte um backup automático do banco `escriba`, que rode sozinho em intervalo
   fixo e guarde mais de uma geração, com regra para remover as antigas.
2. Envie cada backup para um armazenamento fora do servidor da disciplina.
3. Teste a restauração antes de quinta-feira, num banco ou tabela separada.
4. Depois do ataque, restaure a tabela `registros` com o conteúdo original.
5. Registre tudo neste repositório.

Não altere a estrutura da tabela `registros` e não apague linhas dela. O programa
que insere depende dela.

## O que fica no repositório

- `backup.sh`: o script de backup, ou o arquivo de configuração da ferramenta que
  você usou.
- `CRON.txt`: a linha do `cron` que executa o backup.
- `backup.log`: o log das execuções, com uma linha por execução, com data e
  resultado. Faça commit dele antes do prazo.
- `RESTAURACAO.md`: onde o backup fica, a retenção, o procedimento de restauração
  e o que aconteceu no ataque.

Senha do banco, chave do provedor e senha do restic ficam fora do git. O
`.gitignore` deste repositório já exclui os nomes mais comuns; confira antes de
cada commit.

## O que eu verifico

- O backup roda sozinho: o `backup.log` mostra execuções periódicas, com data e
  resultado, desde antes do ataque.
- Existe retenção: o script guarda mais de uma geração e remove as antigas por
  regra.
- A cópia sai do servidor: o script envia para um armazenamento externo, e o
  `RESTAURACAO.md` diz qual.
- A tabela `registros` voltou, e cada linha restaurada confere com a que o
  programa inseriu, em conteúdo e em hash.
- Perda de dados (RPO): quantas linhas inseridas antes do ataque ficaram de fora
  da restauração.
- Tempo de recuperação (RTO): o tempo entre o ataque e a primeira inserção que
  volta a funcionar.
- Nenhum segredo no repositório.

## Armazenamento fora do servidor

Escolha um provedor de armazenamento de objetos compatível com S3. Os dois
primeiros foram conferidos nas páginas oficiais em 16/11/2026; confira o limite
atual antes de criar a conta.

| Provedor | Faixa gratuita | Trava de retenção |
|---|---|---|
| Cloudflare R2 | 10 GB por mês, sem cobrança de saída | *bucket lock* |
| Backblaze B2 | os primeiros 10 GB | Object Lock, ligado na criação do bucket |
| IDrive e2 | camada gratuita anunciada; confira o limite | confira |
| Oracle Cloud (Always Free) | armazenamento de objetos incluso; confira o limite | confira |

A trava de retenção impede que alguém apague ou sobrescreva o backup durante um
prazo, inclusive quem obtiver a credencial que está no seu laboratório. O texto de
apoio explica como ela funciona e quais ferramentas usar.

## O laboratório

O laboratório só preserva o que está em `/home/aluno`. Pacotes instalados e
serviços iniciados somem se o laboratório for recriado.

- Cliente do PostgreSQL: `sudo apt install postgresql-client`. Instala a versão
  15, igual à do servidor. O `pg_dump` recusa servidor mais novo que ele.
- Agendador: `sudo apt install cron` e `sudo service cron start`. O laboratório
  não tem systemd.
- rclone: `sudo apt install rclone`. A versão do Debian já tem a Cloudflare R2.
- restic: a versão do `apt` é a 0.14, sem a opção `--stdin-from-command`. A
  versão nova se baixa da página de *releases* do projeto no GitHub.
- Teste a conexão: `psql -h gr-<seu slug>-dados -U aluno -d escriba`.
