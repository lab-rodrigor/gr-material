# Atividade 10: missão, o plantão automatizado

- Prazo: sexta-feira, 31/10/2026, às 20h
- Unidade: 2
- Onde: no seu laboratório e no servidor
- Acompanhamento: `/atividades atividade:10` no Discord
- Nota: proporcional aos objetivos cumpridos. 4 objetivos valem 10.

As atividades 7, 8 e 9 trouxeram o passo a passo. Esta traz o objetivo, e o caminho é escolha sua.

## A situação

Você assumiu o plantão de um servidor. Ele tem três camadas rodando em containers, o front publicado numa porta, e nenhuma delas avisa nada quando cai. O seu antecessor olhava a tela de vez em quando.

Entre o dia da entrega desta atividade e o fechamento, em algum minuto que você não sabe, eu vou derrubar uma camada da sua topologia, como fiz na atividade 5. Você precisa que o sistema perceba sozinho, registre o que aconteceu e avise, sem você estar olhando.

## Os 4 objetivos

**1. Histórico de tráfego.** No painel do Grafana, a última hora de tráfego da interface do seu laboratório precisa estar visível a qualquer momento, com dado contínuo.

**2. Aviso de queda.** Quando a topologia cair, o servidor precisa receber uma notificação SNMP vinda do seu laboratório, em até cinco minutos, carregando qual camada caiu.

**3. Registro do incidente.** Depois que você consertar, o arquivo `~/incidente.md` do seu laboratório precisa trazer: a hora em que a falha entrou, a camada afetada, a hora em que o seu sistema percebeu, a hora do conserto e o que foi feito.

**4. Topologia no ar.** Ao fim do prazo, os 8 itens da atividade 2 precisam voltar a passar.

## O que vale e o que não vale

Vale qualquer caminho: métrica própria com `extend`, script que observa e dispara `snmptrap`, Kuma com notificação, coletor próprio em Python. Vale combinar peças das atividades anteriores de um jeito diferente do que elas ensinaram.

Não vale você mesmo mandar o aviso depois de perceber na mão: o objetivo 2 pede que o aviso saia sem a sua intervenção, e a hora dele comparada com a hora da falha mostra isso.

## O que eu verifico

A série do seu laboratório no Prometheus, com pelo menos 50 minutos de dados na hora do fechamento.

A notificação no receptor do servidor, com o horário dela comparado ao horário da falha que eu injetei, e com o nome da camada no conteúdo.

O `~/incidente.md`, conferido contra o que eu registrei quando derrubei a camada. A hora da falha precisa bater com a minha, com dez minutos de folga.

O estado final da topologia.

## O que você já tem

A atividade 7 deu o agente e a leitura de valores. A atividade 8 deu a notificação e a publicação de métrica própria. A atividade 9 deu o coletor, o histórico e o painel. A atividade 5 mostrou o que acontece quando ninguém está olhando.

Não vou dizer qual combinação usar. Se alguma peça parecer faltar, releia o que cada atividade montou.

## Quando travar

Pergunte no canal `#atv-10-missao`. Eu respondo sobre o funcionamento das ferramentas e sobre o que o objetivo quer dizer, e não sobre qual caminho seguir.
