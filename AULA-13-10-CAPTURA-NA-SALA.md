# Aula de 13/10: captura na rede da sala

- Quando: hoje, das 10h às 11h30
- Onde: na máquina que você está usando na sala, e o laboratório como alternativa
- Entrega: três códigos em `~/aula.txt`, dentro do seu laboratório
- Acompanhamento: `/atividades atividade:aula` no Discord

O computador do professor emite, de 20 em 20 segundos, quatro datagramas UDP em broadcast na rede da sala. Eles carregam três códigos, cada um escondido de um jeito diferente. O servidor repete a mesma emissão dentro da rede dos laboratórios, para quem não conseguir capturar na própria máquina.

A porta é a mesma nas duas redes, e o professor diz qual no começo da aula.

## As 3 missões

1. O código em texto puro, no campo `codigo=`.
2. O código partido em dois datagramas seguidos, nos campos `parte=1` e `parte=2`. Você junta.
3. O código em base64, no campo `b64=`. Você decodifica.

## Capturar na máquina da sala

```
sudo tcpdump -i any -n -A udp port PORTA
```

Sem `sudo`, ou com o Wireshark, se for o que estiver instalado. No Wireshark, o filtro é `udp.port == PORTA`.

Os pacotes chegam a você porque o destino é o endereço de broadcast da sub-rede. Num switch, um pacote unicast só chega ao destinatário, e você não veria nada.

## Capturar pelo laboratório

Se a sua máquina não deixar capturar, entre no seu laboratório e capture lá:

```
ssh -i ~/.ssh/gr-lab -p SUAPORTA aluno@gr.lab.ayty.org
sudo tcpdump -i any -n -A udp port PORTA
```

É a mesma emissão, repetida pelo servidor.

## Entregar

No laboratório, uma linha por missão:

```
missao1=AULA-XXXXXXXXXX
missao2=AULA-XXXXXXXXXX
missao3=AULA-XXXXXXXXXX
```

Maiúscula e minúscula não importam. Confira com `/atividades atividade:aula`.

## Quando algo não funciona

- Nada aparece na sua máquina: o seu laptop pode estar em outra rede, por exemplo no wi-fi enquanto a emissão sai no cabo. Use o laboratório.
- `sudo tcpdump` pede senha e você não tem: use o laboratório.
- Aparece `parte=1` e nunca `parte=2`: eles saem com meio segundo de intervalo, então role a tela para cima.
- O `base64 -d` reclama: copie só o valor depois de `b64=`, sem espaço no fim.
