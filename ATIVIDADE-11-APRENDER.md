# Atividade: CTF de aprendizado

Você vai aprender um assunto que não vimos em aula, usando a IA como guia de estudo. O assunto foi sorteado para você e está na sua DM. O do seu colega é outro.

A flag não está na IA. Ela sai do seu ambiente: o corretor roda o que você construiu e, quando a medição aparece, imprime a flag. Nenhum modelo consegue entregar essa string, porque ela depende da sua matrícula e do seu ambiente funcionando.

Use o Gemini da sua conta `@dcx.ufpb.br`. Não precisa de chave nem de cartão.

Parte da nota é o que você aprendeu. Parte é como você usou a IA, e isso eu leio na conversa que você entrega.

## As três flags

**Flag 1, roteiro.** Use a IA para montar o `ROTEIRO.md`: o objetivo em uma frase, 3 marcos e, para cada marco, o comando que prova o marco. Comando, não descrição. Se um marco não tem como ser testado, ele não é um marco.

**Flag 2, prova técnica.** Escreva o `PROVA.sh`, executável, que roda sozinho e imprime a medição que a sua DM pede, antes e depois. O corretor executa o seu script no seu ambiente. Saída zero e as duas medições liberam a flag.

**Flag 3, prova sem IA.** Às 11h40, eu mando 3 perguntas por DM. Elas saem do seu próprio `ROTEIRO.md` e dos números que o seu `PROVA.sh` imprimiu. Nenhuma IA responde por você, porque nenhuma IA viu o seu ambiente. Dez minutos para responder.

## Como eu avalio o uso da IA

Você entrega a conversa inteira em `CONVERSA.md`, pergunta e resposta, da primeira à última. Eu procuro 4 coisas nela e no `APRENDI.md`:

- você pediu plano e exercício, em vez de resposta pronta
- você mandou a sua própria tentativa para ser corrigida, pelo menos uma vez
- você conferiu alguma afirmação do modelo contra fonte primária: manual, `--help`, documentação oficial
- você achou uma afirmação errada ou desatualizada do modelo, e registrou qual, com o comando ou a página que mostrou o erro

O último item vale ponto e costuma acontecer. Os assuntos do sorteio foram escolhidos em pontos onde os modelos respondem com a ferramenta antiga ou com o padrão de uma versão que já mudou.

## Entrega

Três arquivos em `~/aprender/`, dentro do seu laboratório:

- `ROTEIRO.md`
- `PROVA.sh`, executável
- `CONVERSA.md`: a conversa inteira com o Gemini, colada
- `APRENDI.md`: o que você aprendeu, o erro que você achou no modelo, e como você provou que era erro

## Regras

- Conversa sem a sua condução não vale nota. Pedir explicação, exercício e correção da sua tentativa vale.
- Conversa que não corresponde ao que está no seu `PROVA.sh` eu trato como entrega de outra pessoa.
- Assunto de colega não serve. O corretor confere o seu contra a sua matrícula.
- Prazo das flags 1 e 2: hoje, até 23h59. Unidade 3.

## Ambiente

A IA é o Gemini da sua conta institucional. Entre com o endereço `@dcx.ufpb.br`.

O `PROVA.sh` roda dentro do seu laboratório. Containers de apoio vão em uma rede própria, com nome `aprender-<seu slug>`. Não use a topologia da atividade 10: a falha dela entra entre hoje às 8h e quinta às 20h, e o que você mexer lá eu não consigo separar do que eu injetei.

## Esqueleto

Para nao perder tempo com nome de arquivo:

```
curl -fsSLO https://raw.githubusercontent.com/lab-rodrigor/gr-material/main/aprender-iniciar
sh aprender-iniciar
```

Ele cria os quatro arquivos vazios em `~/aprender/` e nao sobrescreve o que voce ja escreveu. Na sua conta do servidor o comando `aprender-iniciar` ja esta instalado, e basta rodar o nome.
