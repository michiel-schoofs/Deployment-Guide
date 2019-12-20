# Cent-OS Setting up everything and fi(ni)shing

## Desktop installation because I need salvation

Follow the Cent-OS installation, just click next a couple of times and be sure the internet configurations are set correctly.

If the Cent-OS installation only contains a kernel without a GUI you can install GNOME. A Guide on how to do that can be found [here]( https://www.itzgeek.com/how-tos/linux/centos-how-tos/install-gnome-gui-on-centos-7-rhel-7.html ). GNOME makes the configuration a bit less of a pain in the ass to complete. The following installation guide will assume you have GNOME installed.

## Git installation and default configuration to avoid a moronic situation

Be sure you're the root user of the system you can do that using the following commando:

```bash
su root
```

Enter the root password and enjoy being a ***SUPER USER*** 

**note**:  if you run all the commands in root user you can't run docker in your user account. If you don't want this just preface all commands with sudo that way it's all configured under your local user account. In order to run commands with sudo you need to add your user to the group **wheel**. you can do this by switching to root and running the command `usermod -aG wheel (username of your account)` 

update your Cent-OS installation to the latest version using yum (default package manager for Cent-OS). Be sure you're connected to the internet otherwise you can't connect or complete the following steps. If you're not connected to the internet you can check the settings in: **Applications**  ➜  **System tools** ➜  **Settings**

```bash
yum check-update
yum update
```

Next up the fun stuff begins time to step into the developer zone by installing git. Installing git is easy as pie

```
yum install git
git --version
```

By default however your git isn't configured to your account so lets do the proper way of adding an ssh key in order to get everything up and running note that the following instructions can also be found [here](). It's advised to run this step under your local account (in my case san). 

```bash
su san
ssh-keygen -t rsa -b 4096 -C "michiel.schoofs@student.hogent.be"
```

The key-gen command uses following arguments:

- -t is the encryption algorithm being used in our case we're going to use RSA since that's what GitHub recommends us doing
- -b is the amount of bits generated, again 4096 is the recommended amount by GitHub
- -C the email you're generating the key for

The key will be generated (Do not enter a password if you want to make your life easier while pulling). The default installation location is `/home/(name of your user)/.ssh/` . You will now have to files a public key which you'll give to GitHub and a private key to verify it's you! (<u>**never­**</u> leak this one). Navigate to where you've saved your ssh key and verify that both files are there:

```
cd ~/.ssh/
ls -a
```

Now we need to tell our Cent-OS we're using an ssh key and where this one can be found. This is done using an ssh-agent. To run our agent and add our key run the following commands:

```
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa
```

Now we need to copy the contents of our public key into our GitHub itself depending on whether or not you have gnome this step might be tedious as you'll have to copy over the entire key to a browser (so you might have to retype the key in your windows,I hope you have gnome :) ). Make a new key in  [your profile settings](https://github.com/settings/profile) ➜ **SSH and GPG keys** ➜ **new SSH key** give it a catch name like *Unicorn kitties* and copy the contents of your id_rsa.pub file in the Key section. Finally lets connect and verify everything is setup correctly by running the following command:

```bash
ssh -T git@github.com
```



## Installing docker containers and containing yourself

### Docker installation

Docker requires some libraries in order to work so lets get those shall we?

```
yum install -y yum-utils 
yum install -y device-mapper-persistent-data 
yum install -y lvm2
```

Next up is loving and praying to docker and adding it to know repositories so we can install from it

```
yum-config-manager --add-repo  https://download.docker.com/linux/centos/docker-ce.repo
```

Time to dock home and install the actual docker files and start the docker service:

```
yum install docker-ce docker-ce-cli containerd.io
systemctl start docker
```

Docker comes with it's own version of hello world to verify the installation you can run the following command:

```
docker run hello-world
```

### SQL server on LINUX???????

Believe it or not cats and dogs can collide with each other time to install a Microsoft product on Linux! (kudos to Pieter for the guide). The first step is getting the container this is in all actuality just a self contained windows installation with SQL server.  So it's not like cats and dogs but more like a difficult divorce settlement. 

```
docker pull microsoft/mssql-server-linux:2017-latest
```

This will install the container itself however note that this is only an image for a container (just like an iso file). This doesn't run the container yet so lets get the container up and functional:

```
docker run -d --name SQLServer -e 'ACCEPT_EULA=Y' -e
'SA_PASSWORD=p@ssw0rd' -e 'MSSQL_PID=Developer' -p 1433:1433 microsoft/mssql-server-linux:2017-latest
```

Let's breakdown this command to see the different configurations we're doing here:

- `Docker run` is to start a container up

- -d starts the container in detached mode this makes it that our console isn't attached to the container itself (it's running on the background) this however means that any error messages won't pop up so be careful. For debugging purposes you can drop this argument so you get an error output if there is one
- -e here we set a variable **inside** the container itself, the variable `ACCEPT_EULA` we set is to accept the user agreement. We also set the  password for our sysadmin in the variable `SA_PASSWORD` . The variable for `MSSQL_PID` sets the version of SQLServer we want to run in our case developer edition. You can see all the variables [here]( https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-configure-environment-variables?view=sql-server-ver15 ) 
- -p exposes specific ports to the host of our docker container so we can interact with our container on port `1433`.
- Finally the name of our container self: `microsoft/mssql-server-linux:2017-latest`

