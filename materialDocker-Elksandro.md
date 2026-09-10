# Roteiro de Comandos Docker

**Seminário de José Elksandro do Nascimento Silva**
Gerência de Redes 2026.2 · DCX/UFPB · 10 de setembro de 2026

> Material produzido pelo aluno para o seminário da disciplina. Publicado aqui
> como referência da turma.
>
> **Nota sobre o servidor da disciplina:** os comandos abaixo são o Docker
> completo, como você usa na sua máquina. No servidor da turma o acesso é
> mediado pelo `gr-docker`, que aceita um subconjunto — veja
> [DOCKER.md](./DOCKER.md). Dentro do seu laboratório do CTF, vale tudo o que
> está aqui.

---

## 1. Imagens

Uma imagem é o molde: um pacote somente leitura com tudo que a aplicação precisa para rodar. Um container é a instância em execução dessa imagem.

`docker pull nginx:alpine`
Baixa uma imagem do Docker Hub para a máquina local, fixando a tag em vez de usar "latest".

`docker images`
Lista todas as imagens já baixadas na máquina local.

`docker rmi nginx:alpine`
Remove uma imagem específica do disco local.

`docker image prune`
Remove todas as imagens que não estão sendo usadas por nenhum container.

---

## 2. Containers

Container é uma instância de uma imagem Docker que executa um processo em isolamento. Ele possui estados (criado, rodando, parado etc.) e mantém seus dados enquanto existir. Ao ser removido, perde os dados que não estiverem em volumes ou bind mounts.

### 2.1 Criar e executar

`docker run -it ubuntu bash`
Cria um container a partir da imagem "ubuntu" e abre um terminal interativo dentro dele.

`docker run -d --name meu-site -p 8080:80 nginx:alpine`
Cria e executa um container em segundo plano, com nome definido e a porta 80 do container publicada na porta 8080 do host.

`docker run --rm -it ubuntu bash`
Executa um container temporário: ele é removido automaticamente ao ser encerrado.

### 2.2 Listar

`docker ps`
Lista os containers que estão em execução no momento.

`docker ps -a`
Lista todos os containers, incluindo os que já foram parados.

### 2.3 Controlar o ciclo de vida

`docker stop fb1e2bcb1bcf`
Para um container em execução, identificado pelo seu ID ou nome. O conteúdo interno não se perde.

`docker start fb1e2bcb1bcf`
Inicia novamente um container que já existe e está parado, preservando o que havia dentro dele.

`docker restart fb1e2bcb1bcf`
Reinicia um container em execução.

`docker rm fb1e2bcb1bcf`
Remove definitivamente um container parado - aqui sim, os dados internos são perdidos.

### 2.4 Acessar e depurar

`docker exec -it fb1e2bcb1bcf bash`
Abre um terminal dentro de um container que já está em execução.

`docker logs -f fb1e2bcb1bcf`
Acompanha, em tempo real, a saída padrão do container. É para lá que a aplicação escreve seus logs.

`docker inspect fb1e2bcb1bcf`
Exibe todos os metadados de um container em formato JSON: rede, volumes, etc.

---

## 3. Rede

Rede Docker permite comunicação entre containers e controla o isolamento de rede. Estar na mesma rede não significa estar exposto à internet.

`docker network create minha-rede`
Cria uma rede bridge personalizada. Containers conectados a ela podem se localizar pelo nome, via DNS interno do Docker.

`docker network ls`
Lista todas as redes existentes na máquina.

`docker network inspect minha-rede`
Mostra os detalhes de uma rede.

### 3.1 Conectar e desconectar um container de uma rede

**CLI**

```
docker network connect minha-rede fb1e2bcb1bcf
docker network disconnect minha-rede fb1e2bcb1bcf
```

**Compose**

```yaml
services:
  backend:
    image: minha-api:1.0
    networks:
      - minha-rede

networks:
  minha-rede:
```

Não se cria uma rede porque o Docker exige que containers tenham rede - eles já têm, por padrão, a rede `bridge`. Cria-se uma rede própria quando se quer **controlar a arquitetura da comunicação**: definir explicitamente quem precisa conversar com quem, em vez de colocar todos os containers juntos na rede padrão.

### 3.2 Duas coisas chamadas "bridge"

Vale separar dois conceitos que usam o mesmo nome:

```
bridge
├── nome da rede Docker padrão
│
└── bridge
    └── driver de rede
```

A rede padrão chamada `bridge` usa o driver `bridge`. Ao rodar `docker network create minha-rede`, por padrão também se está criando uma rede com o driver `bridge` - só que essa rede nova e a padrão **não são equivalentes em comportamento**, especialmente quanto a DNS.

