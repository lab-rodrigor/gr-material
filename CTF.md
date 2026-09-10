# CTF — Preparando um servidor do zero

**Gerência de Redes 2026.2 · DCX/UFPB**

Você recebeu um servidor. Ele está no ar, acessível pela internet, e **ninguém o
preparou**: root entra com senha, há contas que ninguém explica, serviços
escutando à toa e nenhum firewall.

Seu trabalho é deixá-lo pronto para uso. São **12 missões**, e o servidor é
verificado de verdade — não há flag escondida para caçar. Cada missão é um
**estado do sistema** que ou está correto, ou não está.

---

## Antes de começar

### Entre no seu laboratório

Sua chave e a sua porta estão na **sua página**: <https://gr.lab.ayty.org>

```bash
chmod 600 ~/Downloads/sua-chave      # o SSH recusa chave com permissão frouxa
ssh -i ~/Downloads/sua-chave -p SUA-PORTA aluno@gr.lab.ayty.org
```

Você entra como **`aluno`**, com `sudo` liberado. **Não** entra como root — e
uma das missões é justamente garantir que ninguém mais entre.

### Regras de sobrevivência

**Nunca feche a sessão que funciona.** Ao mexer em SSH, firewall ou rede, deixe
esta janela aberta e teste a mudança **numa segunda janela**. Se errar, você
volta pela primeira. Fechar a única sessão antes de testar é como a maioria das
pessoas se tranca fora de um servidor — inclusive gente experiente.

**O que está em `/home/aluno` sobrevive** a uma recriação do laboratório. O
resto, não. Se destruir tudo, peça a recriação no Discord: leva segundos, e
recomeçar faz parte.

**Você não está sozinho na rede.** Os laboratórios da turma estão em
`172.30.0.0/16` e respondem. Isso é de propósito — a missão 12 depende disso.

### Acompanhe pelo bot

```
/ctf
```

Mostra quais missões já contam como cumpridas, o que ainda falta e **por quê** —
com a evidência que o servidor devolveu. O botão *Explicar a missão* traz o texto
completo daquela missão, o mesmo que está aqui.

O `/ctf` não guarda pontuação: ele pergunta ao seu servidor na hora. Se você
quebrar algo depois, a missão volta a ficar aberta. É honesto assim.

---

## As missões

### 1. Desligar os serviços que ninguém pediu

**O que é.** Um servidor recém-instalado quase sempre traz serviços ligados que ninguém pediu. Neste aqui há dois: o **rpcbind** (porta 111), peça antiga do NFS, e o **vsftpd** (porta 21), um servidor de FTP.

**Por que importa.** Cada porta aberta é uma porta que alguém pode bater. E não é figura de linguagem: um servidor novo na internet começa a receber varredura automática em minutos. O FTP é especialmente ruim porque, na configuração padrão, senha e arquivos viajam **em texto puro** — quem estiver no caminho lê tudo. O rpcbind já foi usado em ataques de amplificação, em que o seu servidor vira ferramenta para atacar terceiros.

**Investigue.** `ss -lntup` lista o que está escutando, em qual porta, e qual processo é dono. As letras importam: `l` de listening, `n` para não resolver nomes, `t` e `u` para TCP e UDP, `p` para mostrar o processo.

Compare com `ps aux`. **Rodar e estar exposto são coisas diferentes** — muita coisa roda sem abrir porta, e é justamente o que abre porta que precisa da sua atenção. Confundir os dois é como se deixa um serviço acessível sem perceber.

**Faça.** Pare o serviço (`systemctl stop`, ou `service` aqui) e **remova o pacote** (`apt purge`). Parar sem remover é conserto que dura até o próximo boot.

**Cuidado.** Antes de remover qualquer coisa, pergunte-se o que depende dela. Aqui nada depende — mas o hábito de perguntar é o que evita derrubar produção.

> **Como eu verifico:** `ss -lntup` mostra quem escuta. Pare e remova o que não for usado.