To verify that the container is running you can use the command `docker ps -a`. Always make sure that the machine you're trying to run the docker container on has **at least** 2GB of Ram. You can also specify a specific amount of memory using the -m argument when setting up your containers. 

## Notes about docker. Anchoring yourself in the deep end.

The way docker works is that by running the previous command you've made a container or a self contained windows installation with SQL server on it. This container exists so running the above command more then once will tell you the container already exists. Here are a couple of commands to deal with docker and your container once it's setup.

| **Command**                          | **Description**                     | **Example**            |
| ------------------------------------ | ----------------------------------- | ---------------------- |
| docker start ***name of container*** | Starts the container you specified  | docker start SQLServer |
| docker ps -a                         | Gives the status of your containers | docker ps -a           |
| docker rm ***name of container***    | Removes the specified container     | docker rm SQLServer    |

Also note that by default after a reboot the docker service is not running you can make docker run at default by writing a startup script (See later). You can also manually start/stop the service by running 

```
systemctl start docker
systemctl stop docker
```

# Running from behind (.Net back-end)

## Installing .NET

So the magic has happened and we have a running SQLServer now for actually getting the repository up and running. However we don't really have .net Core installed currently so lets get that up first. Please note that the following installation steps have only been verified for a .net core application so if you're trying to run an actual .Net application I have no clue if this will also suffice for you. First off we have to connect to the Microsoft repositories (note this was more or less the same for our docker installation).

However we'll use RPM which is shorthand for redhead package manager. Run the following commands as root:

```
rpm -Uvh https://packages.microsoft.com/config/rhel/7/packages-microsoft-prod.rpm
```

Now these packages are added to the repositories we can install from, lets also make sure our package manager is completely up to date and knows what packages are in the Microsoft Repos:

```
yum makecache
```

Verify the version of .Net you actually need you can find those by right clicking on your project in visual studio and going to properties. To list all available .Net versions you can install for your .Net project you can run the following command:

```
yum list available | grep dotnet-sdk
```

In our specific case we need version 2.1 so lets install it and verify it works

```
yum install -y dotnet-sdk-2.1
dotnet --version
```

## Setting up our project

The following steps can be done using your normal user (in my case San) so lets switch back and go to our home directory:

```bash
su san 
cd ~
```

Lets clone our Repo, be sure you've setup the ssh key in order to connect to your GitHub account otherwise the gods will frown upon you and smite you with an unauthorised exception. I also setup folders just to organise stuff later on:

```
mkdir repos
mkdir repos/WebAPI
cd repos/WebAPI
git clone git@github.com:HoGent-Projecten3/projecten3-1920-backend-grasmaaier-team.git
//The version of the branch you wanna build your project off
git checkout develop
git pull
```

Now we need to setup a connection you can set the connection up in your appsettings.json . We're putting the connection string there but basically navigate to the folder where your connection string is located and open up the file in a text editor of your choice (we're using gedit but if you're hardcore and not a pussy like me go for vim):

