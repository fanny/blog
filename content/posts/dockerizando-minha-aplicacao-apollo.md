---
title: "Dockerizando Minha Aplicacao Apollo"
date: 2019-04-28T22:30:24-03:00
tags: ["Docker", "Apollo", "Apollo-Server", "Development"]
categories: ["Uncategorized"]
draft: true
---
// MUSIC INTRODUCTIONN

Neste  artigo, iremos cobrir os seguintes tópicos:

O que é Docker e por quê usá-lo? Essa é provavelmente uma das partes mais importantes do artigo, por quê o Docker? Por quê não usar outras tecnologias? Tentarei explicar o quê é docker e em que ele consiste.

“Dockerizando” uma aplicação. Vamos dockerizar uma aplicação para mostrar o que entendemos e podemos usar com os principais conceitos do docker.

Melhorando nossa configuração. Devemos garantir que nossa solução não dependa de valores estáticos. Podemos garantir isso, criando e definindo variáveis de ambiente cujo valor pode ser lido dentro da nossa aplicação.

Gerenciando o container. Agora é muito fácil subir um container, mas também precisamos aprender a gerenciá-lo, já que não queremos que ele fique executando pra sempre.

Essa é a primeira parte da série, nos próximos posts pretendemos abordar, volumes, linkamento, microsserviços, e orquestração.

Por quê usar o docker?

O docker te ajuda a criar um ambiente reproduzível. Você consegue especificar um Sistema Operacional(S.O) específico, a versão exata de diferentes bibliotecas, diferentes variáveis de ambiente e seus valores dentre outras coisas. E o que é mais importante, você consegue executar sua aplicação isoladamente dentro desse ambiente.

E por quê você iria querer isso?

Integração do time com novos desenvolvedores.  A maioria do time pode passar pelo processo de integrar um desenvolvedor a um projeto que já existe, e pode ser um processo traumático  para ele ter que instalar SDK’s, ferramentas de desenvolvimento, banco de dados,  adicionar permissões, muitas vezes chegando a levar semanas.
Ambientes com a mesma configuração. É muito comum querer reproduzir o ambiente de produção no de desenvolvimento, muitas vezes, quando a configuração não é a mesma, pode ser difícil mapear a causa dos erros, porque o ambiente pode ter afetado. Usando o Docker, você pode criar um ambiente DEV, STAGING e PRODUÇÃO, todos com a mesma aparência. 
O famoso jargão “Funciona na minha máquina”. O docker criar contêineres isolados, onde você especifica exatamente o que eles devem conter, você pode enviar esses containers aos clientes e eles irão funcionar da mesma forma que na sua máquina de desenvolvimento.


O que é o docker?
Certo, já temos mencionado boas razões para usar o docker, mas precisamos ir um pouco mais a fundo sobre o que o docker é. Nós definimos que ele nos permite especificar um ambiente como um SO, como encontrar e executar apps e as variáveis necessárias, mas o que o docker tem além disso que podemos saber?

Docker por si só, cria pacotes chamados containers, que contém tudo o que você precisa pra rodar sua aplicação. Cada container tem sua própria memória, CPU e rede e não depende de um sistema operacional ou kernel. A primeira coisa que vem em mente quando citamos containers, são as maquinas virtuais, mas o docker difere-se na maneira com a qual ele compartilha os recursos. Docker usa o chamado “sistema de arquivos em camadas” que permite que os containers compartilhem partes comuns e o resultado final é que os containers são muito menos dependendentes de recursos no sistema host do que uma máquina virtual.

Resumindo, os containers do docker, contém tudo que você precisa pra executar uma aplicação, incluindo o código fonte que você escreveu. Containers também são unidades leves e seguras no seu sistema, Isso torna fácil a criação de vários microserviços que são escritos em diferentes linguagens de programação e que estão usando diferentes versões da mesma biblioteca e do mesmo SO.

Se você está curioso sobre como exatamente funciona o Docker, eu recomendo fortemente que você dê uma olhada nos seguintes links da layered file system and the library runc and also this great wikipedia overview of Docker

Criando a aplicação do apollo-server
 



Criando um dockerfile

Hummm, certo, nós já vimos o que é o Docker e seus benefícios. nós também entendemos que o que executa nossa aplicação é chamado de container. Mas como nós fazemos isso? Bem, nós iniciamos com um arquivo de descrição, chamado de DockerFile. Neste DockerFile, nós especificamos tudo que nós precisamos em termos de OS, variáveis de ambiente, e como obter usa aplicação.