### 3.3 DNS interno: nome do container em vez de IP

Na rede `bridge` padrão, **não** é possível contar com:

```
curl http://backend:8080
```

Seria necessário usar o IP diretamente:

```
curl http://172.17.0.2:8080
```

Já em uma rede criada pelo usuário, o Docker fornece DNS interno automaticamente:

```
docker network create minha-rede
docker run -d --name backend --network minha-rede nginx
docker run -it --network minha-rede curlimages/curl
```

e então:

```
curl http://backend:80
```

funciona direto pelo nome. Prefira sempre o nome, nunca o IP fixo, pois o IP pode mudar a cada reinício do container.

### 3.4 localhost dentro e fora do container

`localhost` significa **quem está fazendo a conexão**, não uma máquina fixa:

```
Aplicação rodando no HOST
  -> localhost = o HOST

Aplicação rodando DENTRO de um container
  -> localhost = o próprio container
```

Por isso, um backend em um container precisa apontar para `postgres:5432` (o nome do serviço do banco), nunca para `localhost:5432`, quando o PostgreSQL está rodando em outro container.

### 3.5 Rede externa compartilhada entre projetos

Quando uma rede precisa ser compartilhada por mais de um arquivo Compose, ela é criada uma vez fora de qualquer projeto e referenciada como externa nos dois:

`docker network create minha-rede`
Cria a rede manualmente, antes de qualquer `docker compose up`.

**Compose (em cada projeto que participa da rede)**

```yaml
networks:
  minha-rede:
    external: true
```

`external: true` avisa ao Compose que essa rede já existe e não deve ser criada nem removida por ele - apenas usada. Assim, um `docker compose down` em um projeto não derruba a rede nem afeta o outro projeto que também a utiliza.

### 3.6 Portas: -p, -P e EXPOSE

**CLI** `docker run -d -p 8080:80 nginx:alpine`

**Compose**

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
```

A sintaxe é sempre HOST:CONTAINER. A porta da esquerda é escolhida por quem sobe o container; a da direita é a porta em que o processo escuta lá dentro.

A porta interna pode se repetir entre containers diferentes sem problema. Já a porta do host não pode: duas publicações para a mesma porta externa entram em conflito.

### 3.7 Drivers de rede

| Driver | Ideia principal |
| --- | --- |
| `bridge` | Containers se comunicam dentro do mesmo host Docker (padrão para a maioria dos casos) |
| `host` | Container usa a rede do próprio host - não precisa de `-p` |
| `none` | Container fica sem rede alguma |
| `overlay` | Containers se comunicam entre hosts Docker diferentes (Swarm) |
| `macvlan` | Container aparece como um dispositivo próprio na rede local (LAN) |
| `ipvlan` | Parecido com macvlan, com tratamento diferente de MAC/interface |

```bash
docker network create --driver bridge minha-rede
docker run --network host nginx
docker run --network none nginx
```

### 3.8 Arquitetura comum

```
              INTERNET
                  |
                  v
                Nginx
               :80/:443
                  |
          +-------+-------+
          v               v
      Frontend          Backend
       :5173             :8080
                            |
                            v
                        Postgres
                          :5432
```

Normalmente, apenas o ponto de entrada (Nginx, ou um proxy reverso) recebe `-p`. Frontend, backend e banco ficam internos à rede Docker, alcançáveis pelos outros containers pelo nome, sem precisar de porta publicada.

---

## 4. Volumes

O container tem seu próprio sistema de arquivos, mas os dados podem precisar de um ciclo de vida diferente do container.

### 4.1 Volume - o Docker escolhe e gerencia

**CLI**

```
docker run -d -v postgres_data:/var/lib/postgresql/data postgres:17
docker volume inspect postgres_data
```

**Compose**

```yaml
services:
  postgres:
    image: postgres:17
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

O Docker decide onde o volume fica fisicamente no host - o caminho exato se descobre com `docker volume inspect`, não se escolhe diretamente.

### 4.2 Bind mount - você escolhe a pasta do host

**CLI**

```
docker run -it -v /home/usuario/projeto:/app ubuntu
```

**Compose**

```yaml
services:
  app:
    image: ubuntu
    volumes:
      - ./projeto:/app
```

A pasta do host e a pasta do container passam a ser a mesma coisa, não uma cópia. Um arquivo criado em `/app` dentro do container aparece imediatamente na pasta do host, e vice-versa. O caminho pode ser absoluto ou relativo (`.` funciona quando o comando é rodado de dentro da pasta do projeto). No Compose, o bind mount não precisa ser declarado na chave `volumes:` do final - o caminho `./projeto` já identifica diretamente uma pasta local.