```
cd projecten3-1920-backend-grasmaaier-team/KolveniershofAPI/
gedit appsettings.json
```

Change the connection string into the following:

```json
  "ConnectionStrings": {
    "DefaultConnection": "Server=127.0.0.1,1433; User Id=sa;Password=p@ssw0rd;Database=Kolveniershof"
  }
```

You'll recognise a couple of elements in this connection string:

- We're running the docker container with SQLServer locally so our localhost is present `127.0.0.1` 

- The port we setup when configuring our container is also here: `,1443`

- We're logging in as the system administrator since we setup a password for that user: `sa`

- The password for our administrator: `p@ssw0rd`

- Finally the name of our database: `Kolveniershof`

  

Next up is the user secrets. This is not necessary for people who don't use a JWT token but basically we encrypt our token using a user secret the way you do this on Cent-OS is:

```
dotnet user-secrets set "Tokens:Key" "HaveYouHearedThatUnicornKittiesAreTheBest"
```

So now we're all set for running our Back-end first lets make an executable (.dll) out of your code you can do this by running the following command in the folder containing all your files (where appsettings.json is located) by running the following command:

```
dotnet publish -r centos.7-x64
```

The -r specifies what we're building for in our case we're building our dotnet application for centos.7-x64. Normally you should now have a folder bin in the project where a dll is located you can run this dll that's your project. Run the dll file itself.

```
cd bin/Debug/netcoreapp2.1/centos.7-x64/publish/
dotnet KolveniershofAPI.dll
```

If you have some issues fix them. Issues we encountered for example:

- Relative paths and case sensitive files for example Bakken.JPG is not the same on linux as Bakken.jpg while it doesn't matter on windows. To fix this issue run following commands:

  ```
  //Navigate back to the base directory with the appsettings.json
  cd ../../../../../
  cd Data/Seeding/Pictos
  rename -v .JPG .jpg *.JPG 
  //Now to fix the paths So we're in our seeding directory
  cd ..
  nano DataInit.cs
  //replace the line 
  byte[] bytes = File.ReadAllBytes($"Data\\Seeding\\Pictos\\{fotoNaam}.jpg");
  //to
  byte[] bytes = File.ReadAllBytes($"Data/Seeding/Pictos/{fotoNaam}.jpg");
  ```

