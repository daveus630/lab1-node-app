# aks containerization lab

## Goal

To demonstrate how to create Docker images of existing applications (React & Node) and run them inside containers.

This will also be an introduction to Docker and Docker Hub.

Jump to:

1. [Lab 0 Prerequisites](#lab-0-prerequisites)
1. [Lab 1 Node Application](#lab-1-node-application)
1. [Lab 2 React Application](#lab-2-react-application)
1. [Lab 3 Multiple Stage Build for React](#lab-3-multiple-stage-build-for-react)

## Lab 0 Prerequisites

Make sure you have followed all of the necessary [AKS booster prerequisites](https://mfc.sharepoint.com/sites/MU/SitePages/Canada-Programs-AKS.aspx#pre-installation-list) to get Docker up, running can configured on your local machine.


**Note:** Make sure you have your `.npmrc` file handy since it'll need to be copied into the local folder of each project.  The reason for this is we'll be creating an image, which will require `npm  install` to run.



## Lab 1 Node Application

### Preparation

In this lab we're going to clone an existing Node JS application, configure it and deploy to your local Docker container.

1. Log into [github](https://github.com/manulife-ca-sandbox) and change you organization to **manulife-ca-sandbox**.  Search for **YOUR OWN COPY** of the starter coach repo [mu-aks-starter-api](https://github.com/manulife-ca/mu-aks-starter-api). Your copy should follow the naming convention of `mu-aks-<location>-<##>-<lanid>-api`. Make sure to clone this repo to your local machine.
    - `<location>` Location where the MU program is hosted. This value could be `kw` (MU Kitchener) or `ph`(MU Manila)
    - `<##>` is your cohort number
    - `<lanid>` is your LAN ID

1. Open up the application in Visual Studio Code and a new terminal window.

1. On line 2 `package.json`,  replace the `<lanid>` in `<lanid>-mu-aks-api` to your LANID.

    **Note: `<lanid>-mu-aks-api` will be the name of your image going forward**

1. Copy our local `.npmrc` file into the root folder of this project.

1. Install your dependencies:

   ```bash
   npm install
   ```

1. Run your application:

   ```bash
   npm run start
   ```

1. Open up a new browser and navigate to `localhost:3001/api`. The following message should appear on a rather empty page: **Welcome to the employee service - IsValid = true**

1. Stop your application from running.

1. Add a new file to your application root folder called `Dockerfile`.

   You can do it via commandline:

   ```bash
   touch Dockerfile
   ```

   [<img alt="lab1-dockerfile" src="images/lab1-dockerfile.png" height="250px">](images/lab1-dockefile.png)

1. Open up the file and let's start adding to it! The first thing we want to define is the base image of the `OS` we want our application to run in:

   ```Dockerfile
   # pull official base image
   FROM node:16.15-alpine
   ```

   We're basically telling Docker to pull the base image, `node:16.15-alpine`, to be used for our application.

1. Now set the working directory for the application inside our image:

   ```Dockerfile
   # set working directory
   WORKDIR /usr/src/app
   ```

1. Let's copy `.npmrc`, `package.json` and `package-lock.json` into the image to use for dependency installation later:

   ```Dockerfile
   # copy .npmrc to get npm packages from Manulife Artifactory
   COPY .npmrc ./

   # copy package.json and package-lock.json to get dependencies
   COPY package*.json ./
   ```

1. Since this is a node app, let's install dependencies for the application:

   ```Dockerfile
   # install npm dependencies
   RUN npm ci
   ```

1. Let's copy over our the actual project code:

   ```Dockerfile
   # copy other project files (unless ignored in .dockerignore)
   COPY src ./src
   ```

   > Note that we copy `package*.json` files and install dependencies before we copy the project code, and this
   > for a very important reason: caching. Each command in a Dockerfile is basically creating a new image, or
   > _layer_, and docker actually keeps a cached version of each layer to make future builds faster. Each cache
   > layer has a checksum mechanism to detect changes so that the cache is invalidated **for the changed layer and all the following layers**. By copying code after installing dependencies, future builds can just use the
   > dependency installation layer from last time (which saves a lot of time!) and then only create a new layer
   > for copying new code files.

1. Let's lets add metadata to the image that any containers created from it will be listening on port 3001:

   ```Dockerfile
   # describe that the container is listening on port 3001
   EXPOSE 3001
   ```

1. Finally we need for a way to start up our application:

   ```Dockerfile
   # start container with npm run start
   CMD [ "npm", "run", "start" ]
   ```

   Note: This will build and run the application.

1. Your final `Dockerfile` will now look like the following:

   ```Dockerfile
   # pull official base image
   FROM node:16.15-alpine

   # set working directory
   WORKDIR /usr/src/app

   # copy .npmrc to get npm packages from Manulife Artifactory
   COPY .npmrc ./

   # copy package.json and package-lock.json to get dependencies
   COPY package*.json ./

   # install npm dependencies
   RUN npm ci

   # copy other project files (unless ignored in .dockerignore)
   COPY src ./src

   # describe that the container is listening on port 3001
   EXPOSE 3001

   # start container with npm run start
   CMD [ "npm", "run", "start" ]
   ```

1. Let's add a `.dockerignore` file to our application's root folder to exclude files/folders we don't need. We want to keep the image as clean as possible!

   Here are the lines we'll need to add in your new `.dockerignore` file:

   ```gitignore
   node_modules
   npm-debug.log

   Dockerfile*

   .git
   .gitignore

   .prettierrc
   .vscode
   ```

   Note - this is very similar in principle to `.gitignore`.

1. Finally you're ready to build your image!

### Building

Before starting, let's run the follow command to check out which `Docker` images you have available in your local repo:

```bash
docker images
```

You should see something similiar to the following output:

```bash
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
docker/getting-started   latest              1f32459ef038        3 months ago        26.8MB
registry                 latest              2d4f4b5309b1        4 months ago        26.2MB
```

Next, run the following to get a list of containers you have running locally:

```bash
docker container ls
```

Something like this will show up (assuming you started a container with the `docker/getting-started` image):

```bash
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS              PORTS                NAMES
e4495656585b        docker/getting-started   "/docker-entrypoint.…"   3 days ago          Up 3 days           0.0.0.0:80->80/tcp   jovial_germain
```

If you connect the 2 outputs, you can very quickly see that container with alias `jovial_germain` with id `e4495656585b` is running the image `docker/getting-started`.

1. Open up a new terminal window and make sure you `cd` into your application's root directory. Run the following command to build you image using the `Dockerfile` you created in the previous section, **replacing <PLACEHOLDER> with your lan ID**:

   ```bash
   docker build -t <PLACEHOLDER>-mu-aks-api:1.0 .
   ```

   - the `build` command tells `Docker` to build.
   - the `-t` gives a **name** (<PLACEHOLDER>-mu-aks-api) and a **tag** (1.0) to your image.
   - the `.` tells docker to use the current directory as the **build context** with the default `Dockerfile`.

   Depending on your processing power and internet speed, you may have to wait between 20 to 60 seconds. A whole bunch of options will appear on screen, which happens to be the console output of the image as it's executing the commands in your `Dockerfile`.

   > **Pro tip:** Add a new npm script to your `package.json` file with the above command, like so:
   >
   > `"docker:local:build": "docker build -t <PLACEHOLDER>-mu-aks-api:1.0 ."`
   >
   > That way you can save future typing, and also even if you forget this command in the future, you can
   > simply run it with:
   >
   > `npm run docker:local:build`

1. Next run the command to view your image:

   ```bash
   docker images
   ```

   OR

   ```bash
   docker image ls
   ```

   You should have at least 1 newly created image, which is your `<PLACEHOLDER>-mu-aks-api`. You may also see another image called `node` which is the base image used to build your `<PLACEHOLDER>-mu-aks-api`.

   ```bash
   REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
   <PLACEHOLDER>-mu-aks-api 1:0                 032bb9080ee0        2 minutes ago       272MB
   node                     alpine              fa2fa5d4e6f4        3 days ago          117MB
   docker/getting-started   latest              1f32459ef038        3 months ago        26.8MB
   registry                 latest              2d4f4b5309b1        4 months ago        26.2MB
   ```

1. Let's run your image:

   ```bash
   docker run -p 3001:3001 <PLACEHOLDER>-mu-aks-api:1.0
   ```

   Here's what's happening with the commands:

   - `-p` tells docker to do a **host:container** port mapping, in this case both are 3001.
   - `<PLACEHOLDER>-mu-aks-api:1.0` is a reference to a specific image defined by its name and tag (1.0).

   > If you want you can also add the switch `-d` to tells the container to run in _detached mode_, so you can
   > still use your terminal.

   > **Pro tip:** Add a new npm script to your `package.json` file with the above command, like so:
   >
   > `"docker:local:run": "docker run -p 3001:3001 <PLACEHOLDER>-mu-aks-api:1.0"`
   >
   > That way you can save future typing, and also even if you forget this command in the future, you can
   > simply run it with:
   >
   > `npm run docker:local:run`

1. Open up a browser and navigate to `localhost:3001/api`. Again, the following message should appear: **Welcome to the employee service - IsValid = true**

1. You can also list the current running containers to see your container. In another terminal, run the command:

   ```bash
   docker container ls
   ```

   There should now be a new container that is running your image:

   ```bash
   CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS                    NAMES
   ebf2fd624754        ...-mu-aks-api:1.0     "docker-entrypoint.s…"   10 seconds ago      Up 9 seconds        0.0.0.0:3001->3001/tcp   busy_jones
   ```

   Notice the `PORTS` column which tells you how the ports are mapped. Requesting from your local machine `0.0.0.0:3001` will forward the request to the container under port `3001`, which happens to be the default port that your `<PLACEHOLDER>-mu-aks-api` should be running on.

   1. You can terminate the application with `Ctrl+C` on the terminal that is running the container.

## Lab 2 React Application

In this lab we're going to clone an existing React application, configure it and deploy to your local Docker container.

1. Log into [github](https://github.com/manulife-ca) and change you organization to **manulife-ca**.  Search for your copy of `<PLACEHOLDER>-mu-aks-react` and clone it to your local machine.
    - `##` is your cohort number
    - `<PLACEHOLDER>` is your usename

1. Open up the application in Visual Studio Code and a new terminal window.

1. On line 2 `package.json`,  replace the `<lanid>` in `<lanid>-mu-aks-react` to your LANID.

    **Note: `<lanid>-mu-aks-react` will be the name of your image going forward**
1. Copy your local `.npmrc` file to the root folder of this project.

1. Install your dependencies:

   ```bash
   npm install
   ```

1. Run your application:

   ```bash
   npm run start
   ```

1. A new browser window will open with your application.

   Here is the screen you should be getting:

   [<img alt="lab1-ui" src="images/lab1-ui.png" height="400px">](images/lab1-ui.png)

   NOTE: remember the port number that it's running under as you'll need this.

1. Stop your application from running.

1. Add a new `Dockerfile.local` file with the following contents:
   > **Important note**: We're naming the file Dockerfile.local because this is a development build of the our react application.  We wouldn't be deploying this version to production.

   ```Dockerfile
   # pull official base image
   FROM node:16.15-alpine

   # set working directory
   WORKDIR /usr/src/app

   # copy .npmrc to get npm packages from Manulife Artifactory
   COPY .npmrc ./

   # copy package.json and package-lock.json to get dependencies
   COPY package*.json ./

   # install npm dependencies
   RUN npm ci

   # copy other project files (unless ignored in .dockerignore)
   COPY src ./src
   COPY public ./public
   COPY .env ./

   # describe that the container is listening on port 3000
   EXPOSE 3000

   # start container with npm run start
   CMD [ "npm", "run", "start" ]
   ```

   Here's what's happening:

   - we're using `node:16.15-alpine` as our base image
   - we want to copy all files over to the `WORKDIR`
   - the applcation will run out of the `/usr/src/app` folder in our base image
   - `COPY` to copy any necessary files we need
   - `RUN` to install all dependencies
   - just like in lab 1, we install dependencies before copying project code files for better caching performance
   - `CMD` to start up the our Node application

1. As we've done in the previous exercise, let's add a `.dockerignore` file to our root folder with the typical items to exclude in the creation of our `Docker` image:

   ```yaml
   node_modules
   npm-debug.log

   Dockerfile*

   .git
   .gitignore

   .prettierrc
   .vscode

   build
   ```

1. Let's build our image, **replacing <PLACEHOLDER> with your lan ID**:

   ```bash
   docker build -t <PLACEHOLDER>-mu-aks-react:local -f Dockerfile.local .
   ```

1. And run it:

   ```bash
   docker run -p 3000:3000 <PLACEHOLDER>-mu-aks-react:local
   ```

   Note: remember to use the port your React application is running on, `3000`, and a port that does not conflict with your `<PLACEHOLDER>-mu-aks-api`. eg. port `3000` should work!

   > **Pro tip:** Add npm scripts to your `package.json` file with the above build and run commands, like so:
   >
   > `"docker:local:build": "docker build -t <PLACEHOLDER>-mu-aks-react:local ."`
   >
   > `"docker:local:run": "docker run -p 3000:3000 <PLACEHOLDER>-mu-aks-react:local"`

1. Open up a web browser window and navigate to `http://localhost:3000`

   Your webpage should now be loaded up!

1. On a different terminal, run the API from the previous exercise (using its docker image) so that the React app can retrieve data from it.

1. Now click on **Load Employees** to load employees from the API.

1. You should see the following page:

   [<img alt="lab3-landing" src="images/lab3-landing.png" height="500px">](images/lab3-landing.png)

1. When you click on one of the employees, it should load that employee's details and show it:

   [<img alt="lab3-details" src="images/lab3-details.png" height="520px">](images/lab3-details.png)


1. Finally you can remove the container by stopping the process in the terminal window where it's running from.


## Lab 3 Multiple Stage Build for React

The thing to note with the previous step is we're running our `React` application under a development environment. Remember that React app is a browser app, meaning **it's just HTML, CSS and JavaScript**. When you do `npm start`, it's just running a non-production-ready local web server to serve the React app for development. So what we did above shouldn't be used for actual deployments. Additionally, the image is fairly bloated due to loads of dependencies in `node_modules` that are only for development.

A big challenge to building images is to keep their size to a minimum. Each `Dockerfile` instruction adds an extra layer to the image, so it's always a good idea to do clean-up of anything you don't need before moving to the next layer.

This is known as the [multi-stage build pattern](https://docs.docker.com/develop/develop-images/multistage-build/).

The goal in this lab exercise is build a temporary image, which is used to build our `artifact` --- the production ready `React` static files. Next we copy the `React` static files to a production image and discard the rest (including the tempoary image).

1. In your starter react app `<PLACEHOLDER>-mu-aks-react` and create a new file called `Dockerfile` in your root directory:

   [<img alt="lab3-landing" src="images/lab3-dockerfile.png" height="200px">](images/lab3-dockerfile.png)

1. For references only, here are the contents from our original `Dockerfile.local`:

   ```Dockerfile
   # pull official base image
   FROM node:16.15-alpine

   # set working directory
   WORKDIR /usr/src/app

   # copy .npmrc to get npm packages from Manulife Artifactory
   COPY .npmrc ./

   # copy package.json and package-lock.json to get dependencies
   COPY package*.json ./

   # install npm dependencies
   RUN npm ci

   # copy other project files (unless ignored in .dockerignore)
   COPY src ./src
   COPY public ./public
   COPY .env ./

   # describe that the container is listening on port 3000
   EXPOSE 3000

   # start container with npm run start
   CMD [ "npm", "run", "start" ]
   ```

   The following is the content to be added to your new `Dockerfile`:

   ```Dockerfile
   # build environment
   FROM node:16.15-alpine AS builder
   WORKDIR /usr/src/app
   COPY .npmrc ./
   COPY package*.json ./
   RUN npm ci
   COPY src ./src
   COPY public ./public
   COPY .env ./
   RUN npm run build
   ```

   Here's what's happening:

   - `as builder` is added to `FROM` to mark a build step/layer
   - `/usr/src/app` is set as the working directory
   - copy both `package.json` and its "lock" file, and `.npmrc`
   - `RUN npm ci` does an clean install of dependencies eg. deletes `node_modules` before installing
   - copy project code files
   - `RUN npm run build` will build the static _deployable_ artifact of the React app in the `/build` folder

1. Next, let's add another stage to our `Dockerfile` file:

   ```Dockerfile
   # production environment
   FROM nginx:1.19-alpine
   COPY --from=builder /usr/src/app/build /usr/share/nginx/html
   COPY ./nginx/default.conf.template /etc/nginx/templates/
   EXPOSE 80
   CMD ["nginx", "-g", "daemon off;"]
   ```

   Here's what's happening:

   - the `FROM` command will use the `nginx:1.19-alpine` image
   - we `COPY` the `React` static files from the previous stage (or temporary image) located at `/usr/src/app/build` into our new image under `/usr/share/nginx/html`, which is the location from which nginx serves the web page
   - `EXPOSE 80` means your app is running on the server under the nginx default port of `80`
   - finally we execute `nginx`, _without daemonizing_ --- do you know why? ;)

   > Check out the nginx configuration template file `nginx/default.conf.template`. If you've never used nginx
   > before it will seem daunting. Take your time, it has a lot of comments. You may not understand everything
   > in 1 day, but eventually it'll settle in :)
   >
   > Also note that it actually uses a feature that's available in the nginx 1.19+ docker image, which will
   > actually text replace strings in the file from environment variables being passed to the actual container
   > (i.e. in runtime, not build time). We're using that feature to put the API URL, which differs per
   > deployment. The feature is documented at the following link: https://github.com/docker-library/docs/tree/master/nginx#using-environment-variables-in-nginx-configuration-new-in-119

1. Finally let's build and tag the Docker image:

   ```bash
   docker build -t <PLACEHOLDER>-mu-aks-react:1.0 -f Dockerfile .
   ```

1. Finally let's run the image in a container:

   ```bash
   docker run -p 80:80 -e API_URL=http://localhost:3001/api <PLACEHOLDER>-mu-aks-react:1.0
   ```

   We're **setting an environment variable here!** This is because of how the React app and nginx are configured, to allow injecting **API_URL** into the React app for different environments.

   > **Pro tip:** Add npm scripts to your `package.json` file with the above build and run commands, like so:
   >
   > `"docker:local:buildprod": "docker build -t <PLACEHOLDER>-mu-aks-react:1.0 -f Dockerfile ."`
   >
   > `"docker:local:runprod": "docker run -p 80:80 -e API_URL=http://localhost:3001/api <PLACEHOLDER>-mu-aks-react:1.0"`

1. Open up a web browser window and navigate to http://localhost

   Your webpage should now be loaded up!

1. Run the following command to pull up some stats on your images:

   ```bash
   docker images
   ```

   OR

   ```bash
   docker image ls
   ```

1. Find `mu-aks-react` under `RESPOSITORY` with `1.0` as its `tag`. Notice how TINY it is!

   ```bash
   REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
   mu-aks-react             1.0                 36003937017d        20 minutes ago      28.8MB
   mu-aks-react             local               c03aa07854ee        13 days ago         322MB
   ...
   ```

   Compare that to the `local` version of `mu-aks-react`, which is the development environment, including the source code and supporting libraries eg. `node_modules`.

   If you're keen, you'll also notice the temporary image that was used to build your `prod` image; it has a name of `<none>`. Its size is similar to the `local` version of your `mu-aks-react`. You'll also see `nginx` and `node` in there.

