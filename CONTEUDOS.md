# Conteúdos de formação de Gerência de Redes 2026.2, até 05/11/2026

O que o aluno aprendeu, por área, com o comando que exercitou cada item.
Ficaram de fora as regras de operação da disciplina: portas e endereços do
laboratório, limites usados na correção, scripts do servidor e pontuação das
atividades.

## 1. Operação de um servidor Linux

- Acesso por SSH com chave, com a chave privada em modo 600.
- Geração de par de chaves com `ssh-keygen -t ed25519`.
- Inventário do que está em escuta com `ss -lntup`, e o processo por trás da
  porta com `ps aux`.
- Remoção de serviço desnecessário, com `systemctl stop`, `systemctl disable` e
  `apt purge`.
- Atualização de pacotes, com a diferença entre `apt update` e `apt upgrade`, e
  simulação por `apt-get -s upgrade`.
- Atualização automática com `unattended-upgrades`.
- Auditoria de contas em `/etc/passwd`, `/etc/sudoers.d/` e `lastlog`, com
  remoção de conta indevida.
- Superfície de exposição, segurança por obscuridade, rastreabilidade por conta
  nominal e a janela entre a publicação de uma correção e a aplicação dela.

## 2. Endurecimento de serviço

- Configuração efetiva do sshd lida com `sshd -T`, em vez do arquivo.
- `PermitRootLogin`, `PasswordAuthentication`, `MaxAuthTries`, `LoginGraceTime`,
  `AllowTcpForwarding` e troca da porta de escuta.
- Ordem de operação que evita perder o acesso: liberar antes de restringir, e
  manter uma sessão aberta ao reiniciar o sshd.
- Firewall com `ufw`: política padrão, regra por porta e protocolo, e
  `ufw status verbose`.
- Abertura restrita a uma faixa de rede, em vez de abertura geral.
- `fail2ban` com filter, jail e action, e os parâmetros `maxretry`, `findtime` e
  `bantime`.
- Leitura de `/var/log/auth.log` como fonte do filtro, e verificação do
  banimento com `fail2ban-client status sshd`.
- Teste do próprio bloqueio, atacando o serviço a partir de outra máquina.

## 3. Containers e topologia de aplicação

- Aplicação dividida em camadas, uma por container: banco, API e front.
- Publicação de porta e a diferença entre porta publicada e porta exposta.
- Rede interna sem rota para a internet, e resolução de nome entre containers.
- Menor privilégio aplicado à topologia: a camada que não precisa falar com o
  banco fica fora da rede dele.
- Volume nomeado, com o ciclo de vida dos dados separado do container.
- Configuração por variável de ambiente.
- Diagnóstico por log e por inspeção do container, com o 502 do proxy lido como
  nome não resolvido.
- Senha gravada no volume na primeira inicialização do banco, e o erro de
  autenticação que aparece quando a senha muda depois.

## 4. Entrega automatizada

- Pipeline disparado por push na branch principal, e disparo manual.
- Segredo guardado fora do repositório, com o valor injetado no pipeline.
- Chave SSH presa a um comando, que executa só a operação de deploy.
- Deploy idempotente: derrubar antes de criar, tratar o recurso ausente e
  preservar o volume de dados.
- Versão em produção ligada ao commit que a gerou.
- Leitura da execução do pipeline para achar o passo que falhou.

## 5. Captura e análise de tráfego

- `tcpdump` com `-i any`, `-n`, `-nn`, `-A`, `-X`, `-w` e `-r`.
- Filtro por porta, por protocolo e por negação, e por que filtrar na captura.
- Aperto de mão TCP em `[S]`, `[S.]` e `[.]`, e porta fechada em `[S]` seguido
  de `[R]`.
- HTTP em texto puro, com cabeçalho e corpo legíveis, e o mesmo risco para token
  e cookie sem TLS.
