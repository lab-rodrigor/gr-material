# Atividade 7: SNMP, perguntar números a uma máquina

- Prazo: sexta-feira, 16/10/2026, às 20h
- Unidade: 2
- Onde: no seu laboratório
- Acompanhamento: `/atividades atividade:7` no Discord
- Nota: proporcional aos itens cumpridos. 5 itens valem 10.

Até agora você mediu de fora: o monitor da atividade 5 perguntava ao front por HTTP, e o Uptime Kuma fazia o mesmo com uma tela melhor. Os dois sabem se o serviço respondeu, e mais nada. Quantos bytes passaram pela interface na última hora, quanto de disco sobrou, há quanto tempo a máquina está ligada: nada disso sai de um GET.

É para isso que existe SNMP.

## O que é SNMP

SNMP é um protocolo para perguntar números a um equipamento. Ele tem dois lados:

O **agente** roda na máquina observada e mantém uma lista de valores: bytes que passaram em cada interface, memória livre, processos rodando, há quanto tempo está no ar. Ele não envia nada por conta própria, fica esperando a pergunta.

O **gerente** é quem pergunta. Pode ser um comando de uma linha, como você vai usar hoje, ou um sistema que pergunta a trezentos equipamentos de minuto em minuto e desenha gráficos.

A conversa é por UDP, na porta 161. Uma pergunta, uma resposta.

### O nome de cada valor

Cada valor que o agente guarda tem um endereço numérico, chamado OID, que é um caminho numa árvore:

```
1.3.6.1.2.1.1.1.0
```

Esse caminho é sempre o mesmo em qualquer equipamento do mundo: é a descrição do sistema. Um fabricante de roteador não pode mudá-lo de lugar, e é isso que faz um gerente falar com equipamento de dez marcas diferentes.

Decorar número seria impossível, então existe a **MIB**, que é um dicionário de nomes para esses caminhos. Com a MIB instalada, o mesmo valor se escreve assim:

```
SNMPv2-MIB::sysDescr.0
```

O `.0` no fim não é enfeite. Ele significa "este valor é único, não é uma lista". Valores que existem em várias cópias, um por interface, terminam com o número da interface.

### As três versões

A **v1** é de 1988 e ainda aparece em equipamento velho.

A **v2c** é a mais usada. A autenticação é uma palavra chamada *community*, que viaja em texto puro dentro do pacote. Quem captura o pacote lê a palavra. Você vai ver isso acontecer hoje.

A **v3** tem usuário, senha e cifra. É o que se usa quando a consulta sai da rede local.

## O que você vai fazer

Primeiro você é o gerente, consultando o agente da disciplina. Depois você é o agente, respondendo às perguntas que o servidor faz ao seu laboratório.

O agente da disciplina é uma máquina de brinquedo chamada `roteador-escola`, no endereço `172.30.0.161`, alcançável de dentro do seu laboratório. A community de leitura é `redes2026`.

## Passo 1: entre no laboratório

```
ssh -i ~/.ssh/gr-lab -p SUAPORTA aluno@gr.lab.ayty.org
```

O `snmpget`, o `snmpwalk` e o `snmptable` já estão instalados, com as MIBs.

## Passo 2: a primeira pergunta

```
snmpget -v2c -c redes2026 172.30.0.161 SNMPv2-MIB::sysDescr.0
```

Lendo o comando: `-v2c` é a versão, `-c redes2026` é a community, depois vem o endereço do agente e o valor que você quer.

A resposta sai assim:

```
SNMPv2-MIB::sysDescr.0 = STRING: Linux roteador-escola 6.8.0-137-generic ...
```

**Escreva essa resposta** em `~/snmp.txt`, só o texto depois de `STRING: `:

```
item1=Linux roteador-escola 6.8.0-137-generic ...
```

Se quiser só o valor, sem o nome do OID na frente, acrescente `-Oqv` ao comando.

## Passo 3: caminhar na árvore

O `snmpget` pede um valor. O `snmpwalk` pede todos os valores abaixo de um ponto:

```
snmpwalk -v2c -c redes2026 172.30.0.161 SNMPv2-MIB::system
```

Saem o nome, a localização, o contato, o tempo no ar. É assim que se descobre o que um equipamento desconhecido tem para oferecer: você caminha e vê.

## Passo 4: a tabela de interfaces

Aqui a coisa fica interessante. Tudo o que tem uma cópia por interface vive numa tabela, e existe um comando que a desenha:

```
snmptable -v2c -c redes2026 -Cb 172.30.0.161 IF-MIB::ifTable
```

```
 Index Descr             Type   Mtu  Speed AdminStatus OperStatus  InOctets OutOctets
     1    lo softwareLoopback 65536 10000000        up         up         0         0
     2  eth0   ethernetCsmacd  1500 ...             up         up      2768      3110
```

Cada linha é uma interface. `InOctets` e `OutOctets` são os bytes que entraram e saíram desde que a máquina ligou.

**Escreva o nome da interface de índice 2** em `~/snmp.txt`:

```
item2=...
```

## Passo 5: de contador para taxa

