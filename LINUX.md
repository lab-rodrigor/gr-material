# Linux pelo terminal — consulta rápida

**Gerência de Redes 2026.2 · DCX/UFPB**

Este documento é para consultar **enquanto você faz o CTF**. Não precisa ler do
começo ao fim: ache a sua pergunta abaixo e vá direto.

Tudo aqui é para rodar no **seu laboratório**. Ler comando sem executar não
ensina nada — abra o terminal antes de continuar.

---

## Achei um erro / não sei o que fazer

| Sua pergunta | Vá para |
|---|---|
| Onde eu estou? Como ando pelas pastas? | [Navegar](#navegar) |
| Como vejo o conteúdo de um arquivo? | [Ler arquivos](#ler-arquivos) |
| Como edito um arquivo de configuração? | [Editar](#editar-arquivos-de-configuração) |
| Quem sou eu? Por que "permission denied"? | [Usuários e permissões](#usuários-e-permissões) |
| O que está rodando nesta máquina? | [Processos](#processos) |
| Como ligo/desligo um serviço? | [Serviços](#serviços) |
| O que está escutando na rede? | [Rede](#rede) |
| Como instalo/removo um programa? | [Pacotes](#pacotes) |
| Onde estão os logs? | [Logs](#logs) |
| O comando deu erro e não entendi | [Erros comuns](#erros-comuns) |
| Como saio do vim?! | [Sobrevivência](#sobrevivência) |

**Mapa das missões do CTF** está [no fim](#o-que-cada-missão-do-ctf-usa).

---

## Sobrevivência

O mínimo para não travar.

| Tecla | O que faz |
|---|---|
| `Tab` | completa nome de arquivo ou comando — **use sempre**, evita erro de digitação |
| `Ctrl+C` | interrompe o comando que está rodando |
| `Ctrl+D` | encerra a sessão (o mesmo que `exit`) |
| `Ctrl+R` | busca no histórico: comece a digitar um comando antigo |
| `↑` `↓` | navega pelos comandos anteriores |
| `Ctrl+L` | limpa a tela (o mesmo que `clear`) |

**Descobrir como um comando funciona:**

```bash
ls --help          # resumo rápido, cabe na tela
man ls             # manual completo — saia com q
apropos firewall   # procura comandos por assunto
```

**Caiu no vim sem querer?** `Esc` e depois `:q!` e `Enter`. Sai sem salvar.

---

## Navegar

| Comando | O que faz |
|---|---|
| `pwd` | mostra em que pasta você está |
| `ls` | lista o conteúdo |
| `ls -la` | lista **tudo**, inclusive ocultos, com detalhes |
| `cd /etc` | entra em `/etc` |
| `cd ..` | sobe um nível |
| `cd` ou `cd ~` | volta para a sua pasta pessoal |
| `cd -` | volta para a pasta anterior |

**Caminho absoluto vs relativo.** Um caminho que começa com `/` é absoluto — vale
de qualquer lugar. Sem a barra, é relativo à pasta onde você está agora. `pwd`
responde onde é "agora".

```
/etc/ssh/sshd_config     absoluto: sempre este arquivo
ssh/sshd_config          relativo: só funciona se você estiver em /etc
```

Atalhos: `~` é a sua pasta pessoal, `.` é a pasta atual, `..` é a pasta acima.

### As pastas que importam

| Pasta | O que tem |
|---|---|
| `/etc` | **configuração** de tudo. É onde você mais vai mexer no CTF |
| `/var/log` | **logs** — o que aconteceu na máquina |
| `/var/www` | páginas servidas pelo nginx |
| `/home` | pastas pessoais dos usuários |
| `/root` | pasta pessoal do root |
| `/usr/bin`, `/usr/sbin` | os programas instalados |
| `/tmp` | temporário; some no boot |
| `/proc` | não são arquivos de verdade — é o kernel expondo o estado do sistema |

Experimente: `cat /proc/cpuinfo`, `cat /proc/uptime`. Nada disso existe em disco.

---

## Ler arquivos

| Comando | Quando usar |
|---|---|
| `cat arquivo` | arquivo curto; despeja tudo na tela |
| `less arquivo` | arquivo longo; navega com setas, sai com `q`, busca com `/` |
| `head -20 arquivo` | as 20 primeiras linhas |
| `tail -20 arquivo` | as 20 últimas |
| `tail -f arquivo` | acompanha **ao vivo** — essencial para log |
| `grep senha arquivo` | só as linhas que contêm "senha" |
| `grep -i -n senha arquivo` | ignorando maiúsculas (`-i`), com número da linha (`-n`) |
| `wc -l arquivo` | quantas linhas |

**Procurar arquivo:**

```bash
find /etc -name "*.conf"        # por nome
grep -rn "PermitRoot" /etc/ssh  # por conteúdo, recursivo
```

### Encadear comandos

O que transforma comandos soltos em ferramenta.

```bash
ps aux | grep nginx           # a saída do primeiro vira entrada do segundo
ss -lntup | grep :80          # filtra a lista de portas
ls -la /etc > lista.txt       # grava a saída num arquivo (sobrescreve)
echo "linha nova" >> arq.txt  # acrescenta no fim
comando1 && comando2          # só roda o segundo se o primeiro der certo
```

`apt update && apt upgrade` usa o `&&` de propósito: se o primeiro falhar, não
faz sentido tentar o segundo.

---

## Editar arquivos de configuração

Use o **nano**. É o editor mais simples disponível.

```bash
sudo nano /etc/ssh/sshd_config
```

| Tecla | Ação |
|---|---|
| `Ctrl+O`, `Enter` | salva |
| `Ctrl+X` | sai |
| `Ctrl+W` | procura no arquivo |
| `Ctrl+K` | recorta a linha |

**Antes de mexer em configuração importante, faça cópia:**

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

Custa dois segundos e já salvou muita gente.

**Depois de editar, quase sempre é preciso recarregar o serviço** — o programa
leu o arquivo quando subiu e não sabe que você mudou:

```bash
sudo systemctl reload ssh     # relê a configuração sem derrubar conexões
sudo systemctl restart nginx  # reinicia de fato
```

---

## Usuários e permissões

| Comando | O que faz |
|---|---|
| `whoami` | qual usuário você é |
| `id` | seu usuário, seus grupos |
| `cat /etc/passwd` | todas as contas do sistema |
| `sudo lastlog` | quando cada conta entrou pela última vez |
| `sudo useradd` / `sudo userdel -r` | cria / remove conta |
| `sudo passwd fulano` | troca a senha de alguém |

**Só as contas de gente** (as de sistema têm UID abaixo de 1000):

```bash
awk -F: '$3 >= 1000' /etc/passwd
```

### Ler as permissões

```
-rw-r--r--  1 root root  1234 Sep  8 10:00 sshd_config
 │└┬┘└┬┘└┬┘    └─┬┘ └┬┘
 │ │  │  │       │   └── grupo dono
 │ │  │  │       └────── usuário dono
 │ │  │  └────────────── outros:  r--  = só leitura
 │ │  └───────────────── grupo:   r--  = só leitura
 │ └──────────────────── dono:    rw-  = leitura e escrita
 └────────────────────── tipo: - arquivo, d pasta, l atalho
```

**Os números.** Cada permissão vale um número: leitura 4, escrita 2, execução 1.
Some para cada um dos três grupos:

| Número | Significa | Uso típico |
|---|---|---|
| `600` | dono lê e escreve; mais ninguém vê | **chave SSH privada** |
| `644` | dono escreve, todos leem | arquivo de configuração comum |
| `755` | dono escreve, todos leem e executam | programa, pasta |
| `700` | só o dono, tudo | pasta `.ssh` |

```bash
chmod 600 ~/.ssh/minha-chave    # o SSH RECUSA chave com permissão frouxa
chmod 644 arquivo
sudo chown aluno:aluno arquivo  # troca dono e grupo
```

### sudo

`sudo comando` executa aquele comando como root. Não é "virar root" — é pedir
permissão para **uma** ação, e ela fica registrada no log com o seu nome.

Quem pode usar `sudo` está em `/etc/sudoers` e nos arquivos de
`/etc/sudoers.d/`. Vale olhar essa pasta: é lá que aparece permissão que
ninguém lembra de ter dado.

---

## Processos

| Comando | O que faz |
|---|---|
| `ps aux` | todos os processos |
| `ps aux \| grep nginx` | só os que interessam |
| `top` / `htop` | acompanha ao vivo; sai com `q` |
| `pgrep -a sshd` | processos que casam com um nome |
| `kill 1234` | pede para o processo 1234 encerrar |
| `kill -9 1234` | mata sem pedir — último recurso |

**Cada processo tem um PID** (número) e um **PPID** (quem o criou). `ps -ef --forest`
mostra a árvore.

> **Rodar não é o mesmo que estar exposto.** Muito processo roda sem abrir porta
> nenhuma. `ps` responde "o que está rodando"; `ss` responde "o que está
> acessível de fora". Confundir os dois é como se deixa um serviço aberto para a
> internet sem perceber. É a diferença que a missão 1 do CTF cobra.

---

## Serviços

Serviço é um programa que o sistema mantém rodando — sshd, nginx, fail2ban.

| Comando | O que faz |
|---|---|
| `systemctl status nginx` | está rodando? desde quando? deu erro? |
| `sudo systemctl start nginx` | liga agora |
| `sudo systemctl stop nginx` | desliga agora |
| `sudo systemctl restart nginx` | desliga e liga |
| `sudo systemctl reload ssh` | relê a configuração **sem derrubar** conexões |
| `sudo systemctl enable nginx` | passa a subir sozinho no boot |
| `sudo systemctl disable nginx` | não sobe mais no boot |
| `systemctl list-units --type=service` | tudo o que existe |

> **`start` e `enable` são coisas diferentes.** `start` liga agora, mas não
> sobrevive ao reboot. `enable` faz subir no boot, mas não liga agora. Quem
> desliga um serviço só com `stop` o encontra de volta na próxima vez que a
> máquina reiniciar — o conserto durou até o boot.

---

## Logs

| Onde | O que tem |
|---|---|
| `/var/log/auth.log` | **autenticação**: quem entrou, quem tentou e falhou |
| `/var/log/syslog` | geral do sistema |
| `/var/log/nginx/` | acessos e erros do nginx |
| `journalctl -u ssh` | log de um serviço específico |
| `journalctl -f` | acompanha ao vivo |

```bash
sudo tail -f /var/log/auth.log     # veja tentativas de login em tempo real
sudo grep "Failed password" /var/log/auth.log | wc -l
```

Vale deixar esse `tail -f` aberto numa janela enquanto você tenta entrar de
outra. Ver a própria tentativa aparecendo no log ensina mais que qualquer
explicação — e é o arquivo que o fail2ban lê.

---

## Rede

| Comando | O que faz |
|---|---|
| `ip addr` | seus endereços IP e interfaces |
| `ip route` | por onde os pacotes saem |
| `ss -lntup` | **o que está escutando**, em qual porta, qual processo |
| `ping 172.30.1.5` | a máquina responde? |
| `traceroute destino` | por onde a conexão passa |
| `dig google.com` | consulta DNS |
| `curl -I localhost` | pede só os cabeçalhos HTTP |
| `curl localhost` | pega a página inteira |
| `nc -zv host 22` | a porta está aberta? |

**Decorando o `ss -lntup`:** `l` listening, `n` números em vez de nomes, `t` TCP,
`u` UDP, `p` processo dono.

### Porta, escutar, e o 0.0.0.0

Um serviço "escuta" numa porta esperando conexões. A porta identifica **qual**
serviço recebe o quê: 22 é SSH por convenção, 80 é HTTP, 443 é HTTPS.

O endereço na frente da porta é o que mais confunde:

| Aparece como | Significa |
|---|---|
| `127.0.0.1:80` | só acessível **de dentro** da própria máquina |
| `0.0.0.0:80` | acessível **de qualquer lugar** — inclusive da internet |
| `[::]:80` | o mesmo, em IPv6 |

Serviço em `0.0.0.0` que você não pretendia expor é exatamente o problema da
missão 1. Olhe essa coluna com atenção.

### Firewall

```bash
sudo ufw status verbose        # política padrão e regras
sudo ufw allow 22/tcp          # libera uma porta
sudo ufw deny 23/tcp           # bloqueia
sudo ufw enable                # ATIVA — libere o SSH antes!
```

> **Ordem importa.** Ativar o firewall antes de liberar a porta do seu SSH
> derruba a sua conexão na hora, e você fica de fora. Sempre: libere primeiro,
> ative depois. E teste numa segunda janela antes de fechar a primeira.

---

## Pacotes

| Comando | O que faz |
|---|---|
| `sudo apt update` | baixa a **lista** do que existe — não instala nada |
| `sudo apt upgrade` | instala as atualizações |
| `sudo apt install nginx` | instala |
| `sudo apt remove nginx` | remove o programa, mantém a configuração |
| `sudo apt purge nginx` | remove **tudo**, inclusive a configuração |
| `sudo apt autoremove` | limpa dependências que ficaram órfãs |
| `dpkg -l \| grep nginx` | está instalado? qual versão? |
| `apt search firewall` | procura pacote por assunto |

> **`update` não atualiza nada.** Ele só sincroniza a lista de pacotes
> disponíveis. Quem instala é o `upgrade`. É o mal-entendido mais comum de quem
> começa, e por isso os dois quase sempre aparecem juntos:
> `sudo apt update && sudo apt upgrade`.

Para desligar um serviço de vez, `purge` é melhor que `remove`: o `remove` deixa
a configuração para trás, e reinstalar traz o problema de volta.

---

## Erros comuns

| Mensagem | O que significa | O que fazer |
|---|---|---|
| `Permission denied` | você não tem permissão | use `sudo`, ou veja o dono com `ls -l` |
| `command not found` | o programa não está instalado — ou você errou o nome | `apt search`, e confira a digitação |
| `No such file or directory` | o caminho não existe | `pwd` e `ls` para se situar; use `Tab` |
| `Unit ssh.service could not be found` | o nome do serviço é outro | `systemctl list-units --type=service \| grep ssh` |
| `Address already in use` | outro processo já ocupa a porta | `sudo ss -lntup \| grep :PORTA` |
| `WARNING: UNPROTECTED PRIVATE KEY FILE!` | sua chave está com permissão frouxa | `chmod 600 caminho/da/chave` |
| `Connection refused` | nada escutando naquela porta | o serviço está rodando? `systemctl status` |
| `Connection timed out` | algo no caminho descartou o pacote | quase sempre firewall |
| `Permission denied (publickey)` | o SSH não aceitou sua chave | chave certa? `-i` correto? permissão 600? |

**`refused` vs `timed out`** é a distinção mais útil aqui. *Refused* significa
que a máquina respondeu dizendo "não tem ninguém nessa porta" — você chegou
até ela. *Timed out* significa que ninguém respondeu nada — provavelmente um
firewall engoliu o pacote no caminho. Diagnósticos diferentes, causas
diferentes.

---

## O que cada missão do CTF usa

| Missão | Seções deste documento |
|---|---|
| 1. Serviços que ninguém pediu | [Rede](#rede), [Processos](#processos), [Pacotes](#pacotes) |
| 2. Bloquear login de root | [Editar](#editar-arquivos-de-configuração), [Serviços](#serviços) |
| 3. Desabilitar senha | [Editar](#editar-arquivos-de-configuração), [Permissões](#usuários-e-permissões) |
| 4. Pacotes atualizados | [Pacotes](#pacotes) |
| 5. Firewall | [Rede](#rede) |
| 6. fail2ban | [Serviços](#serviços), [Logs](#logs) |
| 7. nginx | [Serviços](#serviços), [Rede](#rede), [Editar](#editar-arquivos-de-configuração) |
| 8. Mudar a porta do SSH | [Editar](#editar-arquivos-de-configuração), [Rede](#rede) |
| 9. Atualizações automáticas | [Pacotes](#pacotes), [Editar](#editar-arquivos-de-configuração) |
| 10. Endurecer o sshd | [Editar](#editar-arquivos-de-configuração) |
| 11. Conta indevida | [Usuários e permissões](#usuários-e-permissões), [Logs](#logs) |
| 12. Provar o fail2ban | [Logs](#logs), [Rede](#rede) |

---

## Apêndice: e o kernel?

Agora que você já viu processos, arquivos e permissões, a peça central faz
sentido.

O **kernel** é o programa que conversa com o hardware. Ele controla memória,
disco, rede e CPU, e decide qual processo roda a cada instante. Nenhum programa
seu fala com o hardware direto: todos pedem ao kernel, por meio de **chamadas de
sistema**.

Daí vem a divisão entre **espaço de kernel** e **espaço de usuário**. Tudo o que
você fez neste documento — `ls`, `nginx`, `ssh` — roda no espaço de usuário. Um
erro ali derruba o programa; um erro no kernel derruba a máquina.

"Linux", estritamente, é só o kernel. O que você usa é o kernel **mais** as
ferramentas ao redor — por isso a distribuição (Debian, Ubuntu, Fedora) importa:
o kernel é quase o mesmo, o que muda é o resto.

```bash
uname -a              # versão do kernel
sudo dmesg | tail     # mensagens do kernel — útil quando hardware falha
lsmod                 # módulos carregados (drivers e afins)
cat /proc/cpuinfo     # o kernel expondo o estado como se fosse arquivo
```

Aquele `/proc` da seção de pastas é isto: uma janela para dentro do kernel, com
cara de sistema de arquivos. Nada ali existe em disco.

---

<sub>Achou algo errado ou faltando? Diga no canal da disciplina — este documento
é para ser corrigido.</sub>
