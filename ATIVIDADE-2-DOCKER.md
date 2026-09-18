# Atividade 2 — Três camadas com Docker

- Prazo: sábado, 19/09/2026, às 20h
- Unidade: 1
- Nota: proporcional aos itens cumpridos. 8 itens valem 10.
- Onde: no servidor da disciplina, com `gr-docker`
- Acompanhamento: `/atividades atividade:2` no Discord
- Canal: `#atv-2-tres-camadas-com-docker`

Às 20h do prazo o servidor registra o estado dos seus containers. Conta o que estiver de pé nessa hora.

## O que você monta

Uma aplicação em três containers: um banco de dados, uma API e um front que lista pokémons.

```
navegador ──▶ front ──[ web ]── api ──[ dados, rede interna ]── banco
         porta publicada                                    volume
```

- Banco: `postgres:17-alpine`
- API: `gr-atv2-api:1`
- Front: `gr-atv2-front:1`

As três imagens já estão no servidor.

## Antes de começar

O acesso ao servidor da disciplina e o `gr-docker` estão explicados em [DOCKER.md](DOCKER.md).

`gr-docker ajuda` mostra a sua faixa de portas.

O `gr-docker` acrescenta o prefixo `gr-SEU-SLUG-` ao nome de containers, redes e volumes. Um container criado com `--name api` se chama `gr-SEU-SLUG-api`. Os comandos do `gr-docker` aceitam os dois nomes. As variáveis de ambiente precisam do nome completo.

## Variáveis de ambiente

Banco:

- `POSTGRES_PASSWORD`: uma senha escolhida por você
- `POSTGRES_DB`: `pokedex`

API:

- `POSTGRES_HOST`: nome completo do container do banco, por exemplo `gr-SEU-SLUG-banco`
- `POSTGRES_DB`: `pokedex`
- `POSTGRES_PASSWORD`: a mesma senha do banco

Front:

- `API`: nome completo do container da API, por exemplo `gr-SEU-SLUG-api`

A API cria a tabela e os pokémons na primeira vez que alcança o banco. Enquanto o banco não responde, ela tenta de novo a cada 2 segundos, por 1 minuto, e registra cada tentativa no log.

## Os 8 itens

### 1. As três camadas no ar

**O que é.** A atividade tem três camadas, cada uma no seu container: o banco (`postgres:17-alpine`), a API (`gr-atv2-api:1`) e o front (`gr-atv2-front:1`).

**Por que importa.** Separar não é burocracia. Cada camada tem um motivo diferente para reiniciar, uma superfície de ataque diferente e um dono diferente do problema quando algo quebra. Juntar tudo num container faz o front cair junto quando a API sobe uma versão nova.

**Faça.** Três `gr-docker run`, um por imagem. Confira com `gr-docker meus`.

**Cuidado com a ordem.** Suba o banco primeiro. A API espera por ele durante um minuto e loga cada tentativa, então subir fora de ordem não quebra nada — mas olhar o log ajuda a entender o que está acontecendo.

### 2. Só o front publicado, na sua faixa

**O que é.** Só o front tem porta publicada para a internet, na sua faixa.

**Por que importa.** Porta publicada é a única forma de alguém de fora chegar ao seu container. Cada uma que você abre é uma a mais para defender. O front precisa de uma; as outras duas camadas não precisam de nenhuma, porque quem fala com elas está dentro do servidor.

**Faça.** `-p SUA_PORTA:80` no front. A sua faixa aparece em `gr-docker ajuda`.

**Verifique de fora**, não de dentro: abra `http://gr.lab.ayty.org:SUA_PORTA` no navegador. Funcionar no servidor e não funcionar de fora é justamente o caso que interessa.

### 3. A API sem porta publicada

**O que é.** A API não tem porta publicada.

**Por que importa.** É a diferença entre alguém falar com a sua API e alguém falar com o seu front. Se a API estivesse publicada, qualquer pessoa na internet conversaria direto com ela, sem passar pelo que o front controla.

O navegador não precisa alcançar a API. Ele pede `/api/pokemons` ao front, e o nginx do front repassa por dentro da rede. Quem faz a chamada é o front, não o navegador.

**Faça.** Nenhum `-p` no `gr-docker run` da API.