### 2. Bloquear o login de root

**O que é.** O SSH está aceitando login direto como **root**, com senha.

**Por que importa.** Quem ataca por força bruta precisa acertar duas coisas: usuário e senha. Com root habilitado, metade do trabalho já está feita — o nome é o mesmo em todo servidor Linux do mundo. É por isso que `root` é o usuário mais tentado em qualquer log de servidor exposto.

Há um segundo motivo, menos óbvio: **rastreabilidade**. Se cinco pessoas entram como root, o log não diz quem fez o quê. Entrando cada uma com a sua conta e usando `sudo`, cada ação fica registrada com nome.

**Faça.** Em `/etc/ssh/sshd_config`, ajuste `PermitRootLogin no` e recarregue o sshd. Você continua entrando como `aluno` e escalando com `sudo` quando precisar — que é como se administra uma máquina de verdade.

**Verifique com `sshd -T`**, não com o arquivo. O `-T` mostra a configuração **efetiva**, já resolvendo includes e valores padrão. É comum alguém editar uma linha e não perceber que outra, mais abaixo, sobrescreve.

> **Como eu verifico:** `PermitRootLogin no` no sshd_config, e recarregue o sshd.

### 3. Desabilitar autenticação por senha

**O que é.** O servidor aceita autenticação por senha.

**Por que importa.** Senha é adivinhável; chave, não. Uma chave ed25519 tem espaço de busca astronômico — ninguém a descobre por tentativa. Uma senha de oito caracteres cai em horas, e a maioria das pessoas reusa a mesma em vários lugares, então basta um vazamento em outro serviço para o seu servidor cair junto.

Desligar a senha muda a natureza do problema: o atacante deixa de ter algo a adivinhar. É a única mudança desta lista que **elimina** uma classe inteira de ataque, em vez de dificultá-la.

**Cuidado com a ordem — esta é a missão em que mais gente se tranca fora.** Confirme que a sua chave funciona **antes** de desligar a senha. O jeito certo:

1. deixe esta sessão aberta, sem fechar
2. abra uma **segunda janela** e conecte com a chave
3. só depois de a segunda funcionar, aplique a mudança

Se algo der errado, você volta pela primeira janela. Fechar a única sessão antes de testar é como profissionais experientes perdem servidores.

**Faça.** `PasswordAuthentication no` e recarregue o sshd.

> **Como eu verifico:** `PasswordAuthentication no`. Confirme que sua chave funciona ANTES.

### 4. Deixar os pacotes atualizados

**O que é.** O sistema tem pacotes desatualizados.

**Por que importa.** A maioria das invasões não usa falha nova e secreta: usa falha **conhecida, publicada e corrigida** há meses, num servidor que ninguém atualizou. Quando uma vulnerabilidade é divulgada, a correção costuma sair junto — e o que separa quem foi invadido de quem não foi é ter aplicado.

**Entenda a diferença.** `apt update` **não atualiza nada**: ele só baixa a lista do que existe nos repositórios. Quem instala é o `apt upgrade`. Rodar só o primeiro e achar que atualizou é um mal-entendido comum.

**Faça.** `sudo apt update && sudo apt upgrade`.

**Repare** na saída: o apt informa quantos pacotes serão atualizados e o quanto vai baixar. Ler isso é parte do trabalho — um dia vai aparecer um pacote que você não esperava, e é aí que a leitura vale.

Isto resolve hoje. Manter atualizado amanhã é a missão 9.

> **Como eu verifico:** `apt update && apt upgrade`. Atualizar uma vez não basta — veja a missão 9.

### 5. Ligar o firewall liberando só o necessário

**O que é.** O servidor não tem firewall: tudo o que escuta está acessível.

**Por que importa.** Firewall não serve para 'bloquear ataque' — serve para **reduzir a superfície**. O que não está aberto não precisa ser defendido, monitorado nem corrigido às pressas. É a diferença entre proteger uma casa com dez portas e proteger uma com duas.

