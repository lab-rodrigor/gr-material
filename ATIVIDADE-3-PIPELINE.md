# Atividade 3 — Pipeline

- Prazo: sábado, 26/09/2026, às 20h
- Unidade: 2
- Nota: proporcional aos itens cumpridos. 6 itens valem 10.
- Repositório: `atividade3-pipeline-SEU-EMAIL`, na organização lab-rodrigor
- Acompanhamento: `/atividades atividade:3` no Discord

Você já montou a topologia à mão na atividade 2. Agora quem monta é o GitHub Actions, a cada push na `main`.

## Como o pipeline entra no servidor

Você recebe por e-mail uma chave de deploy. Ela entra no servidor presa a um comando: só executa `gr-docker`, e nada mais. Shell, `cat`, `rm` são recusados.

```
ssh -i chave gr-SEU-SLUG@gr.lab.ayty.org gr-docker ps      → funciona
ssh -i chave gr-SEU-SLUG@gr.lab.ayty.org                   → recusado
```

O pipeline tem o mesmo poder que você tem no terminal, nem mais.

## Os secrets

Em **Settings → Secrets and variables → Actions**, no seu repositório:

- `DEPLOY_KEY`: o conteúdo inteiro do arquivo da chave privada, incluindo as linhas `BEGIN` e `END`
- `DB_SENHA`: a senha do Postgres

A chave nunca vai para dentro do repositório. Ele é público.

## A senha do banco vem do volume

O Postgres grava a senha na primeira vez que inicializa o volume. Depois disso, `POSTGRES_PASSWORD` não muda nada: vale a senha antiga.

Se você criou o `pgdata` na atividade 2, `DB_SENHA` tem de ser a senha que você usou lá. A API vai recusar a conexão com qualquer outra, e o log dela diz `password authentication failed`.

Para recomeçar do zero: `gr-docker volume rm pgdata` apaga o volume e os dados junto.

## O que completar

O arquivo é `.github/workflows/deploy.yml`, e os passos estão marcados com TODO:

1. Derrubar a topologia antiga
2. Criar as duas redes e o volume
3. Subir banco, API e front
4. Conferir que a listagem responde

Preencha também `SLUG` e `PORTA` no bloco `env:` do topo.

## Rodar duas vezes é a atividade

O primeiro deploy encontra o servidor limpo. O segundo encontra tudo de pé: nome de container em uso, rede que já existe, volume que já existe. É aí que o script ingênuo quebra.

Pense no que deve sobreviver a um deploy. O volume do banco guarda os dados dos pokémons.

## Os 6 itens

### 1. Workflow que dispara em push na main

**O que é.** O seu repositório tem um workflow em `.github/workflows/` que dispara em push na `main`.

**Por que importa.** Pipeline que só roda quando alguém clica é script com outro nome. O que muda o jeito de trabalhar é a entrega acontecer sozinha, no mesmo gesto de publicar o código.

**Faça.** No template já existe `deploy.yml` com o `on: push` pronto. Complete os passos marcados com TODO.

**Repare** que `workflow_dispatch` também está lá: ele deixa você rodar à mão enquanto depura, sem precisar de um commit vazio.

### 2. A última execução terminou verde

**O que é.** A última execução do workflow na `main` terminou verde.

**Por que importa.** Vermelho no fim do pipeline significa que o que está publicado não é o que está no repositório. Quanto mais tempo assim, menos confiança o pipeline tem.

**Investigue.** Aba **Actions** do seu repositório, última execução, passo vermelho. A saída do passo diz o que o servidor respondeu.

**Erros comuns.** A chave não chegou no formato certo em `secrets.DEPLOY_KEY`; o slug em `env:` não é o seu; a porta está fora da sua faixa.

### 3. Rodou verde pelo menos duas vezes

**O que é.** O workflow rodou e terminou verde pelo menos duas vezes.

**Por que importa.** É aqui que a atividade deixa de ser um `docker run`. O primeiro deploy sobe numa máquina limpa. O segundo encontra tudo de pé, e é nele que o script ingênuo quebra: nome de container em uso, rede que já existe, volume que já existe.

**Faça.** Antes de criar, derrube o que existe. Trate o caso em que não existe nada: `gr-docker rm` de um container ausente devolve erro, e o passo inteiro falha junto.

**Pense** no que deve sobreviver a um deploy. O volume do banco guarda os dados; apagá-lo a cada push transforma a sua aplicação numa que esquece tudo quando você publica.

### 4. A topologia de pé, como na atividade 2

**O que é.** A topologia da atividade 2 está de pé: três camadas, banco em rede interna, só o front publicado, dados em volume.

**Por que importa.** O pipeline entrega a mesma coisa que você entregou à mão. Se o que sobe agora é diferente, o pipeline não automatizou a atividade 2, fez outra coisa.

**Veja item a item** com `/atividades atividade:2`. Os 8 itens de lá valem aqui.

### 5. O front roda o commit que o pipeline publicou

**O que é.** O container do front roda com `DEPLOY_SHA` igual ao último commit da `main`.

**Por que importa.** É o que liga o que está no ar ao que está no repositório. Numa produção de verdade, é assim que você responde "qual versão está rodando?" sem adivinhar.

**Faça.** No passo que sobe o front:

```
-e DEPLOY_SHA=${{ github.sha }}
```

**Repare** que isso também é a prova da atividade: montar à mão não produz o sha do commit que o Actions estava publicando.

### 6. A chave privada fora do repositório

**O que é.** A chave privada de deploy não está commitada no repositório.

**Por que importa.** Segredo em repositório continua no histórico depois de apagado, e qualquer pessoa com acesso ao repositório vira dona dele. Esta chave abre o seu usuário no servidor da disciplina.

**Faça.** A chave vive em **Settings → Secrets and variables → Actions**, com o nome `DEPLOY_KEY`. No workflow ela aparece como `${{ secrets.DEPLOY_KEY }}`, e o GitHub esconde o valor no log.

**Se você já commitou**, trocar o arquivo não basta: avise o professor para gerar outra chave.

## Quando algo não funciona

- Job vermelho sem nenhum passo: o problema é da organização, não seu. Avise no canal.
- `Permission denied (publickey)`: o secret `DEPLOY_KEY` não tem o conteúdo inteiro do arquivo.
- `password authentication failed` no log da API: `DB_SENHA` não é a senha que criou o volume.
- 502 no front: o nginx não achou a API. Veja `gr-docker logs front`.
- Outras dúvidas: `/ajuda problema texto: ...` no Discord.
