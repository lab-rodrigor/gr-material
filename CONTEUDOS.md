# Tópicos e comandos de Gerência de Redes 2026.2, até 05/11/2026

Lista do que a turma executou, por tópico. A referência entre parênteses é a
atividade em que o item apareceu. Comando citado aqui é comando que está no
enunciado, no corretor ou na correção automática.

## 1. Acesso e operação do servidor Linux

### 1.1 Acesso por SSH (atividade 1)

- `ssh -i <chave> -p <porta> aluno@gr.lab.ayty.org`, com a chave em modo 600.
- Porta do host traduzida para o container: `SUA_PORTA` para a 22 e
  `SUA_PORTA+100` para a 2222.
- `scp` e `ssh-keygen -t ed25519` para gerar par próprio.

### 1.2 Inventário do que está no ar (atividade 1)

- `ss -lntup` para portas em escuta, com processo e usuário.
- `ps aux` para o processo por trás da porta.
- `systemctl stop`, `systemctl disable`, `apt purge` para remover `rpcbind` (111)
  e `vsftpd` (21).

### 1.3 Configuração do sshd (atividade 1)

- `sshd -T` para ler a configuração efetiva, em vez do arquivo.
- Diretivas: `PermitRootLogin`, `PasswordAuthentication`, `MaxAuthTries` até 3,
  `LoginGraceTime` até 60, `AllowTcpForwarding no`, `Port 2222`.
- `systemctl restart ssh` com sessão aberta em paralelo, para não se trancar fora.

### 1.4 Atualização de pacotes (atividade 1)

- `apt update` contra `apt upgrade`, e `apt-get -s upgrade` para simular.
- `/var/lib/apt/lists/` como prova de que as listas foram baixadas.
- `unattended-upgrades` e `/etc/apt/apt.conf.d/20auto-upgrades`, com
  `Unattended-Upgrade "1"`.

### 1.5 Firewall (atividades 1, 7, 8, 9)

- `ufw allow`, `ufw enable`, `ufw status verbose`, com a regra de SSH liberada
  antes de ativar.
- Porta aberta por atividade: 161/udp para o agente SNMP, saída 162/udp para o
  trap, 9116/tcp para o exportador, cada uma restrita à rede `172.30.0.0/16`.

### 1.6 fail2ban (atividade 1)

- Conceitos de filter, jail e action.
- Parâmetros `maxretry`, `findtime` e `bantime`.
- `fail2ban-client status sshd`, com `Total banned` maior que zero.
- `/var/log/auth.log` como fonte do filtro.
- Ataque ao próprio serviço a partir de outro laboratório, para provocar o ban.

### 1.7 Servidor web (atividade 1)

- `nginx` servindo `/var/www/html/index.html` próprio.
- `curl -I localhost` e leitura do cabeçalho `Server:`.

### 1.8 Contas e privilégio (atividade 1)

- `/etc/passwd` filtrado por UID maior ou igual a 1000.
- `/etc/sudoers.d/` e remoção da conta `suporte` com o sudoers dela.
- `lastlog` para último acesso por conta.

## 2. Containers e topologia

### 2.1 Ciclo de vida (atividades 2, 3, 5, 6, 10)

- `gr-docker run`, `ps`, `logs`, `inspect`, `stop`, `start`, que é o wrapper
  com o qual o aluno opera sem acesso ao daemon.
- Imagens usadas: `postgres:17-alpine`, `gr-atv2-api:1`, `gr-atv2-front:1`,
  `louislam/uptime-kuma:1`.

### 2.2 Portas (atividade 2)

- Publicação com `-p`, e a diferença entre porta publicada e porta exposta.
- Leitura de `0.0.0.0:` na coluna PORTS do `ps`.
- Front com porta publicada na faixa do aluno, API e banco sem porta publicada.

### 2.3 Redes (atividade 2)

- Rede `web` e rede `dados`, a segunda criada como interna, sem rota para fora.
- Resolução de nome entre containers pela rede.
- API nas duas redes e front fora da rede do banco, que é menor privilégio
  aplicado à topologia.
