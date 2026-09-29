# Um servidor da turma foi invadido

Em 24/09/2026 descobrimos que o laboratório do professor estava minerando criptomoeda para outra pessoa havia nove dias. O que segue é o registro do caso, com os números que ficaram nos logs.

## A máquina

O laboratório do professor nasceu em 07/09 igual ao de vocês: Debian com sshd, root aceitando senha, uma conta que ninguém sabe explicar, serviços escutando à toa e nenhum firewall. Era o "servidor entregue" do CTF.

Os 31 laboratórios da turma passaram pelas 12 missões. O do professor não. Ele ficou exposto na porta 22000, com senha de root fraca, do dia 07 ao dia 24.

## A linha do tempo

Os números vêm do `/var/log/auth.log` do container.

- 738 tentativas de senha falharam.
- 14 logins de root foram aceitos, vindos de 8 endereços diferentes.
- O primeiro acerto foi em 15/09, às 18:54.
- Em 24/09 havia duas sessões de root abertas, vindas de `62.133.62.86`.

Entre o primeiro acerto e a descoberta passaram-se nove dias.

## O que o invasor instalou

```
/etc/kswpad          minerador, 47% de CPU, 5873 minutos de processador acumulados
/usr/bin/dpkgd/ps    o comando `ps` substituído
198.251.81.61:2070   conexão de saída permanente, comando e controle
```

O `ps` trocado é a parte que interessa. Ele existe para que o minerador não apareça na lista de processos. Um administrador que digitasse `ps aux` naquela máquina veria uma lista limpa.

## Como o caso apareceu

Ninguém foi procurar. A descoberta veio de uma captura de pacotes feita para testar outra atividade:

```
sudo tcpdump -i lo -nn -w /tmp/teste.pcap
```

Oito segundos de captura na interface de loopback devolveram 29.282 pacotes, todos para a porta 6001. Era o minerador conversando consigo mesmo.

A captura mostrou o que o `ps` escondia. Foi por isso que a análise de tráfego entrou no semestre.

## Como se olha uma máquina em que não se pode confiar

Depois que alguém entra como root, os comandos da própria máquina passam a ser suspeitos. `ps`, `ls`, `netstat` e `top` podem ter sido trocados para mentir.

A saída é olhar de fora. Como os laboratórios são containers, o servidor consegue listar tudo o que mudou desde a imagem original, sem executar nada de dentro:

```
docker diff gr-lab-fulano
```

O laboratório suspeito de um aluno tinha 4770 arquivos alterados. Comparando essa lista com a de dois laboratórios limpos, sobraram quatro arquivos exclusivos, e os quatro eram trabalho dele: `jail.local`, `hardening.conf` e `port.conf`.

O segundo teste compara os binários com os de uma máquina limpa:

```
docker cp gr-lab-fulano:/usr/bin/ps - | sha256sum
```

Hash igual significa binário íntegro. No laboratório do professor, o do `ps` era diferente.

## O laboratório que resistiu

Um laboratório de aluno recebeu 1679 tentativas de senha e teve um login de root aceito, em 11/09 de manhã. Ele fechou root e senha algumas horas depois, ao cumprir as missões 2 e 3, e nenhum rastro ficou: nenhum binário alterado, nenhuma tarefa no cron, nenhuma chave em `/root/.ssh`, nenhuma conexão de saída. Mesmo assim o laboratório foi recriado, porque quem entra como root pode apagar o próprio rastro.

Nos laboratórios com fail2ban de pé, o contador de banimentos já marcava 68, 8 e 3 em máquinas diferentes. Cada banimento é uma sequência de tentativas que parou de chegar.

## O que cada missão teria impedido

| Missão | O que ela fecha |
|---|---|
| 2. Bloquear o login de root | tira `root` da lista de nomes que o atacante tenta |
| 3. Desabilitar autenticação por senha | acaba com a força bruta: não há o que adivinhar |
| 5. Ligar o firewall | reduz o que está acessível |
| 6. Pôr o fail2ban de pé | corta a sequência de tentativas |
| 8. Mudar a porta do SSH | tira a máquina da varredura automática que só olha a 22 |

As missões 2 e 3 sozinhas teriam bastado. Nenhuma das 14 entradas usou chave: todas usaram senha de root.

## O que foi feito depois

O container comprometido foi parado, copiado como prova e apagado. O laboratório do professor foi recriado já fechado.

Seis laboratórios ainda aceitavam root com senha em 24/09. Os seis foram fechados, e o fail2ban foi instalado e ligado nos 32.

A configuração ficou em `/etc/fail2ban/jail.d/zz-sshd-professor.conf`, com 5 tentativas em 10 minutos e banimento de 1 hora. O `jail.local` que vocês escreveram na missão 6 continua valendo.

## O que sobra disso

Uma máquina exposta na internet com senha de root é encontrada em dias, sem ninguém a escolher. As 738 tentativas no laboratório do professor e as 1679 no do aluno vieram de varredura automática, não de alguém interessado na disciplina.

O servidor que vocês prepararam no CTF é a diferença entre essas duas máquinas.
