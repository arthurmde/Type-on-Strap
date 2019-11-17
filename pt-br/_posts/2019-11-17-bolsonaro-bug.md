---
layout: post
title: "O Bug do Bolsonaro - O fim do horário de verão pode afetar seu sistema"
lang-ref: bolsonaro-bug
locale: pt-br
feature-img: "assets/img/others/bolsonaro-face.png"
thumbnail: "assets/img/others/bolsonaro-face.png"
categories:
  - Hacking
  - Design
tags:
  - ruby
  - rails
  - emberjs
  - javascript
  - bug
  - design
  - frontend
  - backend
excerpt_separator: <!--more-->
---

Vários produtos e aplicativos de software tiveram bugs relacionados ao fuso
horário do Brasil recentemente devido ao [decreto arbitrário de Bolsonaro](https://time.is/time_zone_news/no_dst_in_brazil_in_2019)
que acabou com o horário de verão.
Muitas pessoas ainda estão usando navegadores
que estão com relógios desatualizados pois ainda estão considerando o horário
de verão do Brasil. Você pode ter percebido isso caso use
o WhatsApp ou o Telegram em seu navegador.
Na [Peerdustry](http://peerdustry.com), nós também
enfrentamos um bug interessante em nossa plataforma, que chamamos carinhosamente
de **Bug de Bolsonaro**, o qual merece ser discutido em mais detalhes.

<!--more-->

# Contexto

Antes de explicar nosso bug em específico, vou fornecer um modelo simplificado
do nosso sistema. A plataforma Peerdustry é composta por um Backend em
Ruby on Rails com API para um Frontend baseado no EmberJS.

Um dos principais fluxos implementados em nossa plataforma está relacionado
ao processo de cotação no qual um **Cliente** solicita uma cotação para produzir
uma peça mecânica que será avaliada e respondida por vários **Fornecedores**
que compõem nossa rede. Esses fabricantes são escolhidos pelos administradores
do sistema através da criação de uma tarefa para cada um deles responder à
cotação. Essas tarefas devem ser respondidas antes de um determinado **prazo**
estabelecido pelo **Cliente**.

Recentemente, alguns fornecedores reclamaram que suas tarefas estavam
expirando antes do **prazo**. Além disso, nossos administradores também
relataram ter visto datas e horários estranhos no sistema. Imediatamente,
imaginamos que esses erros surgiram devido as alterações no horário de verão.
Estávamos certos! Porém, isso poderia estar afetando diversas partes do sistema.
Trabalhar com data e hora, fusos horários e
formatos diferentes pode ser complicado, dificultando a investigação para
compreender onde estavam os problemas de fato.

# Um Sistema Mais Simples

Para fins de simplificação, vamos abordar o problema considerando um sistema
mais simples, com apenas os componentes mais importantes para sua compreensão:
um sistema Web de faculdade composto por um Backend em Rails e um Frontend do
EmberJS. Nesse sistema, um **Professor** pode gerar tarefas para os **Alunos**
que devem ser realizadas antes de um determinado **prazo**.
Caso contrário, as tarefas expirarão.

O **Professor** informa a **data limite** enquanto cria as tarefas para os
**Alunos** selecionando uma data através do componente
[Pikaday do Javascript](https://github.com/Pikaday/Pikaday).

![Pikaday JS Calendar]({{ "/assets/img/others/pikaday.png" }})

Antes de enviar esses dados para o Backend, o Frontend o formatará como um
*timestamp* (data / hora) definido no final da data escolhida com a função
[MomentJS' endOf function](https://momentjs.com/docs/#/manipulating/end-of/),
que considera o fuso horário do navegador. Por exemplo, se o professor
escolher 15/11/2019 como prazo final, os dado formatado para ser enviado
ao Backend será `15/11/2019 às 23h59m59s`. Vale a pena notar que todos os
atributos do tipo *timestamp* são formatados e armazenados em **ISO-8601 UTC**.
O formato **GMT** é usado apenas para fins de apresentação da interface do
usuário.

Cada aluno receberá uma tarefa que expira no **prazo final** da tarefa,
que ficará indisponível após esta data. Para isso, sempre que uma tarefa for criada
para um aluno, o Backend agendará um *job* assíncrono com o [Sidekiq](https://github.com/mperham/sidekiq)
para ser executado no **prazo final** que marcará a tarefa como expirada caso
ainda não tenha sido concluída.

Os alunos podem gerenciar suas tarefas pendentes por meio de uma página que
apresenta a lista de tarefas e seus respectivos prazos. Nossos prazos são
exibidos para os usuários formatados como datas simples do Brasil
(por exemplo, 24/11/2019), indicando implicitamente que a tarefa está
disponível até o final do dia informado, conforme ilustrado abaixo.

![Tasks List]({{ "/assets/img/others/bolsobug-task-table.png" }})

Também usamos a biblioteca [MomentJS lib](https://momentjs.com/docs/#/displaying/format/)
para exibir essas datas, que também consideram o fuso horário do navegador.

# O Bug

Após o decreto de Bolsonaro, garantimos que nossos servidores não usassem
o horário de verão de maneira errada para que os *jobs* do Backend fossem
executados no tempo adequado. Como o Backend está operando com o fuso horário
correto **(UTC -3)** e o Frontend sempre fornece os **prazos** no formato `UTC`,
o Backend sempre agenda os *jobs* para expirar as tarefas pendentes no
**timestamp recebido pelo Frontend**.


No entanto, o problema surge quando o **Professor** ou o **Aluno** está usando a
plataforma em um navegador desatualizado que ainda funciona considerando o
horário de verão no Brasil. Assim, alguns usuários do sistema podem ter seus
navegadores com `UTC -3` (fuso horário padrão do Brasil) ou `UTC -2`
(antigo horário de verão do Brasil), o que nos levou a algumas situações
muito estranhas.

Vamos imaginar que um Professor precise criar uma tarefa com prazo para
`01/01/2020`. Temos as seguintes situações:

## 1. Quando o navegador do professor está funcionando corretamente com o UTC -3

Nesse cenário, o prazo informado pelo professor está correto, pois não temos
mais horário de verão e o fuso horário brasileiro original é `UTC -3`.

Se a data informada pelo professor for `01/01/2020`, o Frontend enviará
`02 de janeiro de 2020 02:59:59 UTC` para o Backend
(`01/01/2019 23:59:59 UTC -3`). Como o fuso horário do Backend também está
correto, ele agendará os *jobs* para expirarem as tarefas no horário
esperado pelo professor.

#### 1.A. Quando o navegador do aluno está funcionando corretamente com o UTC -3

Nesse caso, o estudante ao ler a mensagem não irá se confundir, pois o
MomentJS está usando o fuso horário apropriado para exibir a data.
Em outras palavras, o aluno verá a data limite `01/01/2019`, que está correta.

#### 1.B. Quando o navegador do aluno está funcionando incorretamente com o UTC -2 (Horário de Verão)

Nesse caso, o MomentJS aplicará o fuso horário `UTC -2` ao prazo final
recebido do Backend no formato `UTC`, obtendo `02 de janeiro de 2020 00:59:59 UTC -2`.
Como exibimos apenas a data e ocultamos a hora, o usuário verá `02/01/2020`
em vez de `01/01/2020` como o prazo final para sua tarefa, levando-o a
um mal-entendido da data correta. Embora o aluno pense que poderá concluir
sua tarefa até `01/02/2020`, nessa data a tarefa não estará mais disponível.

## 2. Quando o navegador do professor está funcionando incorretamente com o UTC -2 (Horário de Verão)

Nesse cenário, temos um problema grave que independe dos navegadores do aluno,
pois o prazo fornecido ao Backend está incorreto.

Se a entrada do professor for `01/01/2020`, o Frontend enviará
`02 de janeiro de 2020 01:59:59 UTC` para o Backend (`01/01/2019 22:59:59 UTC -3`).
Isso significa que o prazo expirará **1 hora antes do esperado**.

#### 2.A. Quando o navegador do aluno está funcionando corretamente com o UTC -3

Nesse caso, o estudante não se confunde ao ler a data, mesmo que o MomentJS
esteja usando um fuso horário diferente do original para exibir a data.
A aplicação do `UTC -3` ao prazo original produzirá `01 de janeiro de 2020 22:59:59 UTC -3`.

Assim, o aluno veria `01/01/2020` como a data limite, que está correto.
No entanto, ele espera ter o prazo disponível até `23:59:59` horas, o que
não ocorrerá.

Você pode argumentar que exibir a hora e a data do aluno no sistema minimizaria
o problema: `01/01/2020 às 22:59`. Mas o horário provavelmente será ignorado
pelo aluno, pois ele está acostumado a ter tarefas disponíveis até às `23:59`.

#### 2.B. Quando o navegador do aluno está funcionando incorretamente com o UTC -2 (Horário de Verão)

Embora o MomentJS use o mesmo fuso horário do prazo original para exibir
a data, ainda temos alguns problemas.

A aplicação do `UTC -2` ao prazo original produzirá `01 de janeiro de 2020 23:59:59 UTC -2`.
Nesse caso, o aluno verá `01/01/2020` como a data limite, que está correta.
No entanto, ele enfrentará o mesmo problema dos usuários do `UTC -3`,
pois espera ter o prazo disponível até às `23:59h`, o que não ocorrerá.
Pior ainda, não podemos exibir a hora para ele como fizemos no último exemplo,
pois a hora exibida seria incorreta (exibindo `23:59h`, mesmo que expire às `22:59h`).

# Como consertar?

Existem algumas abordagens para minimizar o impacto do **Bug do Bolsonaro**.
A maioria deles é bastante *hacky* na minha opinião.

Em geral, se você garantir que seu sistema armazena e processa dados de data/hora
no formato `UTC`, sua preocupação estará principalmente na sua camada de
apresentação.

No contexto específico da plataforma da **Peerdustry**, os dois papéis,
Fornecedores e Clientes, quase nunca usam a plataforma após as 19h
(final do horário comercial de suas empresas), o que significa que o principal
problema é exibir a data-limite incorreta para os **Fornecedores**
(Cenário 1.B). Nesse sentido, se alterarmos o Frontend para sempre definir o
**prazo** para `22:59:59 UTC-3` antes de enviá-lo ao Backend, os Fornecedores
sempre verão a **data** correta. Mesmo que as tarefas expirem uma hora antes
do esperado, quase ninguém seria impactado por isso.

**Essa abordagem nunca poderia ser aplicada a um sistema de faculdade =D**

Também é possível alterar o fuso horário usado pelo [MomentJS](https://momentjs.com/time zone/docs/)
para imitar as novas regras de Horário de Verão do Brasil. No entanto, esse
é o tipo de abordagem que causará dores de cabeça quando você tiver usuários
em mais de um fuso horário, além de comprometer a internacionalização do seu
sistema.

Na minha opinião, a forma mais apropriada que cobre a maior parte dos casos em
situações similares às causadas pelo Bug do Bolsonaro seria:
* Apresentar a hora junto com a data do prazo final ou até mesmo uma contagem
regressiva de quanto tempo falta para a tarefa expirar.
* Identificar e informar aos usuários quando os seus navegadores estiverem
usando informações desatualizadas sobre o fuso horário do Brasil, alertando
sobre possíveis bugs and solicitando que eles atualizem seus navegadores.


**E você já enfrentou algum bug estranho após o decreto de Bolsonaro?**

<hr>

<small>A imagem de capa é do Fábio Rodrigues Pozzebom/Agência Brasil [CC BY 2.0](https://creativecommons.org/licenses/by/2.0) via Wikimedia Commons</small>

<span>**#ELENAO - Let's get moving on! ;)**</span>


