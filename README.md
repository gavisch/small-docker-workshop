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
mkdir tinyapp
cd tinyapp
npm init -y
```

Now we need to install Express, a minimalistic web framework for node [(click here info)](https://expressjs.com)

```bash
npm install express
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
touch app.js
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
npm start
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
mkdir Dockerfile
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
touch .dockerignore
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
docker build -t your_dockerhub_username/tinyapp-demo .
```

It'll take a few minutes, but once that's over we can look at our image:

```bash
docker images
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
docker run --name tinyapp-demo -p 80:3000 -d your_dockerhub_username/tinyapp-demo
```

Once the container is running, you can check it out and other containers that are running with __docker ps__:

```bash
docker ps
```


# TODO Consider making the nodejs app real quick, adding it to repo then just referencing things about it

## Commands to remember


## Troubleshooting

- If `docker-compose up` is being funky with mongodb, double check you set the right uri in your mongoose.connect and run `docker-compose build` to rebuild.