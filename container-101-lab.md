
# Containers 101 - Labs

In this lab, you will be learning some basics on containers.

## Objectives
- Create Lab VM
- Installing container engine
- Starting container instance
- Running commands in container
- Building container image
- Creating container registry
- Publishing container image
- Pulling and running container image
- Running container serverlessly

## Duration
60 min

## Helper
- [Docker CheatSheet](https://www.docker.com/sites/default/files/Docker_CheatSheet_08.09.2016_0.pdf)

## 1 Install Azure CLI

Follow the instructions of following link to install Azure CLI
> https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest

## 2 Create Lab host
Login to Azure with Azure CLI.

    $ az login

> Optional: If you have multiple Azure subscriptions, please use following commands to switch to your desired subscription.

    $ az account list -o table
    $ az account set -s <subscription id>

Create a resource group in region `eastasia` with name `containers-101`.

    $ az group create -l eastasia -n containers-101 

Create a virtual machine with Azure CLI.

    $ az vm create -g containers-101 -n container-lab \
      --image OpenLogic:CentOS:7.5:latest \
      --size Standard_DS2_v2 \
      --admin-username sysadm \
      --generate-ssh-keys

If everything goes well, you should see following output.

    ResourceGroup    PowerState    PublicIpAddress    Fqdns    PrivateIpAddress    MacAddress         Location    Zones
    ---------------  ------------  -----------------  -------  ------------------  -----------------  ----------  -------
    containers-101    VM running    13.70.1.93                  10.0.0.4            00-0D-3A-80-1E-7F  eastasia

Run command `az vm show`, you will see the summery information of the newly created VM `container-lab`. Make a note of the public IP address of the virtual machine.
​    
    $ az vm show -g containers-101 -n container-lab -o table
    Name           ResourceGroup    PowerState    PublicIps    Fqdns    Location    Zones
    -------------  ---------------  ------------  -----------  -------  ----------  -------
    container-lab  containers-101    VM running    13.70.1.93            eastasia

## 3 Installing container engine

After the lab VM is up and running, connect to the VM via SSH.

> Type `yes` to proceed the connection, if you are asked for confirmation.

    $ ssh sysadm@13.70.1.93

For lab convinence, switch to a root shell.

    $ sudo -i 

Install container engine Docker via yum.

    # yum install docker -y

After installing Docker, start the Docker service. And make it starts up after system starts.

    # systemctl start docker
    # systemctl enable docker

Run command `docker info`, you should see some detail information of the Docker daemon.

    # docker info

## 4 Starting container instance

Now let's try to run our first container instance, a Nginx web server. `nginx` is the name of the target container image. You can find more detail of the image in Docker Hub, which is a public container registry.
> Nginx image homepage: https://hub.docker.com/_/nginx

    # docker run -d --name nginx nginx

You will see some output similar to below example. Docker will detect there is no Nginx image locally, it will download the required image from remote DockerHub container registry.

    Unable to find image 'nginx:latest' locally
    Trying to pull repository docker.io/library/nginx ...
    latest: Pulling from docker.io/library/nginx
    27833a3ba0a5: Pull complete
    ea005e36e544: Pull complete
    d172c7f0578d: Pull complete
    Digest: sha256:e71b1bf4281f25533cf15e6e5f9be4dac74d2328152edf7ecde23abc54e16c1c
    Status: Downloaded newer image for docker.io/nginx:latest
    e01a310e83e94522f043d0271caa6e50296523f50711ae4e83107bbb77a9cf33

Use command `docker ps`, you will see there is a background running container instance, which is created by command `docker run` previously.

    # docker ps
    CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
    6d870a99065e        nginx               "nginx -g 'daemon ..."   24 seconds ago      Up 23 seconds       80/tcp              nginx

Note down the Container ID of the container instance. Run command `docker inspect` to query the IP address of the container isntance. By default, for connectivity, each container will be allocated an IP address.

    # docker inspect nginx |grep IPAddress
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.2",
                    "IPAddress": "172.17.0.2",

You can access the Nginx server by accessing the IP address of the container instance.

    # curl 172.17.0.2
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    ......

## 5 Running commands in container

To view the logs of the Nginx container, run command `docker logs`. You will see the access log of the Nginx server.

    # docker logs nginx
    172.17.0.1 - - [01/May/2019:01:50:16 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"

You can "get inside" a container, and execute commands in the Nginx container.

    # docker exec -it nginx bash

Verify the hostname inside the container.

    root@11972e69704b:/# hostname
    11972e69704b 

Verify the system information. It's different from the host.

    root@11972e69704b:/# uname -a
    Linux 11972e69704b 3.10.0-862.11.6.el7.x86_64 #1 SMP Tue Aug 14 21:49:04 UTC 2018 x86_64 GNU/Linux

Exit the container shell environment.

    root@11972e69704b:/# exit

## 6 Building container image

To start a container instance, you first need to have a container image. You can use an image built by others, or build a image on your own to satisfy you own needs.

First, create a working directory.

    # mkdir /opt/my-image
    # cd /opt/my-image/

Install Git. Git is a source code management tool. 

    # yum install git -y

Git clone to donwload a HTML5 application to the working directory. It's a simple HTML 5 game.

    # git clone https://github.com/nichochen/open-source-monster-fight.git app

Create Dockerfile with following contents. The Dockerfile is very simple, it uses the nginx image as base image, and copys the HTML5 app into foler `/usr/share/nginx/html`, which is the default root directory of Nginx. When the Nginx container is spinned up, Nginx will serve the contents of the HTML5 app.

    cat <<EOF >Dockerfile
    FROM nginx
    
    COPY app /usr/share/nginx/html
    EOF

To build the container image, we need to run a Docker build. Run command `docker build` with option `-t` to specify the name of the image. Here, we name the image `my-nginx:1.0`. Container image support tags for versioning, `1.0` is the tag of the image. 

    # docker build -t my-nginx:1.0 .
    Sending build context to Docker daemon 2.377 MB
    Step 1/2 : FROM nginx
    ---> 27a188018e18
    Step 2/2 : COPY app /usr/share/nginx/html
    ---> ee2394796912
    Removing intermediate container 4108bf287c4d
    Successfully built ee2394796912

After the build is successfully completed, run command `docker images` to inspect the list of container images on the lab host. You might see there are 2 images, the Nginx image and the newly created image `my-image:1.0`.

    # docker images
    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
    my-nginx            1.0                 ee2394796912        34 seconds ago      112 MB
    docker.io/nginx     latest              27a188018e18        12 days ago         109 MB

Run the newly created image `my-image:1.0`. This time, we pass option `-p 80:80` to command `docker run`, this tell Docker to map the host port `80` to container port `80`, this lets users can access the HTML5 app via the lab host ip and port 80. 

    # docker run -d -p 80:80 --name my-nginx my-nginx:1.0
    232f9ec4954e2f4ed9d02d3fc7bcf719ce37d1400eddc85d04bb43ce7f930d02

On the lab host, curl address `127.0.0.1:80`, you will see the output of some HTML and Javascript source code. Terminal is not user friendly for displaying web page, you may want to view the page with a GUI browser. Next, let's do it.

    # curl 127.0.0.1:80

Back to your `local host`, run `az vm open-port` to open port `80`, this allow traffic to port `80` from the internet.

    $ az vm open-port -g containers-101 -n container-lab --port 80

Query the `public IP address` of vm `container-lab`. Access the app with the public IP address of vm `container-lab` in your cell phone browser. Then you can play the game!

    $ az vm show -d  -g containers-101 -n container-lab
    Name           ResourceGroup    PowerState    PublicIps    Fqdns    Location    Zones
    -------------  ---------------  ------------  -----------  -------  ----------  -------
    container-lab  containers-101    VM running    13.70.1.93            eastasia

Run `docker ps`, you will see there are 2 container instances, they are runing independently, listening on 80 on their own IP address respectively. The container my-nginx, is listening on the host port 80 as well.

    # docker ps
    CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                NAMES
    52ad88a5c79c        my-nginx:1.0        "nginx -g 'daemon ..."   8 seconds ago       Up 7 seconds        0.0.0.0:80->80/tcp   my-nginx
    6d870a99065e        nginx               "nginx -g 'daemon ..."   5 minutes ago       Up 5 minutes        80/tcp               nginx

Inspect the Docker bridge network, you will see the detail setting for each containers.

    $ docker network inspect bridge
    ......
     "Containers": {
            "52ad88a5c79cf72bd307be5d69ae6f7dae83f88b3ead324f775ce057a44fb711": {
                "Name": "my-nginx",
                "EndpointID": "0c985c81ec47c04db85658be14ae970cf2fe4f333c0cfd0d6028bed129209d1f",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            },
            "6d870a99065e40014d14c587c4a2a47eda89a2170ffa7c1b196cb57ca5e35727": {
                "Name": "nginx",
                "EndpointID": "06e3815080ca069afd7fd51926fa65213e03b8f355a5dfee8176259e01366149",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
    ......

## 7 Creating container registry

After verifying the container image is working fine, now you can publish this container image to a remote container registry, this will help you distribute your container image to different envritonments.

Back to your local host, run following command to create a container registry with Azure Container Registry (ACR) service.

> To avoid name confliction, you can use env variable ${RANDOM} to generate a unique name for your registry.

    $ az acr create -g containers-101 -n labregistry${RANDOM} --sku Standard --admin-enabled

After the creation, you will see following output. Make a note for the name and URL of the registry. For example, `labregistry8197`, in the sample output.

        NAME             RESOURCE GROUP    LOCATION    SKU       LOGIN SERVER                CREATION DATE         ADMIN ENABLED
    ---------------  ----------------  ----------  --------  --------------------------  --------------------  ---------------
    labregistry8197  containers-101     eastasia    Standard  labregistry8197.azurecr.io  2019-04-30T03:14:09Z  True

ACR secured by Azure AD by default. You can also access the registry with user name and password. Use command `az acr credential show` to get login information.

    $ az acr  credential show -g containers-101 -n labregistry8197
    USERNAME         PASSWORD                          PASSWORD2
    ---------------  --------------------------------  --------------------------------
    labregistry8197  HF90cJGZ5QUIvARm+0IwtHmQG56ljpyE  1zyRnz5WVCT83uZdSFHsrtsO/DF5AIEw

## 8 Publishing container image

Nn your `container lab host`, use command `docker login` to login to the registry, with the registry URL, user name and password.
> Remember to replace the registry URL with your actual registry URL.

    # docker login labregistry8197.azurecr.io
    Username: labregistry8197
    Password:
    Login Succeeded

Rename the image, makes it pointing to the newly created registry. Container images keep their registry url as part of its names.

    # docker tag my-nginx:1.0 labregistry8197.azurecr.io/my-nginx:1.0

After tagging the image, you will find the image in local images list. The newly tagged image and the image `my-nginx:1.0` have identical Image ID.

    # docker images
    REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
    labregistry8197.azurecr.io/my-nginx   1.0                 ee2394796912        11 hours ago        112 MB
    my-nginx                              1.0                 ee2394796912        11 hours ago        112 MB
    docker.io/nginx                       latest              27a188018e18        13 days ago         109 MB

Publish the image to the remote container registry.

    # docker push  labregistry8197.azurecr.io/my-nginx:1.0
    The push refers to a repository [labregistry8197.azurecr.io/my-nginx]
    dc9ac8cbca4d: Pushed
    fc4c9f8e7dac: Pushed
    912ed487215b: Pushed
    5dacd731af1b: Pushed
    1.0: digest: sha256:1486da4e7bbf0ba563a062fc6b32dd4cca7a5ba77e97f76aee74ddee44a92f42 size: 1159

## 9 Pulling and running container image

To verify the published image works, you can try downloading and running the image. To make it more observable, let's clean up related local resources. First, stop and remove all container instances.

    # docker stop $(docker ps -q)
    # docker rm $(docker ps -aq)

Delete all local images.

    # docker rmi $(docker images -q)

Verify all images are gone.

    # docker images

Pull the image from the remote registry. If everything works fine, you should see the image can be pulled successfully.

    # docker pull labregistry8197.azurecr.io/my-nginx:1.0
    Trying to pull repository labregistry8197.azurecr.io/my-nginx ...
    1.0: Pulling from labregistry8197.azurecr.io/my-nginx
    27833a3ba0a5: Pull complete
    ea005e36e544: Pull complete
    d172c7f0578d: Pull complete
    a2c750b4e34e: Pull complete
    Digest: sha256:1486da4e7bbf0ba563a062fc6b32dd4cca7a5ba77e97f76aee74ddee44a92f42
    Status: Downloaded newer image for labregistry8197.azurecr.io/my-nginx:1.0

Run the docker image and test it with curl. You should see some HTML and Javascript source code as output.

    # docker run -d -p 80:80 labregistry8197.azurecr.io/my-nginx:1.0
    # curl 127.0.0.1:80

## 10 Running container serverlessly

You just created a VM, setup a container runtime, and run your containers on it. You can also run a container image with a serverless container service, Azure Container Instance (ACI). With ACI, you no need to manually setup the docker runtime. ACI will take care the underlying infrastructure to run and scale your container application. 

> Tip: please replace the value for `--registry-username` and `--registry-password`.

    # az container create \
      --resource-group containers-101 \
      --name my-nginx \
      --image labregistry8197.azurecr.io/my-nginx:1.0 \
      --ports 80 \
      --dns-name-label my-nginx-${RANDOM} \
      --location eastus \
      --registry-username labregistry8197 \
      --registry-password HF90cJGZ5QUIvARm+0IwtHmQG56ljpyE

From following sample output you can see the command returns the summary of the deployed container instance. 

    Name      ResourceGroup    Status    Image                                    IP:ports           Network    CPU/Memory       OsType    Location
    --------  ---------------  --------  ---------------------------------------  -----------------  ---------  ---------------  --------  ----------
    my-nginx  containers-101    Running   labregistry8197.azurecr.io/my-nginx:1.0  52.191.218.167:80  Public     1.0 core/1.5 gb  Linux     eastus

With the IP:PORTS in the summary above, you can access the web content served by the Nginx container.

    # curl 52.191.218.167:80

## 11 Clean up

Delete the Resource Group, all the resources in it will be deleted as well.

    # az group delete -n containers-101
    Are you sure you want to perform this operation? (y/n): y