Ele também protege do que você não sabe que instalou: um pacote que sobe um serviço numa porta alta fica inacessível de fora, mesmo que você nunca perceba que ele existe.

**Faça.** Instale o `ufw` e configure nesta ordem:

1. **primeiro** permita o SSH — a sua porta, não necessariamente a 22
2. permita o HTTP, que a missão 7 vai usar
3. negue o resto por padrão
4. **por último**, ative

**Cuidado.** Ativar antes de liberar o SSH derruba a sua própria conexão na hora. O ufw até avisa; leia o aviso em vez de confirmar no automático.

**Verifique** com `sudo ufw status verbose` — ele mostra a política padrão e cada regra.

> **Como eu verifico:** Instale o ufw, permita SSH e HTTP, negue o resto, e só então ative.

### 6. Pôr o fail2ban de pé

**O que é.** Não há nada limitando tentativas de login.

**Por que importa.** Mesmo com senha desligada, o servidor continua recebendo tentativas — e cada uma consome CPU, enche o log e esconde o que interessa. O **fail2ban** lê o log de autenticação, detecta repetição vinda do mesmo endereço e o bane no firewall por um tempo.

O efeito não é impedir a invasão (a missão 3 já fez isso), e sim **encarecer a tentativa** e limpar o seu log. Servidor com log limpo é servidor onde você enxerga o evento estranho quando ele aparece.

**Como ele funciona.** Três peças: um *filter* (a expressão que reconhece falha no log), uma *jail* (qual log ler e qual filtro usar) e uma *action* (o que fazer — em geral, regra de firewall). Vale abrir `/etc/fail2ban/jail.conf` e ver os parâmetros `maxretry`, `findtime` e `bantime`.

**Faça.** Instale o `fail2ban`, garanta que ele está rodando e confirme com `sudo fail2ban-client status sshd`.

**Se a jail não aparecer**, olhe se `/var/log/auth.log` existe. Sem log de autenticação, não há o que ler — e o fail2ban falha justamente aí.

> **Como eu verifico:** Instale o fail2ban e confirme com `fail2ban-client status sshd`.

### 7. Subir o nginx com uma página sua

**O que é.** O servidor não serve nada ainda.

**Por que importa.** Até agora você fechou portas. Esta missão faz o caminho inverso: **abre uma de propósito**, com controle. É o que diferencia um servidor trancado de um servidor útil — segurança existe para permitir o uso, não para impedi-lo.

**Por que trocar a página padrão.** A tela 'Welcome to nginx' informa a quem passa qual software você roda. Isso é reconhecimento gratuito para quem procura alvo: sabendo o servidor, procura-se a falha conhecida daquela versão. Também é sinal de servidor abandonado — e servidor abandonado é alvo preferido.

**Faça.** Instale o nginx, garanta que ele responde, e substitua o `/var/www/html/index.html` por uma página **sua**, com o seu nome.

**Verifique** de dentro com `curl -I localhost` (o `-I` traz só os cabeçalhos) e repare no header `Server:`. Depois abra do seu navegador, pela porta que o firewall liberou.

**Se não abrir de fora** e abrir de dentro, o problema é firewall — e você acaba de aprender a diagnosticar pela diferença entre os dois testes.

> **Como eu verifico:** Depois de instalar, troque o index.html pela sua página.

### 8. Mudar a porta do SSH

**O que é.** O SSH está na porta 22, onde todo mundo procura.

**Por que importa — e o que isto NÃO é.** Mudar a porta **não é segurança de verdade**: quem te ataca de propósito roda um scan e acha em segundos. É *segurança por obscuridade*, e vale conhecer o termo para saber que não se constrói defesa em cima disso.