- Consulta DNS na porta 53, com o nome perguntado visível antes de qualquer
  conexão cifrada.
- ICMP sem porta, com dado no payload.
- SNI em claro no ClientHello, e a distinção entre o que a cifra esconde e o que
  continua visível: destino, tamanho e tempo.
- Remontagem de fluxo TCP, com Follow TCP Stream, quando o valor vem partido em
  dois pacotes.
- Interface de captura: por que `-i eth0` perde o tráfego que passa por `lo`.
- Filtro BPF para isolar um sintoma dentro de uma captura.

## 6. SNMP: consulta

- Papéis de agente e de gerente, e consulta por UDP sem conexão estabelecida,
  com tempo de espera e repetição.
- OID e MIB: a árvore, os ramos de padrão e de fabricante, e a tradução de
  número para nome pelos arquivos de MIB.
- Objeto e instância, com `.0` para instância única e índice para linha de
  tabela.
- Tipos de valor: `STRING`, `INTEGER` com rótulo, `Gauge32`, `Counter32` e
  `Timeticks`.
- `snmpget`, `snmpwalk`, `snmptable` e `snmptranslate -On`.
- Tabela de interfaces da `IF-MIB`, com descrição, estado administrativo, estado
  operacional e contadores de entrada e saída.
- Cálculo de taxa a partir de duas leituras de um contador, dividido pelo
  intervalo.
- Estouro do contador de 32 bits, o contador de 64 bits e o descarte da
  diferença negativa depois de um reinício.
- Configuração do próprio agente: endereço de escuta, community restrita a uma
  rede e identificação do equipamento.
- Versões v1, v2c e v3, com os níveis `noAuthNoPriv`, `authNoPriv` e `authPriv`.
- Criação de usuário v3 com autenticação SHA e cifra AES.
- Community da v2c visível em captura, e a mesma consulta ilegível na v3.
- Diagnóstico: `Timeout: No Response`, porta de escuta e MIB ausente.

## 7. SNMP: notificação e métrica própria

- Notificação saindo do agente para o receptor, em porta separada da consulta.
- Trap da v1 e `inform` da v2c, com a diferença na confirmação de entrega.
- Conteúdo de uma notificação: tempo de atividade, identificador do evento e
  varbinds escolhidos por quem envia.
- Notificações padrão de partida e de mudança de estado de interface.
- Envio manual com `snmptrap`, com o varbind em OID, tipo e valor.
- Notificação automática na subida do agente, configurada no arquivo do serviço.
- Limites da notificação: entrega sem confirmação e enxurrada de avisos quando a
  interface oscila.
- Métrica própria publicada por `extend`, ligando um comando local a um OID.
- Custo de comando lento dentro da consulta.
- Varredura e notificação usadas juntas, com o atraso de descoberta de cada uma.

## 8. Métricas, séries temporais e painéis

- Modelo de coleta por exportador: o alvo expõe, o coletor busca em intervalo
  fixo e guarda a série.
- Exportador que traduz SNMP para o formato de exposição do coletor.
- Arquivo de módulos do exportador, gerado a partir das MIBs, com o ramo a
  percorrer e a credencial de consulta.
- Formato de exposição com rótulos por interface.
- Consulta à série temporal para saber se o alvo está sendo coletado e quantos
  pontos existem na janela.
- Painel com entrada e saída por interface, estado operacional e tempo de
  resposta do agente.
- Por que a taxa só aparece com pelo menos duas amostras.
- Regra de alerta com `for`, e os estados `inactive`, `pending` e `firing`.

## 9. Monitoração de disponibilidade

- Monitor escrito pelo próprio aluno, com uma linha por verificação, instante em
  ISO com fuso e o código de resposta de cada camada.
- Código reservado para ausência de resposta, distinto de um erro do servidor.
- Processo que sobrevive ao fim da sessão: `nohup`, cron ou unit do systemd.
- Ferramenta pronta de monitoração, com monitor de requisição e monitor de
  palavra-chave na resposta.