### 4.3 tmpfs - armazenamento temporário na RAM

`docker run -d --tmpfs /tmp nginx:alpine`
Os dados gravados em `/tmp` ficam apenas na memória RAM e desaparecem quando o container é parado ou reiniciado. Útil para caches e arquivos temporários que não devem persistir.

### 4.4 Sem mount algum

Sem `-v` ou `--mount`, tudo que o container escreve fica na sua própria camada gravável - os dados pertencem ao ciclo de vida do container. Eles sobrevivem a um `docker stop`/`docker start`, porque o container ainda existe, mas somem definitivamente com `docker rm`. Com um volume, os dados continuam existindo mesmo depois de o container ser removido.

### 4.5 down e volumes

`docker compose down` normalmente remove containers e networks, mas **preserva volumes nomeados**. Já `docker compose down -v` remove os volumes também.

---

## 5. Dockerfile

Um Dockerfile é a receita para construir uma imagem: uma sequência de instruções de texto que o Docker executa, em ordem, para montar o sistema de arquivos final.

```docker
FROM ubuntu:24.04

RUN apt-get update
RUN apt-get install -y nginx

COPY . /app

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

`FROM` define a imagem base. `RUN` executa um comando durante a construção da imagem (por exemplo, instalar um pacote). `COPY` leva arquivos do host para dentro da imagem. `EXPOSE` documenta a porta usada pela aplicação. `CMD` é o comando executado quando o container é iniciado.

### 5.1 Construir, marcar e publicar

`docker build -t primeira-imagem:1.0 .`
Constrói uma imagem local a partir do Dockerfile presente no diretório atual (o ".").

`docker tag primeira-imagem:1.0 usuario/app:1.0`
Cria um apelido (tag) para a imagem, associando-a a um repositório remoto.

`docker push usuario/app:1.0`
Envia a imagem para o registry (Docker Hub, por exemplo), disponibilizando-a para outros hosts.

---

## 6. Docker Compose - o essencial

O Compose descreve múltiplos serviços em um único arquivo declarativo ("compose.yaml") e os orquestra com um só comando.

```yaml
services:
  backend:
    build: ./backend
    ports:
      - "3000:3000"
  db:
    image: postgres:17
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

`docker compose up -d`
Constrói e inicia todos os serviços definidos, em segundo plano.

`docker compose ps`
Lista o status de todos os serviços do projeto.

`docker compose logs -f`
Acompanha, em tempo real, os logs de todos os serviços.

`docker compose exec backend bash`
Abre um terminal dentro de um serviço específico já em execução.

`docker compose down`
Para e remove os containers e redes criados pelo Compose. Volumes nomeados são preservados.

`docker compose down -v`
Para e remove containers, redes e também os volumes nomeados.

### 6.1 Variáveis de ambiente

```yaml
services:
  backend:
    image: minha-api:1.0
    environment:
      - NODE_ENV=production
    env_file:
      - .env
```

`environment` define variáveis diretamente no arquivo. `env_file` aponta para um arquivo externo (.env).

---

## 7. Bônus

### 7.1 depends_on com condição

```yaml
services:
  backend:
    image: minha-api:1.0
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:17
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

Sem condição, `depends_on` só espera o container do banco iniciar e não que ele já aceite conexões. Com `condition: service_healthy`, o backend só sobe depois que o healthcheck do banco reportar sucesso pela primeira vez.

### 7.2 healthcheck

Define como o Docker verifica se um serviço está de fato saudável, além de apenas "rodando". O exemplo acima usa `pg_isready`; qualquer comando que retorne código de saída zero em caso de sucesso serve.

### 7.3 restart - as quatro opções

| Valor | Significado |
| --- | --- |
| `no` (padrão) | nunca reinicia sozinho |
| `always` | reinicia sempre que parar, sob qualquer motivo |
| `on-failure` | reinicia só se o processo terminar com erro (exit code de falha) |
| `unless-stopped` | reinicia sempre, exceto se alguém parou manualmente |

### 7.4 profiles

```yaml
services:
  grafana:
    image: grafana/grafana
    profiles:
      - monitoring
```

Um serviço com `profiles` só sobe quando o perfil correspondente é ativado explicitamente (`docker compose --profile monitoring up`). Um `docker compose up` comum ignora esse serviço - útil para partes opcionais do projeto, como observabilidade.

### 7.5 Rede externa compartilhada

```yaml
networks:
  shared:
    external: true
    name: shared-network
```

A rede já existe fora deste projeto, e o Compose apenas a referencia pelo nome real (`name`), sem tentar criá-la ou removê-la.