**Confira** com `gr-docker meus`: a coluna PORTS da API deve mostrar `8000/tcp`, sem `0.0.0.0:` na frente. Com o `0.0.0.0:`, está publicada.

### 4. O banco sem porta publicada

**O que é.** O banco não tem porta publicada.

**Por que importa.** Postgres exposto na internet é um dos jeitos mais rápidos de perder um servidor. Varredura automática encontra a 5432 aberta em minutos e começa a tentar `postgres` com senhas comuns. Aqui a senha é fraca de propósito, então a única coisa que protege o banco é ele não estar alcançável.

**Faça.** Nenhum `-p` no `gr-docker run` do banco.

**Repare** que a API alcança o banco mesmo assim. Containers na mesma rede se falam pelo nome, sem publicar nada. Publicar é para quem vem de fora.

### 5. O banco numa rede interna

**O que é.** O banco fica numa rede criada com `--interna`, que não tem rota para a internet.

**Por que importa.** Publicar porta controla quem entra. A rede interna controla o outro sentido: quem sai. Se alguém conseguir executar código dentro do container do banco, sem saída para a internet ele não baixa ferramenta, não abre túnel e não manda os seus dados para fora. É a diferença entre um incidente e um vazamento.

**Faça.** `gr-docker network create dados --interna`, e suba o banco com `--network dados`.

**Teste.** De dentro do banco, um `wget` para qualquer endereço externo deve dar timeout. Se responder, a rede não está interna.

### 6. O front fora da rede do banco

**O que é.** O front não participa da rede do banco.

**Por que importa.** É o princípio do menor privilégio aplicado a rede. O front serve HTML e repassa `/api/`; ele não tem nenhum motivo para alcançar a 5432. Se amanhã aparecer uma falha no nginx, quem a explorar não encontra o banco do outro lado.

**Como fica.** A API é a única em duas redes: ela conversa com o front de um lado e com o banco do outro. É ela que decide o que passa entre os dois.

```
front  ──[ web ]──  api  ──[ dados (interna) ]──  banco
```

**Faça.** Suba a API numa rede e junte a outra depois:
`gr-docker network connect web api`.

### 7. A listagem funciona de ponta a ponta

**O que é.** A listagem de pokémons aparece no navegador, com dados vindos do banco.

**Por que importa.** É o item que prova que a topologia inteira está de pé. Os outros verificam peças isoladas; este exige que as três camadas se falem na ordem certa.

**Quando falha**, o front mostra o status HTTP e diz onde olhar. Um 502 quer dizer que o nginx não achou a API: ou o nome em `-e API=` está errado, ou os dois não estão na mesma rede.

**Investigue nesta ordem:**

1. `gr-docker logs front` — ele diz qual nome não resolveu
2. `gr-docker logs api` — ela loga cada tentativa de falar com o banco
3. `gr-docker inspect api` — em quais redes ela está

O log do front resolve a maioria dos casos.

### 8. Dados do banco num volume seu

**O que é.** Os dados do Postgres ficam num volume nomeado seu, montado em `/var/lib/postgresql/data`.

**Por que importa.** O que um container escreve no próprio sistema de arquivos morre com ele. Um `rm` para trocar de versão, um `--force-recreate`, uma atualização de imagem, e o banco volta vazio. Volume é o que separa o ciclo de vida dos dados do ciclo de vida do container.

**Faça.** `gr-docker volume create pgdata` e suba o banco com `-v pgdata:/var/lib/postgresql/data`.

**Prove para você mesmo.** Apague o container do banco, suba outro com o mesmo volume, e recarregue a página. A lista continua lá. Sem o volume, ela volta do zero.

**Repare** que montar um caminho do servidor (`-v /home/...`) não é a mesma coisa e não é aceito aqui: daria a você o disco do host.

## Quando algo não funciona

- Front mostra erro 502: o nginx do front não alcançou a API. Veja `gr-docker logs front`.
- API para depois de 1 minuto: ela não alcançou o banco. Confira `POSTGRES_HOST` e se API e banco estão na mesma rede.
- `gr-docker` recusa `-v /caminho`: aceita só volume nomeado. Crie antes com `gr-docker volume create`.
- Outras dúvidas: `/ajuda problema texto: ...` no Discord.