- `Internal` no `inspect` como prova do isolamento.

### 2.4 Dados (atividades 2, 3)

- Volume nomeado montado em `/var/lib/postgresql/data`.
- Volume preservado entre deploys, e bind mount recusado pelo `gr-docker`.
- Variáveis `POSTGRES_HOST`, `POSTGRES_DB`, `POSTGRES_PASSWORD` e `API`.
- Senha do Postgres gravada no volume na primeira subida, com
  `password authentication failed` como sintoma de senha trocada depois.

### 2.5 Diagnóstico (atividades 2, 5)

- 502 do nginx lido como nome não resolvido.
- Assinatura de sintoma por camada, descrita no tópico 8.

## 3. Entrega automatizada

### 3.1 GitHub Actions (atividade 3)

- `on: push` na `main` e `workflow_dispatch`.
- Bloco `env:` e leitura da aba Actions para depurar passo vermelho.
- No mínimo duas execuções verdes, com a última verde no fechamento.

### 3.2 Segredos e chave de entrega (atividade 3)

- Secrets `DEPLOY_KEY` e `DB_SENHA` no repositório.
- Chave SSH presa a um comando, que executa `gr-docker` e recusa shell.
- Nenhuma chave privada no histórico do git.

### 3.3 Deploy idempotente (atividades 3, 5)

- Derrubar antes de criar, tratar o recurso ausente e preservar o volume.
- `DEPLOY_SHA=${{ github.sha }}` no container do front, conferido contra o
  último commit da `main`.
- Conserto por push com `DEPLOY_SHA` novo, ou por `~/consertar.sh` com os três
  `gr-docker run`.

## 4. Captura e análise de tráfego

### 4.1 tcpdump (atividade 4)

- Interface e formato: `-i any`, `-i lo`, `-n`, `-nn`, `-A`, `-X`.
- Arquivo: `-w ~/capturas/<nome>.pcap` e `-r` para reler.
- Filtros por porta, por protocolo e por negação, como `not port 22`.
- Captura em segundo plano, com filtro na captura para não encher o disco.

### 4.2 Leitura de pacote (atividade 4)

- Aperto de mão completo em `[S]`, `[S.]` e `[.]`.
- `[S]` seguido de `[R]` como porta fechada, usado para achar a porta de destino.
- ICMP sem porta, com dado no payload.
- Consulta DNS na 53, com o nome perguntado visível.
- HTTP em texto puro, com cabeçalho e corpo legíveis.
- SNI em claro no ClientHello, antes da cifra.
- O que a cifra esconde e o que continua visível: destino, tamanho e tempo.

### 4.3 Wireshark (atividade 4 e seminário 4)

- Follow TCP Stream para remontar valor partido em dois pacotes.

## 5. SNMP, do gerente para o agente

### 5.1 Modelo (atividade 7)

- Papéis de agente e gerente, consulta por UDP na 161, sem conexão estabelecida,
  com tempo de espera e repetição.
- Árvore `1.3.6.1`, ramos `mgmt` e `private`, MIB-2 da RFC 1213, ramo
  `enterprises` por fabricante.
- Objeto e instância: `.0` para instância única, índice da linha em tabela.
- Tipos `STRING`, `INTEGER` com rótulo, `Gauge32`, `Counter32` e `Timeticks`.

### 5.2 Consulta (atividades 7, 9 e teste de 13/10)

- `snmpget -v2c -c <community> <alvo> sysDescr.0`
- `snmpwalk` sobre um ramo inteiro.
- `snmptable -Cb IF-MIB::ifTable`, com Index, Descr, Type, Mtu, Speed,
  AdminStatus, OperStatus, InOctets e OutOctets.
- `snmptranslate -On` para o OID numérico de um nome.
- Objetos cobrados no teste: `sysContact.0`, `sysServices.0`, `ifNumber.0`,
  `ifSpeed.1`, `ifSpeed.2`, `ifType.2`, `ifMtu.1`.

