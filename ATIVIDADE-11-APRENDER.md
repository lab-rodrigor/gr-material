# Atividade: CTF de aprendizado

Você vai aprender um assunto que não vimos em aula, usando a IA como guia de estudo. O assunto foi sorteado para você e está na sua DM. O do seu colega é outro.

A flag não está na IA. Ela sai do seu ambiente: o corretor roda o que você construiu e, quando a medição aparece, imprime a flag. Nenhum modelo consegue entregar essa string, porque ela depende da sua matrícula e do seu ambiente funcionando.

Metade da nota é o que você aprendeu. A outra metade é como você usou a IA, e eu leio as conversas pelo proxy da turma.

## As três flags

**Flag 1, roteiro.** Use a IA para montar o `ROTEIRO.md`: o objetivo em uma frase, 3 marcos e, para cada marco, o comando que prova o marco. Comando, não descrição. Se um marco não tem como ser testado, ele não é um marco.

**Flag 2, prova técnica.** Escreva o `PROVA.sh`, executável, que roda sozinho e imprime a medição que a sua DM pede, antes e depois. O corretor executa o seu script no seu ambiente. Saída zero e as duas medições liberam a flag.

**Flag 3, prova sem IA.** Às 11h40, eu mando 3 perguntas por DM sobre o seu próprio roteiro. Dez minutos para responder, sem IA: a sua chave fica suspensa nesse intervalo.

## Como eu avalio o uso da IA

Eu procuro 4 coisas no log das suas conversas e no `APRENDI.md`:

- você pediu plano e exercício, em vez de resposta pronta
- você mandou a sua própria tentativa para ser corrigida, pelo menos uma vez
- você conferiu alguma afirmação do modelo contra fonte primária: manual, `--help`, documentação oficial
- você achou uma afirmação errada ou desatualizada do modelo, e registrou qual, com o comando ou a página que mostrou o erro

O último item vale ponto e costuma acontecer. Os assuntos do sorteio foram escolhidos em pontos onde os modelos respondem com a ferramenta antiga ou com o padrão de uma versão que já mudou.

## Entrega

Três arquivos em `~/aprender/`, dentro do seu laboratório:

- `ROTEIRO.md`
- `PROVA.sh`, executável
- `APRENDI.md`: o que você aprendeu, o erro que você achou no modelo, e como você provou que era erro

## Regras

- A chave de LLM é sua, com orçamento próprio. Não passe para colega: o consumo fica no log com o seu nome.
- Pedir a resposta pronta aparece no log e não vale nota. Pedir explicação, exercício e correção vale.
- Assunto de colega não serve. O corretor confere o seu contra a sua matrícula.
- Prazo das flags 1 e 2: hoje, até 23h59. Unidade 3.

## Ambiente

Proxy da turma: `http://127.0.0.1:4000/v1`, de dentro do servidor. Modelos: `gpt-4o-mini`, `gpt-4.1-mini`, `gpt-4o`.

O `PROVA.sh` roda dentro do seu laboratório. Containers de apoio vão em uma rede própria, com nome `aprender-<seu slug>`. Não use a topologia da atividade 10: a falha dela entra entre hoje às 8h e quinta às 20h, e o que você mexer lá eu não consigo separar do que eu injetei.
