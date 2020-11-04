# Tarefas em Background utilizando Node.js e Redis

###### Projeto faz parte do Bootcamp Node.js Web Developer do site Digital Innovation One

Projeto original disponível em: https://github.com/robertosousa1/background-jobs-class-by-dio

##### Objetivo do projeto é desenvolver um cadastro de usuário e envio de e-mail de confirmação do cadastro como tarefa em background utilizando Node.js e Redis.



Tecnologias utilizadas:

1. Docker

2. Redis via Docker

3. Visual Code com extensão .env

4. Node.js

5. Framework Express

6. Dependências: nodemailer, dotenv, nodemon, sucrase

7. Bibliotecas: Bull , Bull-Board, Password-generator

8. Site mailtrap.io

9. Insomnia

   

Instalar o Docker for Windows

Verificar se o Docker foi instalado: digite no terminal o comando: **docker -v**

Baixar no Docker a imagem do Redis: 

Ir até Docker Hub e selecionar a imagem do Redis  

[Docker Hub]: https://hub.docker.com/_/redis



Inserir no Terminal o comando: docker pull redis

Verificar as imagens instaladas no Docker: digite no terminal o comando: **docker images**

 

Para iniciar a imagem do Redis no Docker digite o seguinte comando no terminal: **docker run -d -p 6379:6379 -i -t redis: (digite a versão da imagem que baixou).**

Exemplo: docker run --name redis -p 6379:6379 -d -t redis:alpine**

Para verificar os containers em execução digite o comando: **docker ps**

Para encerrar a execução do container digite o comando: **docker stop [container Id],** 

o id do Container pode ser visto com o comando docker ps

 

**Para inicializar um projeto:** utilize os comandos yarn ou npm

yarn init -y (linux)

npm init -y (windows)



Será criado o arquivo package.json, abra projeto no VS Code.

 

Com o projeto aberto instale as dependências via terminal do VS Code: **C:\Projetos\Redis>npm add express nodemailer dotenv (com yarn)**

**C:\Projetos\Redis>npm install express nodemailer dotenv (com npm)**

 

Criar o arquivo**.gitignore** e em seguida no terminal do VS Code digite o comando**: git init**

 

Instale as dependências: **npm install nodemon sucrase -D**



Crie na raiz do projeto o arquivo **nodemon.json**

```
{
    "execMap":{
        "js": "sucrase-node"
    }
}
```

Crie na raiz do projeto a pasta **src** e dentro dela o arquivo **server.js**

 

**Crie um script no arquivo package.json**

```
"scripts": {
    "start": "nodemon src/server.js"
  },
```

Crie na raiz do projeto o arquivo **.env** (é necessário ter a extensão DotEnv instalada no VS Code), configure a porta de acesso dentro do arquivo **.env.**

```
PORT=8080
```

Configure o arquivo **server.js**

```
import 'dotenv/config';
import express from 'express';

const app = express();

app.use(express.json());

app.listen(process.env.PORT, () => {
    console.log(`Server running on the ${process.env.PORT}`)
});
```

Digite no terminal do VS Code o comando: **npm start**

**Deverá obter como resposta:**

**[nodemon] starting `sucrase-node src/server.js`**

**Server running on the 8080**

 

Criar na pasta **src** a pasta **app**

 

Instalar a dependência via terminal do VSCode: **npm install password-generator  (Windows)** ou yarn add password-generator (Linux)

 