- Intervalo de checagem e número de tentativas antes de declarar fora do ar.
- Atraso do alerta igual ao intervalo mais as tentativas, e o custo em
  requisições de encurtar esse atraso.
- Endereço público no monitor, porque `localhost` dentro do container do monitor
  é ele mesmo.
- Histórico de batimentos como prova de queda e de volta.

## 10. Resposta a incidente

- Diagnóstico pela assinatura do sintoma, com o par de códigos de duas camadas
  apontando qual delas caiu.
- Investigar antes de consertar, porque o conserto apaga o sintoma.
- Captura do momento da falha, filtrada pela porta do serviço.
- Registro do incidente com hora da falha, camada, hora da percepção, hora do
  conserto e ação tomada.
- Conserto reproduzível, por pipeline ou por script, em vez de comando manual.
- Detecção automática: o sistema percebe a queda e avisa sozinho, com a camada
  identificada no aviso.
- Diferença entre o aviso que sai do sistema e o aviso mandado por uma pessoa.

## 11. Assuntos de estudo individual, sorteados na atividade de aprender com IA

| Assunto | O que o aluno precisa demonstrar |
|---|---|
| Latência e perda controladas com tc/netem | RTT e perda antes e depois, com `ping` ou `iperf3` |
| NAT de saída com nftables | o pacote saindo com o endereço traduzido, no `tcpdump` |
| SNMPv3 com autenticação e privacidade | a mesma consulta falhando sem credencial e respondendo com ela |
| Zona DNS autoritativa própria, com reversa | `dig` direto e reverso respondendo com `AA`, e o SOA da zona |
| Certificado próprio, SNI e validação de cadeia | `openssl s_client` com e sem a CA no truststore |
| Syslog remoto com retenção | mensagem gerada em um container aparecendo no coletor, com data |
| Alerta no Prometheus com `for` | o alerta em `pending` e depois em `firing`, com horário |
| Filtro BPF que isola um sintoma | contagem de pacotes com e sem o filtro, no mesmo pcap |
| Túnel WireGuard entre dois containers | ping dentro do túnel e o tráfego cifrado visto fora dele |
| Throughput sob concorrência com iperf3 | banda de um fluxo sozinho e dividida entre dois |

## 12. Estudo guiado por modelo de linguagem

- Roteiro de estudo com objetivo em uma frase, marcos e o comando que prova cada
  marco.
- Script que demonstra a medição antes e depois, com saída guardada.
- Verificação do que o modelo afirma contra o manual, o `--help` e a
  documentação oficial.
- Registro do erro encontrado, com a fonte que o desmente.
- Os assuntos foram escolhidos em pontos onde os modelos respondem com a
  ferramenta antiga ou com um padrão que já mudou.

## 13. Tecnologias usadas na formação

| Tecnologia | Para quê |
|---|---|
| Debian e utilitários de linha de comando | operação do servidor |
| OpenSSH | acesso por chave e endurecimento do serviço |
| ufw | firewall de host |
| fail2ban | bloqueio por tentativa repetida |
| nginx | serviço web e proxy da topologia |
| APT e unattended-upgrades | atualização de pacotes |
| Docker | camadas em containers, rede e volume |
| PostgreSQL | banco da topologia de três camadas |
| GitHub Actions | pipeline de entrega |
| tcpdump e Wireshark | captura e análise de tráfego |
| Net-SNMP | agente, consulta, notificação e métrica própria |
| Prometheus e exportador SNMP | coleta e série temporal |
| Grafana | painel das métricas |
| Uptime Kuma | monitoração de disponibilidade |
| systemd, cron e nohup | processo que sobrevive à sessão |
| tc e netem, nftables, BIND, OpenSSL, syslog, BPF, WireGuard e iperf3 | assuntos de estudo individual |
| Gemini | guia de estudo da conta institucional |