O ganho é outro, e é real: **99% do que bate na porta 22 é varredura automática**. Tirando o SSH de lá, o seu log de autenticação passa de milhares de linhas de ruído para quase nada. E aí, quando aparecer uma tentativa, ela **significa alguma coisa**.

**ATENÇÃO — porta de dentro e porta de fora são diferentes.** O seu laboratório roda num container, e o servidor traduz portas na entrada:

```
de fora            dentro do laboratório
SUA_PORTA     ->   22
SUA_PORTA+100 ->   2222
```

Se a sua porta de acesso é a 22018, então **2222 lá dentro aparece como 22118 aqui fora**. Conectar em `gr.lab.ayty.org:2222` dá timeout, porque não há nada escutando nessa porta no host — e o `tcpdump` no laboratório não vê pacote nenhum, que é o sintoma exato.

É a mesma confusão da missão 5: o firewall de dentro enxerga a porta de dentro.

**Faça.** Ponha o sshd na **2222** e libere a **2222** no firewall.

**A ordem é tudo aqui:**

1. libere a 2222 no firewall **antes**
2. mude o `Port` no sshd_config e recarregue
3. teste **numa segunda janela**, em `SUA_PORTA+100`
4. só então feche a antiga

Inverter os passos 1 e 2 é a receita clássica para perder o acesso.

> **Como eu verifico:** Use a 2222 lá dentro. De fora ela aparece na SUA porta + 100 — 22018 vira 22118. Teste ANTES de fechar a antiga.

### 9. Ligar as atualizações automáticas

**O que é.** Nada mantém o sistema atualizado sozinho.

**Por que importa.** A missão 4 resolveu o hoje. Mas a janela perigosa é a que vai da publicação de uma correção até você aplicá-la — e essa janela é medida em dias ou semanas, quando depende de alguém lembrar. Servidor real fica meses sem ninguém olhar.

O `unattended-upgrades` fecha essa janela: ele aplica as atualizações de **segurança** automaticamente, sem intervenção.

**A objeção legítima:** atualizar sozinho pode quebrar algo de madrugada. Por isso o padrão aplica só o repositório de segurança, e não toda atualização — é um equilíbrio entre risco de quebrar e risco de ficar vulnerável. Saber que essa escolha existe é parte da profissão.

**Faça.** Instale o `unattended-upgrades` e habilite em `/etc/apt/apt.conf.d/20auto-upgrades`, com `APT::Periodic::Unattended-Upgrade "1";`.

**Verifique** com `sudo unattended-upgrade --dry-run --debug`.

> **Como eu verifico:** `unattended-upgrades` instalado e habilitado em 20auto-upgrades.

### 10. Endurecer o sshd

**O que é.** O sshd está com os padrões, que são permissivos por conveniência.

**Por que cada um importa:**

**`MaxAuthTries`** — quantas tentativas por conexão. O padrão de 6 (aqui está 10) deixa o atacante testar várias senhas sem reconectar. Com 3, cada tentativa custa uma conexão nova — mais lento para ele, mais visível no log para você.

**`LoginGraceTime`** — quanto tempo a conexão fica aberta antes de autenticar. O padrão de 120 s permite segurar muitas conexões pela metade e esgotar os slots do servidor, um jeito barato de negar serviço. 30 s basta para gente de verdade.

**`AllowTcpForwarding`** — permite usar o seu servidor como **túnel** para alcançar outras máquinas. Se uma conta for comprometida, o atacante entra na sua rede interna por ela. Você não usa isso aqui; desligue.

**Faça.** `MaxAuthTries 3`, `LoginGraceTime 30`, `AllowTcpForwarding no`.

**Verifique com `sshd -T`.** Ele mostra a configuração **efetiva** — não o que você escreveu, e sim o que o serviço está usando. A diferença entre os dois já custou muita noite de depuração.

> **Como eu verifico:** MaxAuthTries ≤ 3, LoginGraceTime ≤ 60, AllowTcpForwarding no.

### 11. Encontrar e remover a conta indevida

