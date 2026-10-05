# Atividade 6: monitoramento com Uptime Kuma

- Prazo: terça-feira, 06/10/2026, ao fim da aula, às 12h
- Unidade: 2
- Onde: no servidor da disciplina, sobre a topologia da atividade 2
- Acompanhamento: `/atividades atividade:6` no Discord

Na atividade 5 você escreveu o monitor. Agora você usa um pronto, o Uptime Kuma, e compara o que cada um entrega.

Nota: proporcional aos itens cumpridos. 3 itens valem 10.

## Os 3 itens

1. O Uptime Kuma de pé, publicado numa porta livre da sua faixa, com os dados num volume seu montado em `/app/data`.
2. Dois monitores ativos e verdes: um HTTP(s) na raiz do seu front, e um HTTP(s) - Keyword na listagem, com uma palavra que apareça na resposta.
3. Uma queda registrada: você derruba o seu front, espera o Kuma marcar fora do ar, e sobe de novo.

## Subir o Kuma

```
gr-docker volume create kuma
gr-docker run louislam/uptime-kuma:1 --name kuma -p SUAPORTA:3001 -v kuma:/app/data
```

Use uma porta da sua faixa diferente da que o front já ocupa. Veja a sua faixa com `gr-docker ajuda`.

Abra `http://gr.lab.ayty.org:SUAPORTA` e crie a conta de administrador na primeira tela. Essa conta é sua, e a senha não é a de nenhum outro serviço da disciplina.

O volume existe pelo mesmo motivo da atividade 2: o Kuma guarda monitores e histórico num SQLite dentro de `/app/data`. Sem volume, isso morre com o container.

## Os dois monitores

O primeiro, tipo HTTP(s):

```
http://gr.lab.ayty.org:SUAPORTA_DO_FRONT/
```

O segundo, tipo HTTP(s) - Keyword:

```
http://gr.lab.ayty.org:SUAPORTA_DO_FRONT/api/pokemons
```

com uma palavra que apareça na listagem, por exemplo `Pikachu`.

A diferença entre os dois é o ponto da atividade. O primeiro só mostra que o nginx respondeu. O segundo prova que a API e o banco responderam, porque a palavra só aparece se o dado veio do banco.

Use intervalo de 60 segundos. O número de tentativas antes de marcar fora do ar fica a seu critério, e ele muda o resultado do item 3.

## A queda

```
gr-docker stop front
```

Olhe a tela do monitor até ele ficar vermelho e anote quanto tempo levou desde o comando. Depois:

```
gr-docker start front
```

O atraso entre a queda real e o alerta é o intervalo de checagem mais as tentativas que você configurou. Diminuir esse atraso custa mais requisições no serviço monitorado, e é essa troca que a aula discute.

## Quando algo não funciona

- O Kuma não abre: confira se a porta publicada é da sua faixa e se nada mais a ocupa, com `gr-docker meus`.
- O monitor fica vermelho desde o começo: use o endereço público `gr.lab.ayty.org:PORTA`, e não `localhost`. O Kuma roda num container, e `localhost` lá dentro é ele mesmo.
- O monitor de palavra-chave não encontra a palavra: abra a listagem no navegador e copie uma palavra que esteja mesmo na resposta.
- Outras dúvidas: `/ajuda problema texto: ...` no Discord.
