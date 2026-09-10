# Docker no servidor da disciplina

**Gerência de Redes 2026.2 · DCX/UFPB**

Até agora você administrou **um** servidor: o seu laboratório. Nesta aula você
sobe um nível e passa a operar containers **no servidor de verdade** — o mesmo
que hospeda os portais, o banco de dados, o bot e os laboratórios de toda a
turma.

Você vai enxergar tudo o que roda ali. E vai poder mexer só no que é seu.

---

## Por que essa distinção existe

Em Docker, quem pode criar container pode virar root da máquina. Não é exagero:

```bash
docker run -v /:/host -it debian chroot /host    # e pronto: root no servidor
```

O container monta o disco inteiro do host e troca a raiz para dentro dele. Por
isso, **pertencer ao grupo `docker` equivale a ter root** — e é por isso que
nenhum aluno está nesse grupo.

O que vocês têm é um comando, o `gr-docker`, que roda com privilégio mas decide
o que pode. É o padrão que se usa em servidor compartilhado de verdade: em vez
de dar a chave da casa, dar uma porta com regra.

> Guarde esta ideia: **acesso não é "sim ou não", é "o quê"**. Boa parte do
> trabalho de administrar sistemas é desenhar essa fronteira.

---

## Entrando

```bash
ssh -i sua-chave gr-SEU-SLUG@gr.lab.ayty.org
```

Repare no que mudou em relação ao laboratório:

| | Laboratório | Servidor |
|---|---|---|
| usuário | `aluno` | `gr-seu-slug` |
| porta | a sua (22001, 22002…) | 22, a padrão |
| você é | dono da máquina | um usuário entre muitos |
| `sudo` | tudo | só o `gr-docker` |

A chave é a **mesma**. O que muda é quem você é do outro lado.

---

## Olhando o servidor

```bash
gr-docker ps          # o que está rodando AGORA, tudo
gr-docker ps -a       # incluindo o que está parado
gr-docker images      # imagens baixadas
gr-docker stats       # consumo de CPU e memória
```

Comece por aqui, e leia com atenção. Você vai ver:

- `portal-gr` e `portal-pa` — os portais das duas disciplinas
- `postgres` — o banco onde ficam os dados de vocês
- `gr-lab-*` — os 32 laboratórios do CTF, inclusive o seu
- `pa-*` — as aplicações publicadas pelos alunos de Projeto
- `caddy`, `umami`, `uptime-kuma` — infraestrutura de apoio

**Perguntas para responder olhando a saída:**

1. Quantos containers estão rodando? E parados?
2. Qual imagem se repete mais? Por quê?
3. Na coluna PORTS, o que significa `127.0.0.1:8500->8500/tcp`? E por que
   alguns não têm nada?
4. Algum container está consumindo muito mais memória que os outros?

A terceira é a mais importante: **`127.0.0.1` significa que a porta só é
alcançável de dentro do servidor.** É assim que o portal fica protegido — quem
chega de fora passa pelo Caddy, nunca direto.

---

## Criando o seu primeiro container

```bash
gr-docker run nginx:alpine --name meu-site -p 23000:80
```

Troque `23000` pela **primeira porta da sua faixa** — cada aluno tem dez, e a
sua está no e-mail de acesso.

O que aconteceu:

- o Docker baixou a imagem `nginx:alpine` do registro público
- criou um container a partir dela
- publicou a porta 80 de dentro na sua porta externa
- carimbou o container com o seu nome

Confira:

```bash
gr-docker meus
curl http://localhost:23000
```

E abra no navegador: `http://gr.lab.ayty.org:23000`

> **De novo a porta de dentro e a de fora.** O nginx escuta na **80**, dentro do
> container. Você chega nele pela **23000**, de fora. O `-p 23000:80` é a
> tradução — e é o mesmo conceito que travou dois colegas no CTF.

---

## Ciclo de vida

Um container não é um programa rodando: é um objeto com estados.