### 5.3 Contadores e taxa (atividade 7)

- Duas leituras de `ifInOctets.2` e a diferença dividida pelo intervalo.
- Estouro do contador de 32 bits e o `ifHCInOctets` de 64 bits.
- Diferença negativa descartada depois de reinício do agente.

### 5.4 Agente próprio (atividade 7)

- `/etc/snmp/snmpd.conf` com `agentaddress`, `rocommunity` restrita a
  `172.30.0.0/16` e `sysLocation` igual ao slug do aluno.
- `snmpd -Lf /dev/null` para subir em primeiro plano.
- `net-snmp-create-v3-user -ro -A <senha> -a SHA-512 -X <senha> -x AES aluno`.
- Níveis `noAuthNoPriv`, `authNoPriv` e `authPriv`.

### 5.5 Diagnóstico (atividade 7)

- `Timeout: No Response` como agente fora do ar ou porta fechada.
- `ss -lnup | grep 161` para confirmar a escuta.
- Linha `mibs :` em `/etc/snmp/snmp.conf` quando o nome não traduz.
- `tcpdump -i any -n -A udp port 161` mostrando a community da v2c em claro.

## 6. SNMP, do agente para o gerente

### 6.1 Notificação (atividade 8)

- `snmptrap -v2c -c alertas <receptor>:162 '' 1.3.6.1.4.1.8072.2.3.0.1 <OID> s <valor>`
- Varbind em três partes: OID, tipo e valor.
- Trap da v1 e `inform` da v2c, com confirmação de entrega no segundo.
- Conteúdo obrigatório da v2c: `sysUpTime.0` e `snmpTrapOID.0`.
- Notificações padrão `coldStart`, `linkDown` e `linkUp`.
- `trap2sink` no `snmpd.conf`, com o `coldStart` saindo na subida do agente.
- Limites: entrega sem confirmação e enxurrada em interface que oscila.

### 6.2 Métrica própria (atividade 8)

- Diretiva `extend` ligando um comando local a um OID.
- Leitura em `nsExtendOutputFull`, `nsExtendOutLine` e `nsExtendResult`.
- Métrica verificada contra `ps -e | wc -l` rodado no laboratório, com folga de 5.
- Custo de comando lento dentro da consulta.

## 7. Métricas, séries e painéis

### 7.1 Exportador (atividade 9)

- `prometheus-snmp-exporter` escutando na 9116, com `--config.file`.
- `snmp.yml` da v0.21.0, gerado a partir das MIBs, com módulo por tipo de
  equipamento.
- Módulo `if_mib` com `auth.community` e `walk: 1.3.6.1.2.1.2`.
- `curl 'localhost:9116/snmp?target=127.0.0.1&module=if_mib'` devolvendo série.
- Formato de exposição:
  `ifHCInOctets{ifAlias="",ifDescr="eth0",ifIndex="2"} 12345`.

### 7.2 Prometheus (atividades 9, 10)

- Coleta por HTTP a cada 30 segundos, com alvo refeito de hora em hora pelo
  `gr-prom-alvos`.
- `up{job="snmp-gr", aluno="<slug>"}` valendo 1.
- `count_over_time(ifHCInOctets{turma="gr", aluno="<slug>"}[10m])` com 2 pontos
  na atividade 9, e `[1h]` com 50 pontos na atividade 10.

### 7.3 Grafana (atividade 9)

- Painel "Redes 2026.2: interfaces por SNMP" em `otel.lab.ayty.org`.
- Entrada e saída por interface, estado operacional e tempo de resposta.
- Taxa só aparece com duas amostras, o que obriga a gerar tráfego.

### 7.4 Regra de alerta (atividade 11, assunto `alerta`)

- Cláusula `for` no arquivo de regras.
- Estados `inactive`, `pending` e `firing`, com o horário de cada transição.
- Avaliação periódica contra o tempo do `for`.

