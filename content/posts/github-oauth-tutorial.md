---
title: "Github OAuth Tutorial"
date: 2019-05-15T22:14:07-03:00
tags: ["Github Oauth", "Nodejs", "Express", "Development"]
author: "Fanny Vieira"
categories: ["Uncategorized"]
---

## *Fluxo* de autenticação no github
    
Quando um usuário deseja se logar na nossa aplicação, ele precisa clicar no botão de Login, ao clicar nele,
o mesmo será direcionado a um form provido pelo github, em que é necessário colocar seu nome de
usuário e senha (caso o usuário não esteja logado na sua conta do github) como mostra a imagem abaixo:  
`// imagem`  
Se esse é o primeiro acesso, será feito um pedido de autorização para o acesso de algumas informações:  
`//imagem`  
Uma vez que o usuário aceitou as permissões, o back conseguirá gerar o token, e passá-lo para o front através de uma url de redirecionamento definida no arquivo `.env`, caso as permissões não sejam aceitas, ele será redirecionado para url de erro, também configurada no `.env`, como veremos a seguir.

Certo, tudo muito lindo, mas como fazemos isso funcionar por debaixo dos panos?

Nós, aqui do panelinha de es, somos devs que gostam de abstrair ideias a partir de figuras, sendo assim, para que você entenda como se dá a comunicação entre o cliente o servidor, observe a figura abaixo:


![](https://i.imgur.com/Q0uQ9K2.png)

A partir da figura acima, vemos que o cliente, apenas requisita o login e o back abstrai toda a lógica de negócio da autenticação, e ao final de todos os processos, redirecionará, para a url de sucesso ou erro configurada.

Okay, agora que definimos isso, vamos pular para o código.


## Criando uma aplicação Github Oauth
Para criar a aplicação do Github OAuth, acesse o github, e siga o seguintes passos:

1. Navegue até a parte de settings.
2. Clique na seção de Developer Settings.
3. Clique no botão para criar uma nova aplicação OAuth.
4. Preencha corretamente as informações necessárias.

## Configurando o servidor
Após termos criado o github oauth app, precisamos configurar nosso servidor, para testar o que definimos anteriormente.

Para isso, baixamos [o projeto do servidor](https://github.com/panelinhadees/server.git), e para que ele funcione corretamente, é necessário criar um  arquivo chamado `.env`, que nos ajuda a definir as configurações necessárias para execução da aplicação, veja o exemplo abaixo e crie o arquivo seguindo esse mesmo template.

Podemos dividir as configurações em quatro tipos:
- **Github OAuth**: Configurações relacionadas a configuração da app auth, que criamos no passo anterior.
- **Server**: Configuração da porta, e da url de acesso ao servidor.
- **Client**: Configuração da porta, e da url de acesso ao cliente(você entenderá a seguir porque isso é necessário).
- **Rotas**: As rotas de redirecionamento para o cliente, em caso de erro ou sucesso.

*.env*
```.env
# Required
GITHUB_OAUTH_CLIENT_ID=your-oauth-app-client-id
GITHUB_OAUTH_CLIENT_SECRET=your-oauth-app-client-secret
GITHUB_OAUTH_CALLBACK_URL=your-oauth-app-callback-url

# Optional - Uncomment if you want to change these values
# SERVER_BASE_URL='http://localhost'
# SERVER_PORT=5000

# CLIENT_BASE_URL='http://localhost'
# CLIENT_PORT=8080

# The auth token will be sent to $CLIENT_BASE_URL:$CLIENT_PORT$CLIENT_AUTH_SUCCESS_PATH/:token
# In case of error a message will be sent to $CLIENT_BASE_URL:$CLIENT_PORT$CLIENT_AUTH_ERROR_PATH

# CLIENT_AUTH_SUCCESS_PATH='/auth'
# CLIENT_AUTH_FAIL_PATH='/error'
```

Agora, podemos instalar as dependências e levantar o nosso servidor.

## Instalando as dependências
Para instalar as dependências, usamos o `npm`, e rodamos o seguinte comando:

`npm install`

E para subir o servidor, executamos esse:

`npm start`

*TODO: docker config*

## Linkando o server ao front

Crie um componente, que contenha um link como o abaixo:

```html
<a id="login-button" href="SERVER_BASE_URL:SERVER_PORT/login">Log In With GitHub</a>
```

## Gerenciando o login
Quando o cliente clica no link de login, é a rota `/login` do servidor que é invocada.

Definida pelo seguinte código:

```js
router.get('/login', (req, res) => {
  const state = generateRandomState(16);
  res.cookie(stateKey, state);

  const queryString = querystring.stringify({
    client_id: config.github.clientId,
    redirect_uri: config.github.redirectUrl,
    state: state,
    scope: config.github.scope
  });

  const url = getAuthBaseURL(`/authorize?${queryString}`);
  res.redirect(url);
});
```
Existe uma série de passos a ser feita para se logar no github, a primeira delas é [requisitar a identidade do usuário](https://github.com/login/oauth/authorize), para tal, requisitamos um endpoint da api do github passando alguns parâmetros, como mostra a especificação a seguir:

`GET https://github.com/login/oauth/authorize`

Os parâmetros passados são:
- o `client_id` da nossa github oauth app, que setamos no arquivo `.env`
- a `redirect_uri`, a url na qual os usuários serão redirecionados, ao final da nossa autenticação, quando todos os passos forem concluidos, no nosso caso, a url de sucesso do cliente que setamos no `.env`
- o `state`, que é só uma string randômica, usada para prevenção de alguns ataques, no próximo passo isso fará mais sentido, temos uma lógica própria pra gerar essa string randômica, verifique [esse arquivo](https://github.com/panelinhadees/server/blob/master/src/auth/util.js)
- o `scope`, que define os tipos de acesso que a nossa aplicação precisa, atualmente apenas os que estão definidas [aqui](https://github.com/panelinhadees/server/blob/33ea7a5f3d11bb9880297efbb15719753a0c9e9f/src/config.js#L7)

*Obs: O state é salvo nos cookies do browser, para serem usados no passo a seguir*

Uma vez que definimos esses parâmetros, requisitamos a url do github, se o usuário aceitar as permissões, o próprio github irá redirecionar pra o link de `callback` definido na criação do github oauth app, no nosso caso, foi `/callback` mesmo.

A rota `/callback` possui o seguinte código:

```js
router.get('/callback', (req, res) => {
  const storedState = req.cookies ? req.cookies[stateKey] : null;
  const { state } = req.query;

  if (state && storedState === state) {
    requestAccessToken(req, res);
  } else {
    const queryString = querystring.stringify({ error: 'state_mismatch' });
    const url = getClientURL(`${config.client.errorPath}?${queryString}`);
    res.redirect(url);
  }
});

const requestAccessToken = (req, res) => {
  res.clearCookie(stateKey);
  const { code } = req.query;

  const requestOptions = {
    url: getAuthBaseURL('/access_token'),
    json: true,
    form: {
      code,
      redirect_uri: config.github.redirectUri,
      client_id: config.github.clientId,
      client_secret: config.github.clientSecret,
    },
  };

  request.post(requestOptions, (error, response, body) => {
    let url;
    if (!error && response.statusCode === 200) {
      const { access_token } = body;
      url = getClientURL(`${config.client.successPath}/${access_token}`);
    } else {
      const queryString = querystring.stringify({ error: 'invalid_token' });
      url = getClientURL(`${config.client.errorPath}?${queryString}`);
    }
    res.redirect(url);
  });
};
```

É a partir dela, que [requisitamos o nosso token de acesso](https://developer.github.com/apps/building-oauth-apps/authorizing-oauth-apps/#2-users-are-redirected-back-to-your-site-by-github) e redirecionamos para url do nosso cliente.

Agora você entenderá porque o `state` é importante, a fim de garantir que o usuário que está requisitando esse token foi o mesmo que fez o passo anterior, logo, não foi alvo de um ataque, o github passa na sua url o state que ele recebeu no primeiro passo, daí se os states forem iguais(por isso fazemos a comparação), prosseguimos com a requisição do token, caso contrário, redirecionamos pra url de erro do cliente.

Dado que os states foram iguais, fazemos uma requisição `POST` pra essa url do github:

`POST https://github.com/login/oauth/access_token`

passando os seguintes itens no header:
- o `code` recebido na nossa url de callback como parâmetro, quando o github redireciona, ela só tem 10 minutos de duração, então se passar disso, você precisa fazer o passo a passo do ínicio.
- a `redirect_uri` url na qual os usuários serão redirecionados, ao final da nossa autenticação, quando todos os passos forem concluidos, no nosso caso, a url de sucesso do cliente que setamos no `.env`.
- o `client_id` da nossa github oauth app, que setamos no arquivo `.env`
- o `client_secret` da nossa github oauth app, que setamos no arquivo `.env`

Uma vez que esses campos foram definidos podemos fazer a requisição, se tudo der certo, o token será retornado e redirecionaremos pra url de sucesso do cliente, caso contrário, redirecionamos pra de erro.  