```bash
gr-docker stop meu-site       # para, mas não some
gr-docker ps -a               # veja: Exited
gr-docker start meu-site      # volta, com o mesmo conteúdo
gr-docker restart meu-site
gr-docker rm meu-site         # aí sim, some
```

**Experimente e observe:** pare o container, edite nada, inicie de novo. O que
sobreviveu? Agora apague e recrie. O que se perdeu?

Essa diferença — **parar preserva, apagar não** — é a base para entender
volumes, que é o próximo assunto.

---

## Entrando no container

```bash
gr-docker exec meu-site
```

Você cai num shell **dentro** do nginx. Explore:

```bash
ls /                      # é um sistema de arquivos inteiro
cat /etc/os-release       # qual distribuição?
ps aux                    # quantos processos existem aqui?
ls /usr/share/nginx/html  # a página servida
exit
```

**Repare:** dentro do container há pouquíssimos processos — talvez dois. Um
container não é uma máquina virtual: **não há init, não há systemd, não há
serviços de sistema**. É um processo isolado, com um sistema de arquivos
próprio, compartilhando o kernel do host.

É por isso que ele sobe em milissegundos, e é por isso que ele não substitui uma
VM em tudo.

---

## Logs

```bash
gr-docker logs meu-site
gr-docker logs meu-site --tail 20
```

Acesse a página pelo navegador e rode de novo: as requisições aparecem. Aplicação
em container escreve log na **saída padrão**, e o Docker guarda — em vez de cada
uma escrever num arquivo diferente.

---

## O que você pode e o que não pode

```bash
gr-docker rm portal-pa           # recusado: não é seu
gr-docker run alpine -v /:/host  # recusado: a flag daria o servidor
gr-docker run nginx -p 8080:80   # recusado: a porta não é sua
```

Cada recusa tem um motivo, e vale ler a mensagem em vez de só tentar outra coisa.

**Flags que ficam de fora e por quê:**

| Flag | O que daria |
|---|---|
| `-v /caminho` | acesso a arquivos do servidor |
| `--privileged` | praticamente root no host |
| `--network host` | a rede do servidor, sem isolamento |
| `--pid host` | ver e matar processos de todos |

**Dentro do seu laboratório do CTF você manda no que quiser** — lá o estrago
máximo é o próprio laboratório. Aqui, não.

Limites por aluno: **5 containers**, 512 MB e meio núcleo cada.

---

## Exercícios

**1. Reconhecimento.** Rode `gr-docker ps` e responda: quantos containers há, e
quantos publicam porta para fora (`0.0.0.0`) em vez de só para dentro
(`127.0.0.1`)? Por que a maioria é `127.0.0.1`?

**2. Suba um site seu.** Crie um nginx na sua porta, entre nele com `exec`, e
troque o `index.html` por uma página com o seu nome. Recarregue no navegador.

**3. Descubra o que se perde.** Pare e reinicie o container: a sua página
continua lá? Agora apague e recrie: continua? Explique o que aconteceu.

**4. Duas versões.** Suba `nginx:alpine` numa porta e `httpd:alpine` em outra.
Compare `gr-docker stats`: qual consome mais? Compare os logs.

**5. O erro proposital.** Tente `gr-docker run nginx -p 80:80`. Leia a mensagem
e explique, com suas palavras, por que essa porta não pode ser sua.

---

## Para levar da aula

**Container não é máquina virtual.** Compartilha o kernel, sobe em
milissegundos, e não roda um sistema inteiro — roda um processo.

**Imagem e container são coisas diferentes.** A imagem é o molde; o container é
a instância. Apagar o container não apaga a imagem, e é por isso que recriar é
rápido.

**A porta de dentro e a de fora nunca são a mesma coisa** — a não ser por
coincidência.

**Quem pode criar container pode virar root.** Toda a arquitetura do
`gr-docker` existe por causa dessa frase.

---

<sub>Dúvidas no canal da disciplina. `gr-docker ajuda` mostra o resumo dos
comandos a qualquer momento.</sub>
