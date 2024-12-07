Dockerise a NestJS App
======================
![cat on a boat](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*zbsujWzYrvF4cNdetw1JPg.jpeg)

Nowadays docker is the preferred way to containerize your application. This gives us a lot of advantages and of course it facilitates cloud hosting such as AWS or Azure.

> This article assumes you are familiar with basic understanding of what docker is and what it does.

Today we are going to put our [**NestJS**](https://docs.nestjs.com/) App in a docker container and then we will test it out. For the past 2 years I have been extensively using NestJS professionally and what a blast it has been ! Nest is my absolute favorite nodejs framework. It was created by [Kamil Mysliwiec](https://medium.com/u/f7cc8266ff67?source=post_page---user_mention--2b7f42fc333f--------------------------------) and he did an amazing job ! Cheers.

**Project Structure**

```
nest-app
--src
--Dockerfile
```

STEP 1: Create a NestJS App
---------------------------

To create a simple nestjs app you need to install [**Node.js**](https://nodejs.org/) (>= 8.9.0) and then install the nest CLI globally

```
npm i -g @nestjs/cli
```

Then we create a simple nestjs app via new command

```
nest new nest-app
```

> If you type the command “npm run start” and visit [http://localhost:3000/](http://localhost:3000/) on your browser, you should be able to see Hello World!

**STEP 2: Download docker**
---------------------------

Go to [https://www.docker.com/](https://www.docker.com/) , sign up and install Docker Desktop for Windows. Keep in mind docker takes some time before running for the first time.

STEP 2: Add a Dockerfile to the project
---------------------------------------

Add a file and name it Dockerfile (yes with no extensions)

STEP 3: Configure Dockerfile
----------------------------

Now we are going to add commands in this dockerfile to able to build our image.

Open the dockerfile and add the following line

```
FROM node:8-alpine
```

This basically tells docker to download the official nodejs image from alpine branch. It will serve as our base image.

Next, we create the work directory and add the package.json file to it. After that we add the run commands.

One of the commands points npm to the official registry (you can set your private registry here if you have one) so that npm installs packages from there and the other command is simply to install all packages

```
WORKDIR /appADD package.json /app/package.jsonRUN npm config set registry http://registry.npmjs.orgRUN npm install
```

After this we use the add command to copy our newly created app directory

```
ADD . /app
```

Now it’s time we expose our docker container’s port so that we can use it later to bind the host port to it. I have used 3000 since that’s the default port nestjs uses. You can change it in the **_src/main.ts_** file

```
EXPOSE 3000
```

Lastly, we need to start a command prompt inside our container to actually start the nestjs app!

```
CMD ["npm", "run", "start"]
```

STEP 3: Build the Docker Image
------------------------------

Open a command prompt in your nest-app folder and give the build command along with a name

```
docker build -t nest-app .
```

Here we have used the -t or a.k.a tag command to give our image a specific name tag.

> You can confirm your image by running the **docker images** command

STEP 4: Run the Image!

Now we are ready to run the image we created. Run the following command in the terminal

```
docker run -p 3000:3000 nest-app
```

Keep in mind that by default your host machine cannot access ports of your container. You need to explicitly expose and bind your container ports with host.

> If you visit [http://localhost:3000/](http://localhost:3000/) on your browser, you should be able to see Hello World!

![captionless image](https://miro.medium.com/v2/resize:fit:1106/format:webp/1*mQ2tNb3Bg6PR5vPJdmvq1w.png)

Congrats! Now you are running your app inside a docker container!

Github: [https://github.com/SpartanX1/nestjs-docker](https://github.com/SpartanX1/nestjs-docker)
