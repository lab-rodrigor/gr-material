# Atividade 8: quando o agente avisa

- Prazo: quinta-feira, 22/10/2026, às 20h
- Unidade: 2
- Onde: no seu laboratório
- Acompanhamento: `/atividades atividade:8` no Discord
- Nota: proporcional aos itens cumpridos. 4 itens valem 10.

Na atividade 7 você perguntou e o agente respondeu. Toda a conversa começava do seu lado: sem pergunta, nenhuma resposta, por mais que a máquina observada tivesse algo a dizer.

Isso tem um custo que aparece quando a rede cresce. Se o seu gerente pergunta a cada cinco minutos, uma interface que cai logo depois da pergunta fica cinco minutos caída sem ninguém saber. Perguntar de dez em dez segundos reduz a espera e multiplica o tráfego e a carga nos equipamentos.

O trap resolve isso pelo outro lado: o agente manda um aviso na hora em que o evento acontece, sem esperar pergunta nenhuma.

## O que é um trap

Um trap é uma mensagem que o agente envia por conta própria ao gerente. A porta é a 162, vizinha da 161 que você já usou, e o transporte continua sendo UDP.

Do outro lado existe um receptor esperando. Nesta atividade ele é o mesmo `roteador-escola`, em `172.30.0.161`, onde eu deixei um `snmptrapd` escutando a porta 162 e registrando tudo o que chega. A community dele é `alertas`.

O que viaja num trap: o tempo que o agente está no ar, um OID que diz o tipo de aviso, e um ou mais pares de OID e valor que o agente quis mandar junto. O conteúdo desses pares é escolha de quem envia.

O trap não tem resposta. O agente manda e segue a vida, e se o pacote se perder no caminho, ninguém reclama. Isso é o UDP aparecendo de novo: existe a versão `inform`, que pede confirmação, e ela não entra nesta atividade.

## Passo 1: mandar um trap à mão

Entre no seu laboratório e mande o primeiro aviso, trocando `SEU-SLUG` pelo seu identificador:

```
snmptrap -v2c -c alertas 172.30.0.161 "" 1.3.6.1.4.1.8072.2.3.0.1 \
  SNMPv2-MIB::sysLocation.0 s "GR8-SEU-SLUG"
```

Lendo o comando por partes:

`-v2c -c alertas` é a versão e a community do receptor, igual ao que você usou no `snmpget`.

`172.30.0.161` é para onde o aviso vai.

As aspas vazias são o tempo de atividade. Deixe assim e o próprio agente preenche.

`1.3.6.1.4.1.8072.2.3.0.1` é o OID que identifica o tipo de aviso. Esse é um OID de exemplo do net-snmp, que serve para testar sem inventar um número próprio.

O resto é o par que viaja junto: o OID `sysLocation.0`, o tipo `s` de texto, e o valor.

O comando não imprime nada quando dá certo, porque não existe resposta para esperar. Confira pelo `/atividades atividade:8`.

## Passo 2: o agente avisando sozinho

Mandar trap na mão serve para entender o caminho. O uso de verdade é o agente avisar quando algo acontece com ele.

Acrescente uma linha ao `/etc/snmp/snmpd.conf`:

```
trap2sink 172.30.0.161 alertas
```

Ela diz ao agente para onde mandar os avisos dele. Reinicie:

```
sudo pkill snmpd
sudo snmpd -Lf /dev/null
```

Ao subir, o agente manda sozinho um `coldStart`, que é o aviso de "acabei de ligar". Todo equipamento de rede faz isso, e é assim que um sistema de monitoramento descobre que um roteador reiniciou de madrugada sem ter perguntado nada a ele.

## Passo 3: publicar uma métrica que não existe na MIB

As MIBs trazem o que os autores previram: interfaces, memória, disco, processos. Elas não trazem quantos pedidos a sua aplicação respondeu, nem quantos arquivos ficaram na fila de backup.

A diretiva `extend` liga um comando seu a um OID. O agente executa o comando quando alguém pergunta, e devolve a saída como valor.

Acrescente ao `/etc/snmp/snmpd.conf`:

```
extend processos /bin/sh -c "ps -e --no-headers | wc -l"
```

Reinicie o agente e leia de dentro do laboratório:

```
snmpget -v2c -c turma -Oqv 127.0.0.1 \
  'NET-SNMP-EXTEND-MIB::nsExtendOutputFull."processos"'
```

O nome que você deu, `processos`, faz parte do OID, entre aspas. Para o item contar, ele precisa ser exatamente esse, porque é por ele que eu vou perguntar.

As aspas simples em volta do OID inteiro existem para o shell não comer as aspas duplas de dentro.

A partir daqui, qualquer gerente do mundo lê o seu número sem saber que por trás dele existe um `ps`. É isso que `extend` faz: transforma um comando local em valor consultável pela rede.

## Passo 4: o trap com a medida dentro

Agora os dois juntos. O aviso vai levar o número que o seu agente publica:

```
V=$(snmpget -v2c -c turma -Oqv 127.0.0.1 \
  'NET-SNMP-EXTEND-MIB::nsExtendOutputFull."processos"')
snmptrap -v2c -c alertas 172.30.0.161 "" 1.3.6.1.4.1.8072.2.3.0.1 \
  SNMPv2-MIB::sysLocation.0 s "$V"
```

A primeira linha guarda o valor numa variável, e a segunda manda esse valor dentro do trap.

Um alerta que chega com a medida dentro poupa uma ida e volta: quem recebe já sabe o número e decide na hora. Um alerta que só diz "algo aconteceu" obriga o gerente a perguntar de volta, e a resposta vem do estado de agora, que pode já ser outro.

## Os 4 itens

1. Um trap seu, mandado à mão, com `GR8-` seguido do seu slug no valor.
2. Um `coldStart` do seu agente, que só chega com o `trap2sink` configurado.
3. A sua métrica publicada em `nsExtendOutputFull."processos"`, com o número batendo com a contagem de processos do seu laboratório.
4. Um trap seu carregando esse número, com folga de 5 para mais ou para menos.

Os traps ficam registrados no servidor com o endereço de origem, e é por ele que eu sei de quem veio cada um. Não existe arquivo para preencher nesta atividade.

Confira o que já conta com `/atividades atividade:8`.

## Quando algo não funciona

- O `snmptrap` não imprime nada: é o comportamento normal dele, porque trap não tem resposta. Confira pelo bot.
- O item 2 não fecha: o `trap2sink` precisa estar no `snmpd.conf` antes de o agente subir. Reinicie e confira com `pgrep -a snmpd`.
- O item 3 diz `No Such Instance`: o agente subiu sem a linha `extend`, ou o nome não é `processos`. Reinicie depois de editar.
- O item 3 diz que o número não bate: o `extend` conta os processos do laboratório inteiro, e a diferença aceita é de 5 para mais ou para menos.
- Nada chega ao servidor: o trap sai do seu laboratório para a porta 162. Se você fechou a saída no firewall durante o CTF, libere com `sudo ufw allow out 162/udp`.
- Outras dúvidas: `/ajuda problema texto: ...` no Discord.
