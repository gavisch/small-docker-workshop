# Getting Started

Quick workshop on using docker for your hacks so you don't have to keep saying "It works on my machine" to your team members :thumbsup:

## Pre-requisites

- A terminal (depends on your OS!)
- Working docker installation (See below)
- npm and nodejs are installed [(Click here for instructions)](https://www.npmjs.com/get-npm)
- A docker hub account [(Click here for a quickstart)](https://docs.docker.com/docker-hub/)

## Install docker CE

Go over to [https://docs.docker.com/install/](https://docs.docker.com/install/) and install the version of docker you need for your operating system. Also make sure to [install docker-compose](https://docs.docker.com/compose/install/)!

### Execute docker without Sudo

Depending on your operating system, you might want to make sure you can run docker without having to deal with annoying sudo. These are the commands for Ubuntu.

Add your username to the docker group:

``` bash
sudo usermod -aG ocker ${USER}
```

Apply the new group membership by typing the following and entering your sudo password:

``` bash
su - ${USER}
```

And confirm your user is now part of the **docker** group with:

``` bash
id -nG
```

### Can't get docker running locally?

Why not try something like Digital Ocean? It's a pretty simple cloud provider with tons of images you can quickly deploy. It does require a credit card to get authorized to use it but at $5 a month is nothing since you'll probably just be using it for a hackaton (it'll be way less since you only pay for what you keep alive). It's also web deployed so linking your domain names to it and using it is easy. To get setup with a docker server, do the following:

- Go to [https://cloud.digital.ocean](https://cloud.digital.ocean) and sign up (credit card required)

- Click on *+ New Project* on the left pane

- Enter your new projects info and then press *Create Project*

- Skip the move resources section

- Click the *Get Started with a Droplet* button

- Click on *Marketplace* under *Choose an image* and select *Docker 18.09.2~3 on 18.04* (or whatever is the newest version)

- Scroll down to *Choose a plan*, leave it as Standard and scroll all the way to the left until you see $5/mo and select that

  - You probably won't need anything beefy for your app, but if you do, select a bigger plan and tear it down as soon as you're done. We will be creating Dockerfiles and adding images to registry so we can run them anywhere we want later!

- Skip over everything until you get to *Add your SSH keys*

- Click on *New SSH Key* and paste your public key, name it and then click *Add SSH Key*

  - You will need this in order to be able to ssh into the server and run bash commands. The alternative of using the web terminal sucks.
  - If you don't know how to make an SSH key, checkout this info on [https://www.ssh.com/ssh/keygen/](https://www.ssh.com/ssh/keygen/)

- Finally click on *Create*! Good job, you got a docker server now!

Now we need to get you logged in to your new server so you can do your thing!

- Go to your newly created droplet and copy the ipv4 address

- In your terminal, type `ssh root@<youripv4addresshere>`

- Type yes and hit enter when it asks `Are you sure you want to continue connecting (yes/no)?`

- You should see a bunch of text on ubuntu and docker come up followed by `root@docker-s-1vcpu-1gb-nyc1-01:~# ` or something along those lines. This means that you succesfully logged in! Congrats!

**Note** Using this method is a good alternative. Just know that you might have to do a little more to open up your ports properly. It's not difficult but know that it is something you'll have to contend with. [This is a good resource on working with ports inside your new server](https://bobcares.com/blog/digitalocean-open-port-8080/)


### Testing your docker installation

Go to your terminal and enter: `docker run hello-world`

Docker should have gone to the docker hub, pulled the hello-world image and ran it. Read the text it prints out and try the little command it tells you to :D

With this, you should be good to continue.

## Building your first dockerized app!
### Setting up your app
First start by making a little folder called `tinyapp`, navigate into it, and initiate npm:

```bash
$ mkdir tinyapp
$ cd tinyapp
$ npm init -y
```

Now we need to install Express, a minimalistic web framework for node [(click here info)](https://expressjs.com)

```bash
$ npm install express
```

Next we need to modify our package.json to add easy start script. Open up package.json with your favorite text editor and add the line `"start": "node app.js",` under `"scripts: {"`. Your package.json should look like this now:

```json
{
  "name": "babyapp",
  "version": "1.0.0",
  "description": "",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "express": "^4.16.4"
  }
}
```

We're just missing our app file. Lets add it!

```bash
$ touch app.js
```

Open app.js with your favorite text editor (vim is #1) and add this to it:

```nodejs
const express = require('express')
const app = express()
const port = 3000

app.get('/', (req, res) => res.send('Hello World!'))

app.listen(port, () => console.log(`Tiny app listening on port ${port}!`))
```

If you're not sure what you're looking at above. I recommend reading [this](https://expressjs.com/en/starter/hello-world.html)

All we have to do now is run our little app through that nice script we added in our package.json

```bash
$ npm start
```

A little message should pop up with `Tiny app listening on port 3000!`. Open up your browser and go to `http://localhost:3000` and you'll see the words "Hello World!"

Your project folder should look like this:

```text
tinyapp/
├── app.js
├── node_modules
├── package.json
└── package-lock.json
```

Great! We got a little app going, so now lets dockarize it!

### Dockarizing your tiny app

Your docker should be good to go for this part. We're going to be making what is called a Dockerfile. A Dockerfile is basically the recipe of what will be included in your application container when it's ran. Using this file lets you define your container environment to avoid problems with dependencies or runtime versions. It lets you avoid the whole "but it works on my machine" problem!

First, lets make our Dockerfile

```bash
$ mkdir Dockerfile
```

Docker images are made with other docker images that end up building on one another. In this case, we're going to add a base image called node:latest. This base image will come with the essentials needed to start a nodejs app. Add the FROM instruction to your Dockerfile:

```Dockerfile
FROM node:latest
```

Lets set what our work directory will be inside our container with the WORKDIR instruction:

```Dockerfile
WORKDIR app/
```

Remember to set a WORKDIR. Docker will just make one for you if you don't. That can be problematic later on!

Now we have to install our app depenencies onto the container. Lets get our package.json and package-lock.json files into the container by using the COPY instructuion to copy them:

```Dockerfile
COPY package*.json ./
```

The `*` is a wildcard that will look ahead past package towards .json. This will grab anything that starts with package and ends in .json which includes package-lock.json!

The `./` at the end is actually the WORKDIR we set earlier `app/`. Docker knows that every command after setting the WORKDIR will be inside that specified folders context. We could use just `.` but we need `./` since we are copying more than one file to our work directory.

We then install our dependencies with the RUN instruction:

```Dockerfile
RUN npm install
```

Next, lets copy our app into the WORKDIR we set:

```Dockerfile
COPY . .
```

Those two dots are weird, right? The `.` means current directory. Remember we put the Dockerfile in the same folder alongside the rest of your application. That `.` will grab everything inside the directory the Dockerfile sits in and puts it inside our WORKDIR `app/` inside the container.

Finally, we can expose port 3000 (the port we set in our app.js) and start the app:

```Dockerfile
EXPOSE 3000
CMD ["npm", "start"]
```

Oddly enough, EXPOSE doesn't actually expose any ports. It's more for documenting what port the container will publish at runtime. CMD runs the "start" npm script we made earlier in our package.json. Do note that there should only be one CMD instruction in a Dockerfile. If you have multiple, only the last one will actually run.

Awesome! We have a Dockerfile! There's a lot more instructions you can use [(click here)](https://docs.docker.com/engine/reference/builder/) but for now our Dockefile should look like this:

```Dockerfile
FROM node:latest

WORKDIR /app

COPY package*.json .

RUN npm install

COPY . .

EXPOSE 3000
CMD [ "npm", "start" ]
```

If we were to build our container now, we'd take everything inside our tinyapp directory. Lets build a .dockerignore to ignore some of the files and directories we don't want copied into our container.

Create a .dockerignore file:

```bash
$ touch .dockerignore
```

Inside the file, add the following:

```text
node_modules
npm-debug.log
Dockerfile
.dockerignore
.git
.gitignore
```

`.git` and `.gitignore` are in there just in case you're working with git.

At this point, your tinyapp directory should look like this:

```test
tinyapp/
├── app.js
├── Dockerfile
├── node_modules
├── package.json
└── package-lock.json
```

Now we're ready to build the image! To do this, we will use the __docker build__ command with the -t flag. The -t flag lets you tag an image with a name of your choosing. Without doing this, your images will end up being a weird default name. We're going to tag this as `tinyapp-demo` but you can change it to whatever you want. And make sure to subtitute `your_dockerhub_username` for your actual docker hub username. We will be publishing this tinyapp image to Docker Hub later! In your terminal, run the following command:

```bash
$ docker build -t your_dockerhub_username/tinyapp-demo .
```

It'll take a few minutes, but once that's over we can look at our image:

```bash
$ docker images
```

You should see:

```bash
REPOSITORY             TAG                 IMAGE ID            CREATED             SIZE
gavisch/tinyapp-demo   latest              00d9d4aac61f        4 seconds ago       906MB
node                   latest              9ff38e3a6d9d        23 hours ago        904MB
```

Now we will use the __docker run__ command with three flags:

- __-p__ This flag publishes the port on the container and maps it to our host. We will be using port 80 on our host but you can change it to whatever you want if necessary.
- __-d__ This runs our container in detached mode. That's a fancy phrase for in the background
- __--name__ This lets us name our container whatever memorable name we want

Run this command to build our container:

```bash
$ docker run --name tinyapp-demo -p 80:3000 -d your_dockerhub_username/tinyapp-demo
```

Once the container is running, you can check it out and other containers that are running with __docker ps__:

```bash
$ docker ps
```

The output looks like this:

```bash
CONTAINER ID        IMAGE                  COMMAND             CREATED             STATUS              PORTS                  NAMES
c479e95cfe56        gavisch/tinyapp-demo   "npm start"         About an hour ago   Up About an hour    0.0.0.0:80->3000/tcp   tinyapp-demo
```

Note the PORTS section. We should be able to hit our app `0.0.0.0:80` through localhost. Let's try it! You can either open up a browser and enter http://localhost as the url, or `curl localhost`, or you can use postman to do a get on http://localhost. Let's do the curl command:

```bash
$ curl localhost
```

Our output is:

```bash
Hello World!
```

### Using a repo with your images

Lets make our image available to our friends by making use of that Docker Hub account we made. The first step is to login to the Hub:

```bash
docker login -u your_dockerhub_username
```

Enter your password when prompted and you should see `Login Succeeded`

Now we push our image to our repo for the rest of our team members to use:

```bash
$ docker push your_dockerhub_username/tinyapp-demo
```

Lets get rid of our local image and pull the tinyapp-demo image that we just pushed. So lets begin with some cleaning! First lets see and stop our container. We're going to do that by running __docker ps__ to see what containers are running then use __docker stop containerID_or_name__:

```bash
$ docker ps
CONTAINER ID        IMAGE                  COMMAND             CREATED             STATUS              PORTS                  NAMES
c479e95cfe56        gavisch/tinyapp-demo   "npm start"         15 hours ago        Up 15 hours         0.0.0.0:80->3000/tcp   tinyapp-demo

$ docker stop tinyapp-demo
tinyapp-demo
```

Now, we're going to look at what images we got saved locally and then delete them all:

```bash
$ docker images -a
REPOSITORY             TAG                 IMAGE ID            CREATED             SIZE
gavisch/tinyapp-demo   latest              dbf67dfc122e        15 hours ago        906MB
<none>                 <none>              2505fd0f466d        15 hours ago        906MB
<none>                 <none>              38048835c2a9        15 hours ago        906MB
<none>                 <none>              51ee47d4031e        15 hours ago        906MB
<none>                 <none>              910e094a4046        15 hours ago        904MB
<none>                 <none>              c515ca355a78        15 hours ago        904MB
node                   latest              9ff38e3a6d9d        38 hours ago        904MB

$ docker system prune -a
```

`docker system prune -a` removes all of the following:

- all stopped containers
- all networks not used by at least one container
- all images without at least one container associated to them
- all build cache

Now we can pull our image and see that it's available locally:

```bash
$ docker pull your_dockerhub_username/tinyapp-demo

$ docker images
REPOSITORY             TAG                 IMAGE ID            CREATED             SIZE
gavisch/tinyapp-demo   latest              dbf67dfc122e        15 hours ago        906MB
```

Then we can run it again, see it running and test it:

```bash
$ docker run --name tinyapp-demo -p 80:3000 -d your_dockerhub_username/tinyapp-demo

$ docker ps
CONTAINER ID        IMAGE                  COMMAND             CREATED             STATUS              PORTS                            NAMES
2456142b1176        gavisch/tinyapp-demo   "npm start"         6 seconds ago       Up 4 seconds        3000/tcp, 0.0.0.0:80->3000/tcp   tinyapp-demo

$ curl localhost
Hello World!
```

Congratulations! You just learned a ton of stuff on docker. Now you're ready for using docker with docker compose :thumbsup:!

### A few more useful commands
If you are ever in need, you can always run `docker --help` to find a list of commands you can use. You can then do docker `<command> --help` to get more info on what options are available for any command.

If you ever run into the problem where you have a container named "blah" and are trying to make another named "blah", you can remove the old container with `docker rm container_name`. Now you should be able to create your new container.

If `docker system prune` is too scary or you just want to selectively remove images then use `docker rmi image_name`.

## Docker compose!

Now that you know how to stand up a little dockerized app we will move onto creating a RESTful API with NodeJS, Express and Mongodb in docker! The good news is that you don't actually need to install any of these. We're going to let our image do that for us.

### Prerequisites

- Install docker-compose
- Clone this repo down

### Composing your keyboard tracker API

For this part, we will be using a docker tool called __compose__ that allows us to build multi-container applications. Trying to connect multiple containers together is a pain so __compose__ allows us to create a __docker-compose.yml__ file that will hold all the instructions for our app to build and run with just a simple __docker-compose up__ command.

Make sure to do some light reading on the [docker compose documentation](https://docs.docker.com/compose/)]

Since this workshop is about docker, we're not going to actually talk about nodejs, mongodb and other stuff too much. Just know that the keyboard tracker api is an Express app that works like any other api. You can create, read, update and delete entries into a mongodb database through calls.

If you want more information, [checkout this tutorial](https://medium.com/@dinyangetoh/how-to-build-simple-restful-api-with-nodejs-expressjs-and-mongodb-99348012925d). I actually learned from this example to build a little app a while ago. It's not the greatest app but it's mine.

Our directory structure should look like this:

```bash
smol-workshop/
├── docker-compose.yml
├── Dockerfile
├── package.json
├── package-lock.json
├── README.md
├── src/
└── tinyapp/ ## Ignore tinyapp
```

Let's take a look at that Dockerfile and talk about what's different between this one and the previous tinyapp one we made:

```Dockerfile
FROM node:latest

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 8080
CMD [ "npm", "start" ]
```

Look at that! It's basically the same except we're using port 8080.

Now that we have the Dockerfile, we will need a docker-compose.yml file. This file will hold the instructions for building our multi-container docker application. Lets take a look at it:

```yml
version: "3"
services:
  app:
    container_name: keyboard-api
    restart: always
    build: ./
    ports:
      - "8080:8080"
    volumes:
      - .:/app
    depends_on:
      - mongo
  mongo:
    container_name: mongo
    image: mongo
    ports:
      - "27017:27017"
```

In this file, we declare two sevices `app` and `mongo`. I'm grabbing the instructions above and putting them into one line for readability. Lets break this file down starting with `app`:

- `container_name: keyboard-api` We set this to be keyboard-api so docker doesn't automatically set the container name for us
- `restart: always` This allows us to restart the server whenever an error code gets thrown. Many will get thrown as you work on your API so its best to leave this on always
- `build: ./` This specifies what directory to do a build command on. Remember when we did that __docker build__ command in tinyapp? This option does the same thing
- `ports: 80:8080` This is the same as the `-p 80:3000` flag we set on __docker run__ in tinyapp above. We set the host (your computer) port to 80 and bind port 3000 within our container to it. This letts our container talk to our host machine
- `volumes: .:app/` A volume is a directory we specify to be mounted onto our container. This allows us to share files between the host and the container between containers. In this case, we're adding our smol-workshop folder to a folder inside our container called app/. Remember that is the WORKDIR we set within our Dockerfile!
- `depends_on: mongo` We add this to make sure that our mongo container spins up first before our keyboard-api container. That way we can guarantee that mongodb is ready to be connected to from our api

And for `mongo`:

- `container_name: mongo` Same thing as above. We add a name ourselves so docker doesn't
- `image: mongo` This pulls the official mongodb image from Docker Hub. Similar to our FROM command inside the Dockerfile
- `ports: "27017:27017"` This binds the port 27017 on our host to the mongodbs default port inside the container. We're basically using our host as a network bridge between containers.

Now comes a part I always forget, setting the mongodb connection URL inside our app to point to the mongodb container.

Inside `src/index.js`, scroll down to this line:

```javascript
// Connect to Mongoose and set connection variable
mongoose.connect('mongodb://localhost:27017/expressmongo');
```

And change it to point to our new mongo container's IP. `mongo` will work similarly to `localhost` but instead of 127.0.0.1 it will be whatever IP address our mongo container has. Change it now:

```javascript
// Connect to Mongoose and set connection variable
mongoose.connect('mongodb://mongo:27017/expressmongo');
```

Now we're ready to spin up our containers! Let's go to our terminal run the docker compose file:

```bash
$ docker compose up
```

A bunch of stuff should happen now as our containers are built and ran! If all goes well, you should see the following towards the bottom of all those lines that popped up:

```bash
keyboard-api | Running RestHub on port 8080
```

You can also run it in __detached__ mode by adding the __-d__ flag at the end `docker-compose up -d`

Let's check out our API!

These are the URIs:

__GET__ /api/keyboards
__POST__ /api/keyboards
__GET__ /api/keyboards/:id
__PUT__ /api/keyboards/:id
__PATCH__ /api/keyboards/:id
__DELETE__ /api/keyboards/:id

You can either test it in postman or through curl. In this case, we'll use curl:

```bash
$ curl -X POST \
  http://localhost:8080/api/keyboards \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'name=Planck&style=ortholinear&switch=Zealio%2065g'
```

This will add a new keyboard entry into mongodb. Lets retrieve it!

```bash
$ curl -X GET \
  http://localhost:8080/api/keyboards/

{"status":"success","message":"Sweet keyboards successfully retrieved!","data":[{"_id":"5c8413f28eb193001ed66783","create_date":"2019-03-09T19:28:50.537Z","name":"Planck","style":"ortholinear","switch":"Zealio 65g","__v":0}]}
```

You just got yourself a nodejs express api with mongodb! yay :smile:

## Troubleshooting

- If `docker-compose up` is being funky with mongodb, double check you set the right uri in your mongoose.connect and run `docker-compose build` to rebuild.