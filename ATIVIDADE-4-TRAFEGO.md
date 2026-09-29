# Atividade 4: captura de tráfego

- Prazo: sexta-feira, 02/10/2026, às 20h
- Unidade: 1
- Nota: proporcional às missões cumpridas. 8 missões valem 10.
- Onde: dentro do seu laboratório do CTF
- Acompanhamento: `/atividades atividade:4` no Discord

O servidor injeta tráfego dentro do seu laboratório de poucos em poucos minutos. Cada emissão carrega uma flag que só existe para você, e o único jeito de lê-la é capturando o pacote.

## Como entregar

As respostas vão em `~/flags.txt`, dentro do laboratório, uma por linha:

```
missao1=GR4-XXXXXXXXXXXX
missao2=54321
missao3=GR4-XXXXXXXXXXXX
```

Maiúscula e minúscula não importam. A missão 2 é um número de porta, a 6 é um nome de servidor, as outras são flags no formato `GR4-` seguido de 12 caracteres.

A missão 8 entrega um arquivo, em `~/capturas/handshake.pcap`.

## Antes de começar

Entre no seu laboratório:

```
ssh -i sua-chave -p 220NN aluno@gr.lab.ayty.org
```

O `tcpdump` já está instalado, e a conta `aluno` tem `sudo`. O material de apoio está em [TRAFEGO.md](TRAFEGO.md).

Capture apenas no seu laboratório. Os laboratórios da turma se enxergam em `172.30.0.0/16`, e ler o tráfego do colega fica fora desta atividade.

## Ritmo das emissões

As missões de 1 a 6 saem a cada 5 minutos. Se você não viu, espere.

A missão 7 sai uma vez por dia, num minuto sorteado entre 8h e 20h. Para essa você precisa deixar a captura gravando em arquivo.

## As 8 missões

### 1. A flag no cabeçalho HTTP

**O que é.** Uma requisição HTTP sai do seu laboratório com um cabeçalho `X-Flag`. HTTP viaja em texto, e o cabeçalho aparece inteiro na captura.

**Faça.** Capture a porta 80 mostrando o conteúdo:

```
sudo tcpdump -i any -n -A port 80
```

Espere a próxima emissão. Ela acontece de poucos em poucos minutos.

Qualquer pessoa no caminho lê esse cabeçalho. O mesmo vale para token e cookie numa aplicação sem TLS.

### 2. A porta de destino

**O que é.** Uma conexão sai para uma porta que só você tem. A resposta desta missão é o número dessa porta.

**O problema.** Você não sabe qual filtro usar, porque o filtro é justamente o que você procura.

**Faça.** Capture sem filtro de porta e olhe o que aparece. Reduza o ruído tirando o SSH, que é a sua própria sessão:

```
sudo tcpdump -i any -nn not port 22
```

O destino recusa a conexão, então você vê o `[S]` seguido de `[R]`. A porta está na linha, depois do endereço.

Use `-nn`, com dois enes. Com um só, o tcpdump troca o número por nome de serviço quando ele existe em `/etc/services`.

### 3. O nome perguntado no DNS

**O que é.** Uma consulta de DNS pergunta por um nome que contém a flag.

**Faça.** A consulta vai para o resolvedor do Docker, na porta 53:

```
sudo tcpdump -i any -n -A port 53
```

O nome perguntado aparece no pacote. Com `-A` você lê o texto; a alternativa é `-v`, que faz o tcpdump decodificar o protocolo e mostrar a pergunta.

A consulta de DNS fica visível mesmo quando a conexão seguinte é cifrada. Quem observa a rede vê o nome que você procurou.

### 4. A flag dentro do ping

**O que é.** Um ping sai com a flag dentro do payload.

**Faça.** ICMP não tem porta, então o filtro é o protocolo:

```
sudo tcpdump -i any -n -A icmp
```

O payload aparece em texto no fim da saída. Com `-X` você vê o hexadecimal ao lado do texto.

Um firewall que libera ICMP sem olhar o conteúdo deixa passar qualquer dado dentro do ping.

### 5. A flag partida em dois pacotes

**O que é.** A flag sai partida em dois pacotes, com um segundo de intervalo.

**O problema.** Cada pacote sozinho tem metade da resposta. Procurar `GR4-` numa linha só devolve pedaço.

**Faça.** Junte os dois. O `-A` já mostra os dois pedaços em sequência na mesma conexão; basta ler na ordem. No Wireshark, **Follow TCP Stream** faz a remontagem.

TCP entrega à aplicação um fluxo contínuo e esconde dela a divisão em pacotes. Na captura essa divisão aparece.

### 6. O nome do servidor no TLS

**O que é.** Uma conexão TLS sai com um nome de servidor que contém a flag. A resposta é esse nome.

**Faça.** Capture a porta 443 e procure o começo da conversa:

```
sudo tcpdump -i any -n -A port 443
```

O nome está no SNI, que viaja em claro dentro do ClientHello, antes de a cifra começar.

A cifra esconde o conteúdo da conexão. O destino, o tamanho e o tempo de cada pacote continuam visíveis.

### 7. A flag que sai uma vez por dia

**O que é.** Esta flag sai uma vez por dia, num minuto sorteado entre 8h e 20h, num cabeçalho `X-Flag-Unica`.

**O problema.** Você não vai estar olhando a tela na hora.

**Faça.** Deixe a captura gravando em arquivo, em segundo plano:

```
sudo tcpdump -i any -n -w ~/capturas/dia.pcap port 80 &
```

Depois leia o arquivo quando quiser:

```
sudo tcpdump -r ~/capturas/dia.pcap -A | grep X-Flag-Unica
```

Captura sem filtro num servidor com tráfego enche o disco. Filtre na captura, e não só na leitura.

### 8. A captura do aperto de mão

**O que é.** Esta missão entrega um arquivo. Grave em `~/capturas/handshake.pcap` uma captura que contenha um aperto de mão TCP completo, originado do seu laboratório.

**Faça.** Em duas sessões. Numa:

```
sudo tcpdump -i any -n -w ~/capturas/handshake.pcap port 80
```

Na outra:

```
curl -s http://172.30.0.1/ > /dev/null
```

Encerre a captura com Ctrl-C.

O corretor confere os três pacotes do aperto de mão, `[S]`, `[S.]` e `[.]`, com o IP do seu laboratório de um dos lados.

## Quando algo não funciona

- Nenhum pacote aparece: confira o filtro. `-i any` escuta em todas as interfaces, e `port 80` só mostra a porta 80.
- A saída rola sem parar: você está capturando a sua própria sessão SSH. Acrescente `not port 22`, ou a porta nova, se você mudou o SSH na missão 8 do CTF.
- O `/atividades` diz "sem resposta em flags.txt": confira o nome do arquivo e o formato `missaoN=valor`.
- Outras dúvidas: `/ajuda problema texto: ...` no Discord.