## 8. Disponibilidade e resposta a incidente

### 8.1 Monitor escrito pelo aluno (atividade 5)

- Uma linha por minuto, com instante ISO com fuso e os códigos de `/` e
  `/api/pokemons`, usando `000` para ausência de resposta.
- Arquivo `~/monitor.log` ou `~/sonda.log`, com no mínimo 60 linhas em 2 horas.
- Processo que sobrevive à sessão: `nohup`, `cron` ou unit do systemd.

### 8.2 Uptime Kuma (atividade 6)

- Container com volume em `/app/data`, guardando o SQLite.
- Monitor HTTP(s) na raiz e monitor Keyword na listagem.
- Intervalo de 60 segundos e número de tentativas antes de marcar fora do ar.
- Atraso do alerta igual ao intervalo mais as tentativas.
- Endereço público no monitor, porque `localhost` dentro do container é ele mesmo.
- Tabelas `monitor` e `heartbeat`, com a sequência status 0 e depois status 1.

### 8.3 Falha injetada (atividades 5, 10)

- Catálogo: parar o banco, `pause` no banco, desconectar o banco da rede
  interna, parar a API, parar o front.
- Minuto sorteado por aluno dentro de uma janela anunciada.

### 8.4 Diagnóstico por sintoma (atividade 5)

- `200/500` aponta a camada do banco.
- `200/000` aponta a camada da API.
- `000/000` aponta a camada do front.
- Investigar antes de consertar, porque o conserto apaga o sintoma.

### 8.5 Registro do incidente (atividades 5, 10)

- Laudo da atividade 5 com `camada`, `hora`, `sintoma` e `acao`.
- `~/incidente.md` da atividade 10 com `falha`, `camada`, `percebido`,
  `consertado` e `acao`, nessa ordem de tempo.
- Hora conferida contra a injeção real, com 10 minutos de folga.

### 8.6 Plantão automatizado (atividade 10)

- Detecção sem o aluno olhando, por qualquer caminho: métrica com `extend`,
  script que dispara `snmptrap`, Kuma com notificação ou coletor em Python.
- Aviso no receptor em até 5 minutos, com o nome da camada no corpo.
- Aviso enviado à mão não conta.

## 9. Assuntos sorteados da atividade 11

Cada aluno recebeu um assunto e a medição que precisa produzir. A ferramenta
central de cada um está na segunda coluna.

| Assunto | Ferramenta e medição |
|---|---|
| Latência e perda com tc/netem | `tc qdisc add dev <if> root netem delay 100ms loss 10%`, com RTT e perda medidos por `ping` ou `iperf3` antes e depois |
| NAT de saída com nftables | `nft` com tabela de NAT, e o endereço traduzido visto no `tcpdump` |
| SNMPv3 em authPriv | `snmpget -v3 -l authPriv`, com a mesma consulta falhando sem credencial |
| Zona DNS autoritativa com reversa | `dig @127.0.0.1 <zona> SOA +noall +comments +answer` e `dig -x <ip>`, com `AA` na resposta |
| Certificado próprio, SNI e cadeia | `openssl s_client -connect -servername`, com e sem a CA no truststore |
| Syslog remoto com retenção | mensagem gerada em um container aparecendo no coletor, com data |
| Alerta no Prometheus com `for` | o alerta em `pending` e depois em `firing`, com horário |
| Filtro BPF que isola um sintoma | contagem de pacotes com e sem o filtro, no mesmo pcap |
| Túnel WireGuard entre dois containers | `wg genkey`, `wg pubkey`, `ip link add wg0 type wireguard`, `wg set`, com ping dentro do túnel e o tráfego cifrado visto fora |
| Throughput sob concorrência | `iperf3 -s` e `iperf3 -c -P 2`, com a banda de um fluxo e de dois |

## 10. Git e GitHub

- Clone por SSH de repositório privado da organização `lab-rodrigor`, com chave
  cadastrada na conta do aluno (atividades 3 e 11).