Daremos os seguintes passos:
Criação da aplicação, nós iremos criar um api servida pelo apollo
Criação de um dockerfile, um arquivo de texto que descreve como o docker deve construir nossa aplicação
Construir uma imagem, um pré-passo para levantar e executar nossa aplicação é ter uma imagem.
Criação de um container, esse é o passo final onde nós vemos nossa aplicaçaõ levantada e executando, nós criaremos um container, a partir de uma imagem docker.

Agora, esse arquivo atua como um manifest, mas também como um arquivo de instrução de build, que nós podemos levantar nossa aplicação e executar. Ok, então o que nós precisamos para ter isso?

copiar todos os arquivos do nosso app no container do docker
instalar as dependencias
expor uma porta, que pode ser acessada de fora do container
instruir, o container sobre como iniciar nossa aplicação

Em aplicações mais complexas, nós podemos precisar fazer coisas como setar variáveis de ambiente, ou credenciais para um banco de dados, usar `seeds` para popular nosso banco de dados. Pra agora, nós só precisamos das coisas que descremos acima, Então vamos expressar isso no nosso Dockerfile:

// CODE


FROM Este comando seleciona a imagem do sistema operacional, a partir do Docker Hub. Docker Hub é um repositório global que contém imagens que podemos baixar localmente. No nosso caso, podemos escolher uma imagem baseada no Ubuntu que tem o node instalado, e nós podemos especificar que queremos a última versão, usando a tag :latest
WORKDIR Isso simplesmente significa que nós estamos configurando um diretório de trabalho. Isso é uma maneira de definir onde nossos arquivos ficaram no container.
COPY Aqui nós copiamos os arquivos do diretório atual, para o diretório que acabamos de criar com o WORKDIR
RUN Isso executa um comando no terminal, no nosso caso, queremos instalar as bibliotecas necessárias para construir nosso apollo-server
EXPOSE Isso significa que estamos expondo uma porta, e é através dessa porta que vamos nos comunicar com o nosso container.
ENTRYPOINT Isso representa o comando que precisamos executar para inicializar nossa aplicação, os comandos precisam ser especificados como um array, no nosso caso, isso será traduzido como `npm start`


Building an image

Há dois passos que precisamos implementar para levantar e executar nosso container, eles são:
Criação de uma imagem, com a ajuda do Dockerfile e do comando docker build nós criaremos uma imagem
Execução do container, Agora que temos uma imagem,  precisamos criar um container.

Primeiro de tudo, vamos criar nossa imagem com o seguinte comando:

> docker build -t chrisnoring/node:latest .

A instrução acima cria uma imagem. O . no final é importante, como o docker instrui, e serve para especficar aonde o dockerfile está localizado, nesse caso, no diretório que você está. Se você não tiver a imagem colocada no dockerfile(usando o comando FROM), no nosso caso, a do docker, o docker irá primeiro baixá-la a partir do dockerhub, e seu terminal parecerá com isso
// TERMINAL

O que você está vendo, é como o docker está baixando a imagem do node a partir do dockerhub, e então cada comando está sendo executado WORKDIR, RUN, e os outros. No final, nós vemos sucessfuly built que é nossa sugetsão de que tudo foi construído com sucesso. Vamos dar uma olhada na nossa imagem com:

docker images

// IMAGE LIST

Nós temos uma imagem, só sucesso! :)
Creating a container

O próximo passo é pegar nossa imagem e construir um container a partir dela. Um container é essa peça isolada que executa nossa aplicação dentro dele. Nós construimos um container usando o comando docker run. O comando completo se parece com:

docker run chrisnoring/node

Mas isso não é o suficiente, precisamos mapear a nossa porta interna a externa, da nossa máquina host. Lembre-se que esse é um app que queremos interagir atraves do nosso browser. nós podemos mapear usando a flag -p 

-p [external port]:[internal port]

Agora o comando completo, se parece com:
docker run -p 8000:3000 chrisnoring/node

Ok, executando esse comando significa que nós conseguiremos visitar nosso container  indo http://localhost:8000, 8000 é a nossa porta externa lembre-se que isso mapea para a ponta interna 3000. Bora ver, abra o browser:

// image

É isto, temos um container pipous! :D