Ir ao site [**https://mailtrap.io/**](https://mailtrap.io/) criar uma conta gratuita ou acessar caso já tenha.

 

Em inboxes na aba SMTP/POP3 em integrations selecionar Nodemailer e copiar o código.

 

Configure o arquivo **.env** com os dados copiados no site do mailtrap.io

```
PORT=8080

MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USER="digite/ou cole aqui o seu código de user copiado do mailtrap "
MAIL_PASS="digite/ou cole aqui o seu password copiado do mailtrap "
```

Criar na pasta **src/app** a pasta **lib** e dentro dessa pasta o arquivo **Mail.js**

```
import nodemailer from 'nodemailer';
import mailConfig from '../../config/mail';

export default nodemailer.createTransport(mailConfig);
```

Na pasta **src/app** crie uma nova pasta com o nome **jobs** e nessa pasta crie o arquivo **RegistrationMail.js**

```
import Mail from '../lib/Mail';

export default {
    key: 'RegistrationMail',
    options: {
        delay: 5000,
        priority: 3
    },
    async handle({ data }) {
        const { user } = data;

        await Mail.sendMail({
            from: 'Sandro <sandro@contato.com.br>',
            to: `${user.name} <${user.email}>`,
            subject: 'Cadastro de usuário',
            html: `Olá, ${user.name}, aprendendo REDIS.`
        });
    }
}
```

Na pasta **src/app/jobs** crie o arquivo **index.js**

```
import RegistrationMail from "./RegistrationMail"

export { default as RegistrationMail } from './RegistrationMail';
```

Instalar a dependência bull através do terminal do VS Code digite: **npm install bull --save**

Instalar a dependência bull-board através do terminal do VS Code digite: **npm install bull-board**

 

Criar dentro pasta **src/config** o arquivo **redis.js**

```
export default {
    host: process.env.REDIS_HOST,
    port: process.env.REDIS_PORT
}
```

Acrescente as informações sobre o Redis no arquivo **.env**

```
PORT=8080

MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USER="digite/ou cole aqui o seu código de user copiado do mailtrap "
MAIL_PASS="digite/ou cole aqui o seu password copiado do mailtrap "

REDIS_HOST=127.0.0.1
REDIS_PORT=6379
```

Criar dentro pasta **src/app**/**lib** o arquivo **Queue.js**

```
import Queue from 'bull';
import redisConfig from '../../config/redis';

import * as jobs from '../jobs';

const queues = Object.values(jobs).map(job => ({
    bull: new Queue(job.key, redisConfig),
    name: job.key,
    handle: job.handle,
    options: job.options,
}));

export default {
    queues,
    add(name, data) {
        const queue = this.queues.find(queue => queue.name === name);

        return queue.bull.add(data, queue.options);
    },
    process(){
        return this.queues.forEach(queue => {
            queue.bull.process(queue.handle);

            queue.bull.on('failed', (job, err) => {
                console.log('Job failed', queue.key, job.data);
                console.log(err);
            })
        })
    }
```

Criar na pasta **src/app** a pasta **controllers** e dentro dessa pasta criar o arquivo **UserController.js**

```
import passwordGenerator from 'password-generator';

import Queue from '../lib/Queue';

export default {
    async store(req, res) {
        const { name, email } = req.body;

        const user = {
            name,
            email,
            password: passwordGenerator(15, false)
        };

        await Queue.add('RegistrationMail', { user });

        return res.json(user);
    }
}
```

**Criar na pasta src** o arquivo **queue.js**

```
import 'dotenv/config';

import Queue from './app/lib/Queue';

Queue.process()
```

Acrescente no script do arquivo **package.json** as informações sobre o **queue.js**

```
"scripts": {
    "start": "nodemon src/server.js",
    "queue": "nodemon src/queue.js"
  },
```

Faça as importações e configurações no arquivo **server.js**

```
import 'dotenv/config';
import express from 'express';
import BullBoard from 'bull-board';

import UserController from './app/controllers/UserController';
import Queue from './app/lib/Queue';

const app = express();
BullBoard.setQueues(Queue.queues.map(queue => queue.bull));

app.use(express.json());

app.post('/users', UserController.store);

app.use('/admin/queues', BullBoard.UI);

app.listen(process.env.PORT, () => {
    console.log(`Server running on the ${process.env.PORT}`)
});
```

Abra um novo terminal no VS Code, a tela do terminal será dividida em duas.

No primeiro terminal caso ainda esteja rodando a aplicação pressione as teclas CTRL + C e digite S para parar a aplicação.

No primeiro terminal digite**: npm start**

No segundo terminal digite: **npm run queue**

Abra um navegador de internet e digite o endereço: http://localhost:8080/admin/queues 

Abra o Postman ou Insomnia e execute um comando Post com o seguinte JSON: 

```
{
	"name": "Joao da Silva",
	"email": "joaodasilva@email.com"
}
```

Assim que o comando POST é executado é possível visualizar no site http://localhost:8080/admin/queues a execução no dashboard RegistrationMail do arquivo enviado.

Ele passa pelas etapas de delayed, active e quando aparecer em completed abra o site [**https://mailtrap.io/**](https://mailtrap.io/) e vá a caixa inboxes e você terá recebido um novo e-mail .