- Some files are not included in our build the fix we found was based on the suggestion found [here]( https://stackoverflow.com/questions/54762744/net-core-include-folder-in-publish ). Navigate back to your root directory. From there edit the KolveniershofAPI.csproj. 

  ```
  vim KolveniershofAPI.csproj
  //add the following lines right before the </Project>
  <ItemGroup>
      <Content Include="Data\Seeding\Pictos\**">
          <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>     
      </Content>
  </ItemGroup>
  
  ```

- User secrets don't work that well! So what we've done is just paste the following lines inside of the appsettings.json:

  ```json
  //navigate to the root directory of your project
  vim appsettings.json
  //paste the follwing code just before the last bracket:
  ,"Tokens":{
          "Key":"HaveYouHearedUnicornKittiesAreAwesome?"
  }
  
  ```

- After all changes make sure you republish your project and also make sure you're running the project as your root user

  ```
  su root
  dotnet publish -r centos.7-x64
  cd bin/Debug/netcoreapp2.1/centos.7-x64/publish/
  dotnet KolveniershofAPI.dll
  ```

Normally now everything should be setup correctly and your back-end should be running just fine and the port it's running on should be displayed. Now for the testing just open your browser and navigate to the following address: `https://localhost:5001/swagger/index.html` note that our port is 5001 but this might differ in your installation.

# Running the angular application

So now that we've got the back-end working properly it's time to start with the Front-End for our example we'll be using an Angular application so all of the following steps apply to angular:

## Getting the project

Again we've already setup the github connection so let's make a new folder and get the actual stuff we're going to build:

```
cd /home/san/Desktop/repos/
mkdir AngularApplication
cd AngularApplication/
git clone git@github.com:HoGent-Projecten3/projecten3-1920-angular-grasmaaier-team.git
cd projecten3-1920-angular-grasmaaier-team/
git checkout master
git pull
```

Note that we're building our current angular application from the master branch. Possibly you might want to consider using a different branch like building of develop.

Verify everything is up to date by using `git status`

## Installing dependencies

We need a couple of things to setup our angular application:

- Node package manager and node.js (used for building and getting dependencies)
- Angular CLI (ng) (used for angular duuuh)
- The dependencies of your angular project

So lets focus on getting the node.js and NPM installed conveniently enough for us those are already included in one neat package called node.js . However we don't have a reference to where that package is located. We've already done something similar with adding the Microsoft repository a while back. The last two commands verify that we've installed both of them.

```
curl -sL https://rpm.nodesource.com/setup_10.x | sudo bash -
sudo yum install -y nodejs
node -v
npm -v
```

Next up the Angular CLI if the last command shows a list of available commands you can pat yourself on the back since you've installed it correctly.

```
sudo npm install -g @angular/cli
ng
```

Well that was easy now for the actual dependencies of your project you can navigate to the root folder (containing src and e2e) and run the command `npm install` 

## Running the web application after installation

First thing first lets test if all packages are installed by running the command `ng version` this should give a list of all the available dependencies if there's an error something went wrong with running the previous commands.

The actual build process might differ for you but for us we need the following commands to run our application `npm start` . The web address where the application is hosted will appear inside of the command line. Just verify it's running by entering it into your browser. `http://localhost:4200/login`

The web page showed but  the actual application doesn't work!! How come? Well time for some debugging. Open the development console in your browser (we're using firefox since that's the browser of choice in GNOME ➜ **ctrl+shift+k**).  We see a lot of 504 errors in our network tab so the connection between the front-end and back-end isn't setup correctly. After quick verification of what's going on we see that the application uses a proxy.conf. Opening the file reveals our mistake. The port isn't configured correctly so lets change it:

```
su san 
cd ~/Desktop/repos/AngularApplication/projecten3-1920-angular-grasmaaier-team/
vim proxy.conf.json
//Replace port 
port = 5001
```

Run your project again and bask in the glory of what is a working website

# Opening up (port forwarding)

Well it's good no doubt that everything is working the way it ought to, however you can't connect to it from the outside. The purpose of your website is that people can visit it no? So lets do some port forwarding. The first step of your endeavour is figuring out what your IP is. You can easily do this by running the command: `ifconfig` it shows me my local adress is 192.168.122.1 you might also want to find your public ip adress just go to `https://whatismyipaddress.com/` to find out that address.

So which ports do we want to open well we want to open 5001 since that's our API port we want to expose and 4200 since angular is running on that one. Lets check the status of both of those ports:

```
netstat -na | grep 4200
netstat -na | grep 5001
```

Normally both of these ports should be active and listing to your request. Now stop both processes (angular and dotnet (**ctrl+c**)) Normally they shouldn't be active anymore so we can configure them correctly.

We need to edit the /etc/services file to include both of these ports and the protocol they'll be using:

```
sudo vim /etc/services
// paste the following two lines at the bottom
dotnet          5001/tcp                # Dotnet API
angular			4200/tcp				# Angular application
```

Now for allowing outside connection:

```
firewall-cmd --zone=public --add-port=5001/tcp --permanent
firewall-cmd --zone=public --add-port=4200/tcp --permanent
firewall-cmd --reload
```

We also need to allow that we can use http and https:

```
sudo firewall-cmd --zone=public --permanent --add-service=http
sudo firewall-cmd --zone=public --permanent --add-service=https
sudo firewall-cmd --reload
```

Finally we can use NGINX to act as a proxy between the two applications lets first install nginx (by adding the repository and installing it).

 ```
sudo rpm -ivh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
sudo yum install -y nginx
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
 ```

# Container diving in docker

## Images and you 

So we've got everything functional but what if we don't want our backend to be on our local machine we also just want to put it in an image. Well luckily we've already seen the concept of docker so lets put it in containers.  First of let me introduce you to the [docker repository]( https://hub.docker.com/) this is a repository full of docker images you can run and pull. Luckily for us this is just like github with push and pull concepts so I won't explain them in depth. To see the images you currently have installed you can run the following command:

```
sudo docker images
```

This lists all available images we can also pull in images we found on our repository:

```
sudo docker image pull busybox
```

This pulls in the busybox image in docker. if you run `sudo docker images` again you will see that this image is added.  To remove an image you can run the command `sudo docker image rm "name of the image"` for example if we wanted to remove busybox we'd run the command `sudo docker image rm busybox`

Now we can run  our docker container with the command

```
sudo docker run busybox
```

To run commands inside of containers we can simply run a command by pasting it behind our container:

```
sudo docker run busybox echo "hello world from busybox container"
```

We can also run our container interactively busybox is a minimal linux distro so let's run our container and preform an ls and echo command:

```
sudo docker run -i busybox
> ls
> echo "test"
> exit
```

**do not forget to delete inactive containers with the docker rm command after you're done**

You can also delete all exited containers with docker prune be careful that you accidently don't delete the wrong containers.

## Writing our own images (example)

So we've seen how to work with images and getting them from the docker repository. You can compare images with physical disks and a container as a working copy of this disk. First thing first we have to explain the concept of parent images. Well you always build on top of a framework for example python, so we need to have python installed in our docker environment. A way would be to use centos install python and run the program but this is really expensive.  A better way would be to use the python image and build the program on top of this image. The python image is a minimal build with only the dependencies necessary to run python. The image of python we're going to be using is the  `python:3-onbuild` image for this example. First we pull in the project and we create what's called a **Dockerfile** this is nothing more or less then a definition of how to build our container.

```
git clone https://github.com/prakhar1989/docker-curriculum.git
cd docker-curriculum/flask-app
touch DockerFile
```

In the **Dockerfile** itself we write the docker definitions:

```
# Specify a base image (in our case a python because we're building a python app)
FROM python:3-onbuild
# Expose a port to the outside since we're hosting a webapplication
EXPOSE 5000
# Tells what command to run when the application is started
CMD ["python", "./app.py"]
```

We can now build our image since we have our Dockerfile itself. To build our own docker we use the following command: `sudo docker build -t sandrageisha/test .`

The -t argument provides a name to our docker container, as the prefix of the name we choose our docker hub name so we can easily push our image to the docker hub repository later on. The second argument `.` is the directory containing our docker file in our case is that the local repository so a .

This command will pull in all of the necessary images and files, these will also be cached so subsequent builds will be significantly lower.

Now that we have our image it's time to run the image itself.

`sudo docker run -p 8888:5000 sandrageisha/test`

note that we map the exposed port 5000 to port 8888. so we can just connect to our web application on http://0.0.0.0:8888/ from within our browser. Awesome and  we even get a cat gif on top of it.

Next up less push our docker image to the dockerhub repository. First off lets login to docker hub (if you don't have an account yet now would be the time)

`sudo docker login`

Do note that your login will be saved in plain text you can also configure a keychain or some secure method of connecting to docker hub. For that you'll have to consult the [documentation](https://docs.docker.com/engine/reference/commandline/login/#credentials-store).

Now for the good part let's push:

` docker push sandrageisha/test` 

## Multi container docker

So now we've build a web app but how about difficult applications like our dotnet backend. We already have a docker container but how do we encapsulate our back-end while still being flexible and supporting docker.

We'll be dockerising a multi layered application written by prakhar. So we'll pull that one in first:

``` 
cd ~/Desktop/dockerTest/
git clone https://github.com/prakhar1989/FoodTrucks
cd FoodTrucks
ls
```

As you can see we have two folders our flask-app and a util folder with several stuff the flask-app needs. So it would only be logical to have two containers one running our flask-app and another one to run our elastic search service.

We've already seen that we can base our flask-app of the python base image but what should we use as the base image for our Elastic Search container? Well we can have a look:

```
sudo docker search ElasticSearch
```

We get a whole list and yes there's an official image! Great so we can use that one as our base. First off lets get the image itself:

```
sudo docker pull docker.elastic.co/elasticsearch/elasticsearch:6.3.2
```

We can try and run this image as well

```
sudo docker run -d --name es -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:6.3.2
```

We've seen all of the command options used so far so I won't go into detail here.

We've got our first container setup correctly. Lets move to the flask-app so now we also unfortunatly need nodejs installed. In order to achieve this we can't simply use our python image anymore so lets work from an ubuntu base image. The Docker file is the following:

```
# start from base
FROM ubuntu:latest

MAINTAINER Prakhar Srivastav <prakhar@prakhar.me>

# install system-wide deps for python and node
RUN apt-get -yqq update
RUN apt-get -yqq install python-pip python-dev curl gnupg
RUN curl -sL https://deb.nodesource.com/setup_10.x | bash
RUN apt-get install -yq nodejs

# copy our application code
ADD flask-app /opt/flask-app
WORKDIR /opt/flask-app

# fetch app specific deps
RUN npm install
RUN npm run build
RUN pip install -r requirements.txt

# expose port
EXPOSE 5000

# start app
CMD [ "python", "./app.py" ]
```

A few things to note here is the MAINTAINER Command who specifies the person responsible for maintaining this docker container, Most of the subsequent commands are run commands or commands executed within the container. Next up is our ADD command which copies the folder to /opt/flask-app. The last command is our WORKDIR command which all subsequent run commands are run from.

Navigate to our flask-app directory and lets build and run our new image.

```
cd flask app
sudo docker build -t sandrageisha/test2 .
sudo docker run -P --rm sandrageisha/test2
```

We get a timed out exception so appearantly these two containers can't connect to one another. So how do we solve this? **Networks.**

Well if we look at our flask app we see that it's trying to connect to elastic search on port 9200 so what if we just set the IP to 0.0.0.0:9200 we can do this on the host since we already exported and exposed the port but this will not work because docker runs on it's own network. The host can connect via 0.0.0.0:9200 but our container cannot since it's in essence its own machine with its own localhost.

So what ip should we connect to? Well lets have a look at networking in docker by running `sudo docker network ls `   We see there's three networks present: 

- Bridge which is responsible for running containers (every container is run in bridge)
- Host a network reserved for our host machine running docker
- None or no network at all

we can also see the details of our docker network by running `sudo docker network inspect bridge` We can see a network and an ip associated to this network. But this is not the way to go forward, actually we want to create our own network and make sure the two containers (flask and our search container) are run from within this network.  So let's create our network:

```
sudo docker network create food-net
sudo docker network ls
sudo docker rm es
sudo docker run -d --name es --net food-net -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:6.3.2
sudo docker network inspect food-net
```

We can see our es container is listed under the container section of this network and there's an ip associated with it. So now lets also run our flask app on the same network:

```
docker run -it --rm --net food-net sandrageisha/test2 bash
$ ls
$ python app.py
```

Great everything is up and running

## Docker dotnet

Well finally we have all the information we need to put our entire .Net application in docker. The first thing we have to do is script our configuration in a docker file. Lets make it all dynamic and make sure we don't have to do anything manually. The first thing we need is the github repository, well we could use the base image of git but for this example I'm going to use the base image of centos since that's what we've been using so far and we're sure all commands work on there. Lets pull in the image.

```
sudo docker pull centos:7
sudo docker images
```

if everything was fine centos:7 should be listed there. We can verify that our image works by running an interactive shell on it:

```
sudo docker run -it --name cent centos:7
$ ls 
$ exit
sudo docker rm cent
```

Jup that works just fine  allright so now lets build a dockerfile first we make a new folder:

```
mkdir $/Desktop/DotnetImage
cd $/Desktop/DotnetImage
touch Dockerfile
// Or any other text editor.
gedit Dockerfile
```

Okay let's make our image

```
# Specify the base image
FROM centos:7

# Put me as a maintainer
LABEL maintainer="michiel.schoofs@student.hogent.be"

# First we need a username and email we also provide a default value if none is given
ARG name=michiel-schoofs
ARG email=michiel.schoofs@student.hogent.be
# Sorry no default value here :)
ARG password

# The branch we're going to run
ARG branch=develop

# Get git installed and configured
RUN yum install -y git
RUN git config --global user.name ${name}
RUN git config --global user.email ${email}

# Pull in the repository
RUN git clone https://${name}:${password}@github.com/HoGent-Projecten3/projecten3-1920-backend-grasmaaier-team.git

# Set the repository as default location
WORKDIR /projecten3-1920-backend-grasmaaier-team/KolveniershofAPI

# Switch branch and pull in dependencies
RUN git checkout ${branch}
RUN git pull

# Install .Net 
RUN rpm -Uvh https://packages.microsoft.com/config/rhel/7/packages-microsoft-prod.rpm
RUN yum makecache
RUN yum install -y dotnet-sdk-2.1

# Do a bit of configuration
RUN dotnet user-secrets set "Tokens:Key" "HaveYouHearedThatUnicornKittiesAreTheBest"

# Expose the ports
EXPOSE 5000

# The original command to run
CMD ["dotnet","run","--urls","http://0.0.0.0:5000"]
```

We need to make sure that SQLServer can communicate with our .NET. Normally we could use stuff like service discovery but I think this document is long enough as it is. So I'll hardcode in the connection string. Well we also need a network so what we can do is think a bit over what ip ranges and subnet we want to allocate.  Well we need a base ip, lets use for example 172.12.0.0/16 as our subnet and 172.12.0.1 as our gateway. Pretty straightforward. The other disadvantage of using a hardcoded ip like this is that the order we assign our containers to the network is also important **First assign SQLServer**

Lets build our network:

```
sudo docker network create -d bridge --subnet 172.12.0.0/16 --gateway 172.12.0.1 dot-net
sudo docker network ls 
sudo docker network inspect dot-net
```



To build we run the command sudo docker build --build-arg password=**passwordOfDefaultUser** sandrageisha/dotnetapp .

Note that we use the no-cache argument since we don't want that the github code is cached. We want to pull in everything anew.

So now that we have our images first lets delete our sql server image because we need  it as part of our new network.

We can run the entire application by executing the following commands:

```
sudo docker rm SQLServer

sudo docker run -d --name SQLServer --net dot-net -e 'ACCEPT_EULA=Y' -e
'SA_PASSWORD=p@ssw0rd' -e 'MSSQL_PID=Developer' -p 1433:1433 microsoft/mssql-server-linux:2017-latest

sudo docker run -d --name DotNet -p 5000:5000 -p 5001:5001 --net dot-net sandrageisha/dotnetapp
```

Everything is working!!

Lets push the image and we're done

 `sudo docker push sandrageisha/dotnetapp`

## Angular dotnet

```
# Specify the base image
FROM centos:7

# Put me as a maintainer
LABEL maintainer="michiel.schoofs@student.hogent.be"

# First we need a username and email we also provide a default value if none is given
ARG name=michiel-schoofs
ARG email=michiel.schoofs@student.hogent.be
# Sorry no default value here :)
ARG password

# The branch we're going to run
ARG branch=develop

# Get git installed and configured
RUN yum install -y git
RUN git config --global user.name ${name}
RUN git config --global user.email ${email}

# Pull in the repository
RUN git clone https://${name}:${password}@github.com/HoGent-Projecten3/projecten3-1920-angular-grasmaaier-team.git

# Install npm and angular dependencies
RUN curl -sL https://rpm.nodesource.com/setup_10.x | bash -
RUN yum install -y nodejs
RUN npm install -g @angular/cli

# Set the working directory 
WORKDIR /projecten3-1920-angular-grasmaaier-team/

#Switch repo and pull
RUN git checkout ${branch}
RUN git pull

# Get the project dependencies
RUN npm install

# Change the proxy config file
RUN sed -i 's/"https:"/"http"/' proxy.conf.json
RUN sed -i 's/"localhost"/"172.12.0.3"/g' proxy.conf.json
RUN sed -i 's/5001/5000/g' proxy.conf.json

# Expose the port to the host system
EXPOSE 4200

# Start command
CMD ["ng","serve","--proxy-config","proxy.conf.json","--host","0.0.0.0"]
```

 sudo docker build --build-arg password=**password of github account** -t sandrageisha/angularapp .



sudo docker pull  sandrageisha/angularapp

sudo docker run -p 4200:4200 --net dot-net --name angapp sandrageisha/angularapp