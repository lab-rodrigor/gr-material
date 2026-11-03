# Aprender um assunto novo usando a IA

Cada aluno tem um repositório privado, criado pelo servidor, com o assunto sorteado em `SEU-ASSUNTO.md`. O link chegou por DM.

Use o Gemini da sua conta `@dcx.ufpb.br` como guia de estudo. Não precisa de chave nem de cartão.

A atividade tem três etapas, na ordem abaixo.

## Etapa 1: roteiro de estudo

Escreva o `ROTEIRO.md` com:

- o objetivo em uma frase
- 3 marcos
- para cada marco, o comando que prova o marco

Marco sem comando que o prove não conta.

## Etapa 2: prova no seu ambiente

Escreva o `PROVA.sh`, executável, que roda sozinho e imprime a medição pedida em `SEU-ASSUNTO.md`, antes e depois. Eu executo esse arquivo dentro do seu laboratório. Ele precisa terminar com status 0.

## Etapa 3: perguntas por DM

Amanhã, quarta 04/11, às 20h, eu mando 3 perguntas por DM. Elas saem do seu `ROTEIRO.md` e dos números que o seu `PROVA.sh` imprimiu, no que estiver commitado nesse horário. Você tem dez minutos para responder.

## O que eu verifico

- `ROTEIRO.md` com 3 marcos e um comando por marco
- `PROVA.sh` terminando com status 0 e imprimindo as duas medições
- `CONVERSA.md` com a conversa inteira: pergunta e resposta, da primeira à última
- `APRENDI.md` com o que você aprendeu, uma afirmação errada ou desatualizada do Gemini, e o comando ou a página oficial que mostrou o erro
- commits ao longo do trabalho
- as respostas das 3 perguntas

Pesos: roteiro 20, prova 30, perguntas 30, conversa e `APRENDI.md` 20.

## Como usar a IA aqui

Peça plano de estudo, explicação e exercício. Mande a sua tentativa para ser corrigida. Confira no manual, no `--help` ou na documentação oficial o que o modelo afirmar.

Os assuntos foram escolhidos em pontos onde os modelos respondem com a ferramenta antiga ou com o padrão de uma versão que já mudou. Achar um desses erros e provar que é erro vale ponto no `APRENDI.md`.

Resposta pronta colada no `PROVA.sh` trava na etapa 3, porque as perguntas vêm dos seus próprios números.

## Entrega

Commit e push neste repositório, que é só seu. Prazo das etapas 1 e 2: hoje, até 23h59. Unidade 3.

## Ambiente

Clone o repositório dentro do seu laboratório, em `~/aprender`, e trabalhe lá.

```
git clone https://github.com/lab-rodrigor/aprender-gr-SEU-USUARIO.git aprender
cd aprender
```

Tudo acontece dentro do laboratório, que já tem `tc`, `nft`, `iptables`, `snmpd`, `dig`, `openssl`, `tcpdump`, saída para a internet e `sudo` sem senha. Falta instalar o que o seu assunto pedir, com `apt-get install`.

Não mexa na topologia da atividade 10, que roda na sua conta do servidor: a falha dela entra entre hoje às 8h e quinta às 20h, e o que você alterar lá eu não consigo separar do que eu injetei.
