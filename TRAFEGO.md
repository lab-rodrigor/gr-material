# Análise de tráfego com tcpdump e Wireshark

Material de apoio do seminário 4.

Log de aplicação conta o que a sua biblioteca achou que aconteceu. A captura conta o que saiu da máquina e o que chegou.

## Onde você pode capturar

No laboratório do CTF: o `tcpdump` já está instalado, e a conta `aluno` tem `sudo`.

```
ssh -i sua-chave -p 220NN aluno@gr.lab.ayty.org
sudo tcpdump -i any -c 5 -n
```

Na sua máquina: com o Wireshark, para abrir o que você capturou no servidor.

Nos containers da atividade 2 a captura não funciona. Eles sobem com todas as capabilities removidas, e capturar exige `CAP_NET_RAW`. Um socket raw ali devolve `Operation not permitted`. É de propósito: um container que captura pacotes lê o tráfego dos vizinhos na mesma rede.

Capture apenas no seu laboratório. Os laboratórios da turma se enxergam em `172.30.0.0/16`, e ler o tráfego do colega não faz parte de nenhuma atividade.

## O básico do tcpdump

```
sudo tcpdump -i any -n -c 20 port 80
```

- `-i any` escuta em todas as interfaces. `-i eth0` escuta em uma.
- `-n` não resolve nomes. Sem isso, a consulta de DNS aparece dentro da sua própria captura.
- `-c 20` para depois de 20 pacotes.
- `port 80` é o filtro de captura. Sem filtro, a saída traz todo o tráfego da máquina.

Filtros que resolvem a maior parte dos casos:

```
host 172.30.1.5              # tudo que entra ou sai desse endereço
port 5432                    # uma porta, nos dois sentidos
tcp port 22 and host X       # combinando
not port 22                  # tudo menos SSH
icmp                         # ping
```

Capturar dentro de uma sessão SSH sem excluir a porta dela realimenta a captura: cada pacote capturado gera saída na tela, que gera mais pacotes. Use `not port 22`, ou a porta nova, se você mudou o SSH na missão 8.

Para guardar em arquivo e ler depois:

```
sudo tcpdump -i any -n -w captura.pcap port 80
sudo tcpdump -r captura.pcap -n
```

O `-A` mostra o conteúdo em texto. Serve para protocolo sem cifra, como HTTP:

```
sudo tcpdump -i any -n -A port 80
```

## Ler uma linha

```
14:02:11.512 IP 172.30.1.7.52134 > 172.30.1.9.80: Flags [S], seq 1829, win 64240, length 0
```

Da esquerda para a direita: hora, origem com porta, destino com porta, flags, número de sequência, janela e tamanho dos dados.

As flags:

| Flag | Significa |
|---|---|
| `[S]` | pedido de conexão |
| `[S.]` | conexão aceita |
| `[.]` | confirmação, sem dados |
| `[P.]` | dados sendo entregues |
| `[F.]` | fim combinado |
| `[R]` | conexão cortada na hora |

Uma conexão saudável começa com `[S]`, `[S.]`, `[.]`. Esses três pacotes são o aperto de mão.

## Falhas que parecem a mesma

O usuário vê "não conectou" nos três casos. A captura separa.

**Recusada.** Sai um `[S]` e volta um `[R]`. A máquina existe, o pacote chegou, e ninguém escutava naquela porta. Erro típico: o serviço não subiu, ou subiu em outra porta.

```
sudo tcpdump -i any -n port 9999 &
curl -s http://localhost:9999
```

**Tempo esgotado.** Sai `[S]`, e não volta nada. O pacote morreu no caminho, quase sempre num firewall. Você vê o mesmo `[S]` repetido em intervalos crescentes: é o seu sistema retransmitindo.

Foi o que aconteceu com dois colegas na missão 5. Eles liberaram no `ufw` a porta externa, `220NN`, enquanto o pacote chegava na porta 22 lá dentro. Uma captura mostraria o `[S]` chegando e nenhuma resposta saindo.

**Cortada no meio.** A conexão abre, troca dados, e aparece um `[R]` fora de hora. O serviço aceitou e morreu, ou alguém no caminho derrubou a conexão.

## Wireshark

Instale na sua máquina e traga o arquivo capturado no servidor:

```
ssh -i sua-chave -p 220NN aluno@gr.lab.ayty.org \
  "sudo tcpdump -i any -n -w - not port 220NN" > captura.pcap
```

O `-w -` manda a captura pela própria sessão SSH, sem arquivo intermediário. Encerre com Ctrl-C e abra `captura.pcap` no Wireshark.

O que o Wireshark faz e o `tcpdump` não:

- O filtro de exibição, diferente do filtro de captura: `tcp.port == 80`, `ip.addr == 172.30.1.9`, `http`, `dns`.
- **Follow TCP Stream**, que remonta a conversa inteira de uma conexão em ordem.
- **Statistics → Conversations**, que lista quem falou com quem, quantos bytes e por quanto tempo.
- `tcp.analysis.retransmission` e `tcp.analysis.zero_window`, que apontam perda e congestionamento sem você procurar pacote a pacote.

## Exercícios no seu laboratório

**1. O aperto de mão.** Em duas sessões. Na primeira:

```
sudo tcpdump -i any -n -c 10 port 80
```

Na segunda:

```
curl -s localhost > /dev/null
```

Identifique `[S]`, `[S.]` e `[.]` na saída da primeira.

**2. HTTP viaja aberto.** Repita com `-A` e leia a requisição e a resposta. Todo cabeçalho, cookie e token que a sua aplicação mandar está ali, legível por qualquer um no caminho.

**3. Recusada contra sem resposta.** Compare `curl http://localhost:9999` com um `curl` para uma porta que o `ufw` bloqueia. Uma devolve `[R]` na hora; a outra repete `[S]` e desiste.

**4. DNS.** Capture `port 53` e rode `curl -s https://example.com > /dev/null`. Você vê a pergunta, a resposta, e só então a conexão.

**5. TLS.** Capture `port 443` durante o mesmo `curl` e procure o handshake. Aparecem o nome do servidor no SNI, os tamanhos e os tempos. O conteúdo fica cifrado.

**6. Leve para o Wireshark.** Grave uns 30 segundos com `-w`, traga para a sua máquina e use Follow TCP Stream numa conexão HTTP.

## O que a captura não resolve

Tráfego cifrado não se lê sem a chave. Em HTTPS você enxerga o handshake, o SNI, os tamanhos e os tempos. Para ver o conteúdo, é preciso capturar antes da cifra: log da aplicação ou proxy no meio.

Volume alto derruba a utilidade. Sem filtro, uma captura de trinta segundos num servidor movimentado vira um arquivo que ninguém lê. Filtre na captura, não só na exibição.

Capturar exige privilégio. O `tcpdump` pede `sudo`, e os containers da atividade 2 não capturam, porque quem captura numa rede lê tudo o que passa nela.

## Para consultar

- `man tcpdump`, seção EXAMPLES, curta e com os filtros mais usados.
- `man pcap-filter`, a sintaxe dos filtros de captura.
- No Wireshark, Help → Manual Pages → Display Filter Reference.
