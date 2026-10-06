# Atividade 5: plantão

- Prazo: quarta-feira, 07/10/2026, às 22h (reaberto)
- Unidade: 1 (última atividade da unidade)
- Onde: no seu laboratório e no repositório da atividade 3
- Acompanhamento: `/atividades atividade:5` no Discord

O servidor vai derrubar uma camada da sua topologia num minuto sorteado, sem avisar. Você precisa perceber, dizer qual camada caiu e a que hora, e consertar por um push na `main`.

A janela: a falha entra entre sexta-feira 02/10 às 20h e domingo 04/10 às 20h. O minuto é sorteado por aluno. Quem ainda não recebeu a falha recebe quando a topologia estiver de pé, até as 20h de 07/10.

Se a sua topologia não estiver de pé na hora sorteada, o servidor adia a falha em uma hora e tenta de novo.

## O que o servidor pode fazer

Uma destas cinco, sorteada por aluno:

- para o container do banco
- congela o container do banco (`docker pause`)
- desconecta o banco da rede interna
- para o container da API
- para o container do front

Todas se desfazem quando o pipeline sobe a topologia de novo. Os dados do banco continuam no volume.

## O que entregar

- `~/monitor.log`, no laboratório: uma linha por minuto, com os dois códigos HTTP do seu front
- `~/capturas/falha.pcap`, no laboratório: o tráfego do monitor na porta do front durante a falha
- `~/laudo.md`, no laboratório: quatro campos, `camada`, `hora`, `sintoma` e `acao`
- o conserto num lugar que outra pessoa roda de novo: um commit na `main` do seu repositório da atividade 3, ou um `~/consertar.sh` executável

Formato de cada linha do `monitor.log`: instante ISO com fuso, código de `/`, código de `/api/pokemons`, separados por espaço. Use `000` quando não houver resposta.

```
2026-10-02T23:03:11+00:00 200 200
2026-10-02T23:04:12+00:00 200 500
```

Formato do `laudo.md`:

```
camada=banco
hora=2026-10-03T14:48:00+00:00
sintoma=/api/pokemons passou a responder 500 e a raiz continuou 200
acao=refiz o deploy pelo pipeline, commit abc1234
```

O corretor também lê `~/sonda.log`, que é o nome que saiu no primeiro e-mail.

O campo `camada` é `banco`, `api` ou `front`. O campo `hora` é o instante ISO com fuso, como o seu monitor grava.

## Os 6 itens

1. O monitor gravando em `~/monitor.log`: no mínimo 60 linhas válidas, cobrindo 2 horas, com intervalo típico de até 3 minutos.
2. A falha aparece no monitor: uma linha `200 200` na meia hora anterior à falha e uma linha com erro nos 20 minutos seguintes.
3. A captura do momento da falha: no mínimo 5 pacotes na sua porta em `~/capturas/falha.pcap`, com pelo menos um entre 10 minutos antes e 2 horas depois da falha.
4. O laudo com a camada certa e a hora dentro de 10 minutos da falha, com `sintoma` e `acao` preenchidos.
5. O conserto é reproduzível: o front no ar foi criado depois da falha, e o conserto saiu de um push na `main` (com `DEPLOY_SHA` igual ao último commit) ou de um `~/consertar.sh` executável com os três `gr-docker run`.
6. A topologia íntegra no fim: os 8 itens da atividade 2 passam de novo.

Nota: proporcional aos itens cumpridos. 6 itens valem 10.

Um `gr-docker start` na mão resolve o serviço e não conta no item 5, porque não remonta a topologia. O item 6 olha só o estado final, então ele conta de qualquer jeito.

## Antes da janela abrir

- a topologia de pé, respondendo na sua porta
- o monitor rodando de um jeito que sobrevive à queda da sua sessão SSH
- o `tcpdump` gravando em arquivo, filtrado pela sua porta
- o conserto pronto: o workflow da atividade 3, ou o `~/consertar.sh`

Esta atividade não exige as anteriores concluídas. Se você não tem a topologia de pé, peça no canal `#atv-5-plantao`: o servidor monta uma para você, com as mesmas imagens e na sua faixa de portas, e os dados de acesso ficam em `~/atv5-ambiente.txt`. Quem não tem o pipeline verde entrega o conserto pelo `~/consertar.sh`.

## Como a camada aparece no monitor

```
/ 200   /api 500   → a camada do banco
/ 200   /api 000   → a camada da API
/ 000   /api 000   → a camada do front
```

Qual das falhas foi, dentro da camada, sai do log da API e de `gr-docker meus`. Olhe antes de consertar: o conserto apaga o sintoma.

## Quando algo não funciona

- O monitor mostra `000` nos dois códigos desde o começo: você está consultando `127.0.0.1`. Dentro do laboratório, o front está em `gr.lab.ayty.org:SUAPORTA`.
- O monitor morreu quando você fechou o terminal: use `nohup`, `cron` ou um serviço do systemd.
- O `/atividades` diz que a falha ainda não aconteceu: o seu minuto ainda não chegou, ou a sua topologia estava fora do ar e o servidor adiou.
- O botão "Explicar o item" no Discord tem o passo a passo de cada item, com os comandos.
- Outras dúvidas: `/ajuda problema texto: ...` no Discord.