O `InOctets` é um número que só cresce. O agente não sabe dizer "estão passando 2 Mbit/s": ele guarda o total desde o início. A velocidade é uma conta que o gerente faz, com duas leituras:

```
A=$(snmpget -v2c -c redes2026 -Oqv 172.30.0.161 IF-MIB::ifInOctets.2)
sleep 10
B=$(snmpget -v2c -c redes2026 -Oqv 172.30.0.161 IF-MIB::ifInOctets.2)
echo "$(( (B - A) / 10 )) bytes por segundo"
```

É exatamente isso que todo gráfico de banda faz, a cada intervalo, para sempre.

Dois detalhes que mordem na vida real. O contador é de 32 bits e volta a zero quando estoura, o que numa interface de 1 Gbit/s acontece em poucos minutos; por isso existe o `ifHCInOctets`, de 64 bits. E se o equipamento reiniciar entre as duas leituras, a subtração dá negativo, o que você precisa tratar.

## Passo 6: a community em texto puro

O servidor da disciplina pergunta ao seu laboratório de dois em dois minutos, com uma community que é só sua. Ela viaja em texto puro dentro do pacote. Capture e leia:

```
sudo tcpdump -i any -n -A udp port 161
```

Espere a próxima pergunta chegar. No meio dos caracteres ilegíveis do pacote aparece uma palavra começando com `GR7-`.

**Escreva essa palavra** em `~/snmp.txt`:

```
item3=GR7-...
```

Ela aparece mesmo com o seu agente desligado: o pacote chega de qualquer maneira, e quem captura, lê. É esse o problema do v2c, e é por isso que ninguém expõe SNMP v2c para a internet.

## Passo 7: agora você é o agente

Edite o `/etc/snmp/snmpd.conf` do seu laboratório e deixe só estas linhas:

```
agentaddress udp:161
rocommunity turma 172.30.0.0/16
rocommunity turma 127.0.0.1
sysLocation SEU-SLUG
```

Troque `SEU-SLUG` pelo seu identificador, aquele que aparece no nome do seu laboratório: se o seu e-mail é `joao.silva@dcx.ufpb.br`, o slug é `joao-silva`.

Lendo as linhas: `agentaddress` diz em que porta escutar, `rocommunity` cria uma senha de leitura válida só para quem vem daquela rede, e `sysLocation` é um texto livre que o agente devolve.

Suba o agente:

```
sudo snmpd -Lf /dev/null
```

E pergunte a ele mesmo, de dentro do laboratório:

```
snmpget -v2c -c turma -Oqv 127.0.0.1 SNMPv2-MIB::sysLocation.0
```

Precisa responder o seu slug. A partir daí o servidor pergunta de fora, e o item fecha sozinho.

## Passo 8: a mesma pergunta, agora cifrada

Pare o agente:

```
sudo pkill snmpd
```

Crie o usuário de v3:

```
sudo net-snmp-create-v3-user -ro -A redes2026snmp -a SHA-512 -X redes2026snmp -x AES aluno
```

Lendo: `-ro` é só leitura, `-a SHA-512` com `-A` é a senha de autenticação, `-x AES` com `-X` é a senha de cifra, e `aluno` é o nome do usuário. O comando escreve a configuração sozinho.

Suba o agente de novo e pergunte em v3:

```
sudo snmpd -Lf /dev/null
snmpget -v3 -l authPriv -u aluno -a SHA-512 -A redes2026snmp -x AES -X redes2026snmp \
  127.0.0.1 SNMPv2-MIB::sysLocation.0
```

`-l authPriv` quer dizer autenticado e cifrado. Existem também `noAuthNoPriv` e `authNoPriv`, que autenticam sem esconder o conteúdo.

**Capture essa consulta** como você fez no passo 6 e compare. Em v2c você lia a community e o valor. Aqui não se lê nem o nome do usuário.

## Os 5 itens

1. A descrição do agente da disciplina, em `item1`.
2. O nome da interface de índice 2, em `item2`.
3. A community que o gerente usa com você, em `item3`.
4. O seu agente respondendo em v2c, com a community `turma` e `sysLocation` igual ao seu slug.
5. O mesmo agente respondendo em v3, com o usuário `aluno`.

Os três primeiros você escreve em `~/snmp.txt`, uma linha cada. Os dois últimos eu verifico perguntando ao seu laboratório.

Confira o que já conta com `/atividades atividade:7`.

## Quando algo não funciona

- `Timeout: No Response`: o agente não está no ar, ou a community está errada, ou o endereço. Confira com `ss -lnup | grep 161` dentro do laboratório.
- Os nomes aparecem como `iso.3.6.1.2.1...`: falta MIB. No seu laboratório elas já estão instaladas; se sumirem, o pacote é o `snmp-mibs-downloader`, e a linha `mibs :` do `/etc/snmp/snmp.conf` precisa estar comentada.
- `snmptable` responde "Was that a table?": é o mesmo problema de MIB.
- O `tcpdump` não mostra nada na 161: a pergunta do servidor chega de dois em dois minutos, então espere. E capture em `-i any`.
- Outras dúvidas: `/ajuda problema texto: ...` no Discord.
