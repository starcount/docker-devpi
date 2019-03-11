docker-devpi
============

This repository contains a Dockerfile for [devpi pypi server](http://doc.devpi.net/latest/).

Use this container to create a PyPi repository that stores python packages and libraries 
from which you can `pip install`.

# Setting up the repository

## Deploying

We are hosting this PyPi repository in an on-prem server, if you need to re-deploy speak to the 
Systems team to get the server details. There is a user setup called `devpi`. To deploy the 
Dockerfile run the `deploy.sh` script.


## Building 

Build the image using

```bash
sudo docker build -t devpi .
```

## Running

Start the server using

```bash
sudo docker run -d --name devpi \
    --publish 3141:3141 \
    --volume $DEVPI_DATA:/data \
    --env=DEVPI_PASSWORD=$DEVPI_PASSWORD \
    --restart always \
    devpi
```

You will need to set the ``DEVPI_PASSWORD`` and ``DEVPI_DATA`` environment variables.

# Using the repository 

## Uploading

You will need to ``pip install devpi-client`` locally to manually upload packages to the 
repository.

First login to devpi using

```bash
devpi use http://$DEVPI_HOST:3141
devpi login root --password=$DEVPI_PASSWORD
devpi use root/starcount
```

where ``DEVPI_HOST`` is the IP address of the server running the repository. 

Then from a directory of a project with a ``setup.py`` file run 

```bash
devpi upload 
```

This will build the package into a gztar formatted release file and upload it 
to the current index using ``setup.py`` under the hood.

You can also configure ``setup.py`` to automatically build & upload releases by
configuring your index server in a `$HOME/.pypirc` file:

```bash
# content of $HOME/.pypirc
[distutils]
index-servers = 
    starcount

[starcount]
repository: http://$DEVPI_HOST:3141/root/starcount/
username: root
password: $DEVPI_PASSWORD 
```
**NOTE: you will need to expand both DEVPI_HOST and DEVPI_PASSWORD here as plain text.** 

Then build your release using
```bash
python setup.py sdist upload -r starcount 
```

**TODO** Replace this with Twine!

## Installing

Once uploaded to the repository a package can be installed using pip both manually
and automatically.

### Using pip manually
To stick with usual convention using pip:
```bash
pip install package_name --index http://$DEVPI_HOST:3141/root/starcount/+simple/ --trusted-host $DEVPI_HOST
```

the `--trusted-host` flag is needed in order for pip to accept the non-https external repository.

### Using pip automatically
In order to avoid having to type the additional flags every time it is possible to store this 
information in your local `~/.pip/pip.conf` file:

```bash
# content of $HOME/.pip/pip.conf
[global]
extra-index-url = http://$DEVPI_HOST:3141/root/starcount/+simple

[install]
trusted-host = $DEVPI_HOST

```

### In a Dockerfile
To use the repository as the cache for your Dockerfile builds, add the following to your 
Dockerfile. The docker containers will try and use our PyPi repository before falling back
to normal servers.

```Dockerfile
# Install netcat for ip route
RUN apt-get update \
 && apt-get install -y netcat \
 && rm -rf /var/lib/apt/lists/*

 # Use Starcount's PyPi repository
RUN export HOST_IP=$(ip route| awk '/^default/ {print $3}') \
 && mkdir -p ~/.pip \
 && echo [global] >> ~/.pip/pip.conf \
 && echo extra-index-url = http://$HOST_IP:3141/root/starcount/+simple >> ~/.pip/pip.conf \
 && echo [install] >> ~/.pip/pip.conf \
 && echo trusted-host = $HOST_IP >> ~/.pip/pip.conf \
 && cat ~/.pip/pip.conf
```

# Extra bits

## Persistence

For devpi to preserve its state across container shutdown and startup you
should mount a volume at `/data`. The command above already includes this.

## Security

Devpi creates a user named root by default, its password should be set with
``DEVPI_PASSWORD`` environment variable. Please set it, otherwise attackers can
*execute arbitrary code* in your application by uploading modified packages.

For additional security the argument `--restrict-modify root` has been added so
only the root may create users and indexes.