**O que é.** Existe uma conta `suporte` neste servidor, com senha e com `sudo` irrestrito. Você não a criou e ninguém sabe explicar de onde veio.

**Por que importa.** É assim que acesso indevido sobrevive de verdade: não por invasão espetacular, e sim por uma conta de um fornecedor antigo, de um estagiário que saiu, de uma migração feita às pressas. Ela fica lá, com senha fraca e permissão total, por anos.

Auditar contas é das tarefas mais subestimadas da administração de sistemas — e das que mais encontram coisa.

**Investigue:**

- `cat /etc/passwd` — todas as contas; repare em quais têm shell de verdade
- `ls -la /etc/sudoers.d/` — permissões concedidas fora do arquivo principal
- `sudo lastlog` — quando cada conta entrou pela última vez
- `awk -F: '$3 >= 1000' /etc/passwd` — só as contas humanas

**Faça.** Remova a conta **e** o arquivo de sudo dela.

**A pegadinha:** apagar o usuário e esquecer o `/etc/sudoers.d/` deixa a permissão órfã — e ela volta a valer no dia em que alguém criar uma conta com o mesmo nome. Limpe os dois.

> **Como eu verifico:** Alguém deixou uma conta com sudo aqui. Ache-a — e apague o sudoers dela também.

### 12. Provar que o fail2ban bane de verdade

**O que é.** Você instalou o fail2ban. Ele funciona mesmo?

**Por que importa.** Esta é a missão mais importante da lista, e a que quase ninguém faz na vida real: **segurança que não foi testada é suposição**. Configurar e seguir em frente é como instalar um alarme sem nunca disparar — você descobre que ele estava mudo no dia em que precisava.

**Faça.** Provoque tentativas de login falhas contra o seu SSH, vindas **de outra máquina**. O laboratório dos seus colegas está na mesma rede (`172.30.0.0/16`) e enxerga o seu — é para isso que a rede é compartilhada. Combine com alguém, ou use um colega como origem.

Repita até estourar o `maxretry`, depois confira:

```
sudo fail2ban-client status sshd
```

Você quer ver o contador de **Total banned** subir.

**Repare no efeito colateral:** se você banir a si mesmo, vai precisar esperar ou remover o ban (`fail2ban-client set sshd unbanip SEU-IP`). Isso também é aprendizado — proteção que funciona atrapalha o dono distraído.

> **Como eu verifico:** Erre a senha algumas vezes de propósito, de outra máquina, e veja o ban.

---

## Ordem sugerida

Você pode fazer na ordem que quiser, mas esta reduz a chance de se trancar fora:

1. **Olhe antes de mexer** (missão 1) — `ss -lntup`, `ps aux`, `cat /etc/passwd`
2. **Limpe o que sobra** (missões 1 e 11) — serviços e contas indevidas
3. **Feche o acesso** (2, 3, 10) — sempre testando numa segunda janela
4. **Atualize** (4 e 9)
5. **Proteja** (5 e 6) — firewall e fail2ban
6. **Sirva** (7) — o nginx com a sua página
7. **Mude a porta** (8) — depois que tudo o mais estiver estável
8. **Prove** (12) — o fail2ban banindo de verdade

A 8 fica perto do fim de propósito: mudar a porta com o firewall já configurado
exige lembrar de liberar a porta nova. Se você fizer fora de ordem, vai descobrir
isso do jeito difícil — o que também ensina.

## Quando terminar

Rode `/ctf` uma última vez. Com 12/12, o servidor está pronto.

Vale mais que a pontuação: você consegue **explicar** cada mudança que fez?
A pergunta na avaliação não vai ser "quais comandos você rodou", e sim "por que
esse servidor está mais seguro do que quando chegou".

---

<sub>Dúvidas no canal da atividade. O `/ctf` verifica o seu servidor de verdade —
se ele discorda de você, ele está certo: olhe a evidência que ele mostra.</sub>