- Repositório por aluno, criado a partir de template, com convite de
  colaborador (atividade 11).
- Commits ao longo do trabalho, usados como evidência de processo.
- Pull request de feedback nas atividades em formato de sala.

## 11. Estudo guiado por modelo de linguagem (atividade 11)

- `ROTEIRO.md` com objetivo em uma frase, 3 marcos e o comando que prova cada um.
- `PROVA.sh` executável, terminando em status 0, imprimindo a medição antes e
  depois.
- `CONVERSA.md` com o registro da conversa, e `APRENDI.md` com o erro encontrado.
- Verificação do que o modelo afirma contra o manual, o `--help` e a
  documentação oficial.
- Prova oral por escrito: 3 perguntas sobre o material do próprio aluno,
  respondidas por DM, sem consulta.

## 13. Tecnologias trabalhadas

| Tecnologia | Versão ou pacote | Onde apareceu |
|---|---|---|
| Debian | bookworm, no container de laboratório | todas |
| OpenSSH | `sshd`, chave ed25519 | atividades 1, 3, 11 |
| ufw | política padrão e faixa por atividade | 1, 7, 8, 9 |
| fail2ban | jail `sshd` | 1 |
| nginx | página própria e papel de front | 1, 2 |
| APT e unattended-upgrades | `20auto-upgrades` | 1 |
| Docker | pelo wrapper `gr-docker`, rede e volume | 2, 3, 5, 6, 10 |
| PostgreSQL | `postgres:17-alpine` | 2, 3 |
| GitHub Actions | workflow no repositório do aluno | 3 |
| Git e GitHub | organização `lab-rodrigor`, repositório por aluno | 3, 11 |
| tcpdump | captura e leitura de pcap | 4, 5, 7, 11 |
| Wireshark | Follow TCP Stream | 4 e seminário 4 |
| Net-SNMP | `snmpd`, `snmpget`, `snmpwalk`, `snmptable`, `snmptrap`, `snmptranslate` | 7, 8, 10 |
| snmptrapd | receptor de traps do servidor | 8, 10 |
| prometheus-snmp-exporter | v0.21.0, porta 9116 | 9, 10 |
| Prometheus | coleta, PromQL, regra com `for` | 9, 10, 11 |
| Grafana | painel em `otel.lab.ayty.org` | 9 |
| Uptime Kuma | `louislam/uptime-kuma:1`, SQLite em `/app/data` | 6 |
| systemd, cron e nohup | processo que sobrevive à sessão | 5 |
| tc e netem | controle de latência e perda | 11 |
| nftables | NAT de saída | 11 |
| BIND | zona autoritativa com reversa | 11 |
| OpenSSL | certificado próprio, SNI e cadeia | 11 |
| syslog e journald | log remoto com retenção | 11 e seminário 5 |
| BPF | filtro de captura | 11 |
| WireGuard | túnel entre dois containers | 11 |
| iperf3 | throughput e concorrência | 11 |
| Zabbix | só no seminário 7 | nenhuma atividade |
| Gemini | guia de estudo da conta `@dcx.ufpb.br` | 11 |
| Discord | prova por DM e canal por atividade | 11 e avisos |

## 12. Seminários

| # | Tema | Data |
|---|---|---|
| 1 | Linux básico | 08/09 |
| 2 | Docker básico | 10/09 |
| 3 | Pipeline e GitHub Actions | 15/09 |
| 4 | Análise de tráfego com tcpdump e Wireshark | 24/09 |
| 5 | Logs centralizados: syslog, journald, retenção | 29/09 |
| 6 | SNMP: agente, MIB, OID, v2c e v3 | 06/10 |
| 7 | Monitoramento com Zabbix | 13/10 |
| 8 | Prometheus e Grafana | 03/11 |

Marcados: proxy reverso e TLS em 10/11, backup e restauração em 17/11, Ansible
em 24/11. Nenhum dos três apareceu em atividade até aqui.
