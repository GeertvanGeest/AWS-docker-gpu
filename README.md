# AWS-docker

Scripts to employ multiple docker containers simultaneously for teaching. 

This is a fork of [github.com/GeertvanGeest/AWS-docker](github.com/GeertvanGeest/AWS-docker) which enables running containers with a gpu. 

## Preparation

Use an AWS deep learning AMI, e.g. [AWS Deep Learning Base AMI](https://aws.amazon.com/marketplace/pp/prodview-dxk3xpeg6znhm?sr=0-1&ref_=beagle&applicationId=AWSMPContessa). These AMI directly support gpu integration.

Clone this repository to the AWS instance:

```sh
git clone https://github.com/GeertvanGeest/AWS-docker-gpu.git
```

## Generate credentials

Note that each container gets one GPU assigned. So the number of users should not be more than the number of available GPU!

You can generate credentials from a tab-delimited list of users, with four columns: first name, last name, e-mail, instance IP. Here's an example:

```
Jan	de Wandelaar	jan.wandel@somedomain.com	18.192.64.150
Piet	Kopstoot	p.kopstoot@anotherdomain.ch	18.192.64.150
Joop	Zoetemelk	joop.zoet@lekkerfietsen.nl	18.192.64.150
```

Run the script `generate_credentials.sh` like this (use `-l` to specify the user list):

```sh
./generate_credentials \
-l examples/user_list_credentials.txt \
-o ./credentials
-p 9001
```

The option `-o` specifies an output directory in which the following files are created:

* `input_docker_start.txt`: A file that serves as input to deploy the docker containers on the server
* `user_info.txt`: A file with user names, passwords and links that can be used to communicate credentials to the participants

The option `-p` is used to generate an individual port for each user. Ports will be assigned in an increasing number from `-p` to each user. So, in the example above, the first user gets port 9001, the second 9002, the third 9003, etc. **Be aware that port 9000 and 10000 are reserved for the admin containers!**

## Deploying containers

Use the script `./generate_credentials.sh` to generate a list of ports, usernames and passwords (or generate it manually otherwise). It should look like this:

```
9001	jdewandelaar	OZDRqwMRmkjKzM48v+I=
9002	pkopstoot	YTnSh6SmhsVUe+aC2HY=
9003	jzoetemelk	LadwVbiYY4rH0S5TjeI=
```

Once deployed, the jupyter notebook or rstudio server will be available through `[HOST IP]:[PORT]`. If you want to have both rstudio server and jupyter notebook running on the same instance, you can generate two tab-delimited files (on for rstudio and one for jupyter) and give them the same passwords for convenience. **Note that each container uses a single port, so the files should contain different ports!**

### Deploy containers based on tensorflow jupyter notebook

Prepare an image that you want to use for the course. This image should be based on a tensorflow jupyter notebook container, e.g. [tensorflow/tensorflow:latest-gpu-jupyter](https://hub.docker.com/r/tensorflow/tensorflow/tags?page=1&ordering=last_updated), and should be available from `dockerhub`. More info [here](https://www.tensorflow.org/install/docker).

Run the script `run_jupyter_notebooks`:

```sh
run_jupyter_notebooks \
-i jupyter/base-notebook \
-u examples/credentials_jupyter/input_docker_start.txt \
-p test1234
```

Here, `-i` is the image tag, `-u` is the user list as generated by `./generate_credentials.sh`, and `-p` is the password for the admin container.
No username is required to log on to a jupyter notebook.

To access the admin container, go to `[HOST IP]:10000`.

Note that each container gets one GPU assigned. So the number of users should not be more than the number of available GPU!

### Deploy containers based on Rstudio server

Prepare an image that you want to use for the course. This image should be based on a rocker ml image, e.g. [rocker/ml](https://hub.docker.com/r/rocker/ml), and should be available from `dockerhub`.

Run the script `run_rstudio_server`:

```sh
run_rstudio_server \
-i rocker/rstudio \
-u examples/credentials_rstudio/input_docker_start.txt \
-p test1234
```

See above for the meaning of the options.

The username to log on to rstudio server is `rstudio`.

To access the admin container, go to `[HOST IP]:9000`

## Restricting resource usage

To prevent overcommitment of the server, it can be convenient to restrict resource usage per participant. You can do that with the options `-c` and `-m`, which are passed to the arguments `--cpus` and `--memory` of `docker run`. Use it like this:

```sh
run_rstudio_server \
-i rocker/rstudio \
-u examples/credentials_rstudio/input_docker_start.txt \
-p test1234 \
-c 2 \
-m 4g
```

Resulting in a hard limit of 2 cpu and 4 Gb of memory for each user. By default these are 2 cpu and 16 Gb of memory. These restrictions are not applied to the admin container.

## Container & volume infrastructure

Below you can find an example of the container infrastructure. Blue squares are containers, yellow are volumes. Arrows indicate accessibility. The data volume is meant to harbour read-only data (e.g. raw data). The group volume is meant as a shared directory, where everybody can read and write.

![container infrastructure](infrastructure.png)

## How to use admin privileges

The admin container (i.e. with sudo rights) is available from port 10000 for the jupyter containers and 9000 for the rstudio containers. The regular users at the ports specified in the tab-delimited text file.

You can check out a user volume with `mount_user_volume.sh`:

```sh
./mount_user_volume.sh user01
```

This will create an ubuntu container directly accessing the home directory of user01. As an alternative to this ubuntu container, you can mount the user volume to any other container.

## Stopping services

You can stop all services (containers and volumes) with the script `stop_services.sh`.
