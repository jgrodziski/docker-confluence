I describe the confluence/postgresql installation with [docker](https://www.docker.io/). I decided to go for a standalone container (java + confluence + postgresql), with in mind to run the container like "start confluence" and with the local data directory as the only parameter.

I start from the [Phusion ubuntu base image](https://github.com/phusion/baseimage-docker) for its docker-friendliness and merge with the [Paintedfox postgresql](https://index.docker.io/u/paintedfox/postgresql/). You can find all the necessary files in the [github repo docker-confluence](https://github.com/jgrodziski/docker-confluence).

# Nginx
```sudo apt-get install nginx```

# Docker
See [Docker Installation](https://www.docker.io/gettingstarted/#h_installation)

# the Dockerfile
3 steps: 

1. install [Java](http://java.com/en/)
2. install [Confluence](https://www.atlassian.com/software/confluence)
3. install [Postgresql](http://www.postgresql.org/)

The post-installation process of Confluence will be done at the first run of the container.

Here is the Dockerfile with comments explaining its actions when building

```
# Pull base image from phusion as it comes with useful presets
FROM phusion/baseimage:0.9.9

# Set correct environment variables.
ENV HOME /root

# Regenerate SSH host keys. baseimage-docker does not contain any, so you
# have to do that yourself. You may also comment out this instruction; the
# init system will auto-generate one during boot.
RUN /etc/my_init.d/00_regen_ssh_host_keys.sh

# Use baseimage-docker's init system.
CMD ["/sbin/my_init"]

# Install Java
RUN apt-get update
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y --force-yes software-properties-common python-software-properties
RUN add-apt-repository -y ppa:webupd8team/java
RUN apt-get update
RUN echo debconf shared/accepted-oracle-license-v1-1 select true | debconf-set-selections
RUN echo debconf shared/accepted-oracle-license-v1-1 seen true | debconf-set-selections
RUN apt-get install -y oracle-java7-installer

# Install confluence
# 1. download the archive
RUN wget -O /tmp/confluence.bin http://www.atlassian.com/software/confluence/downloads/binary/atlassian-confluence-5.4.4-x64.bin
# 2. make it executable
RUN chmod u+x /tmp/confluence.bin
# 3. set the response.varfile of the installer for offline and quiet installation
ADD response.varfile /tmp/response.varfile
# 4. run the installer 
RUN /tmp/confluence.bin -q -varfile /tmp/response.varfile
EXPOSE 8090

# Ensure UTF-8
RUN apt-get update
RUN locale-gen en_US.UTF-8
ENV LANG       en_US.UTF-8
ENV LC_ALL     en_US.UTF-8

# Install the latest postgresql
RUN echo "deb http://archive.ubuntu.com/ubuntu precise universe" > /etc/apt/sources.list.d/pgdg.list &&\
    echo "deb http://apt.postgresql.org/pub/repos/apt/ precise-pgdg main" > /etc/apt/sources.list.d/pgdg.list && \
    apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --force-yes postgresql-9.3 postgresql-client-9.3 postgresql-contrib-9.3 && \
    /etc/init.d/postgresql stop

# Install useful other tools.
RUN DEBIAN_FRONTEND=noninteractive && \
    apt-get install -y pwgen inotify-tools

# Decouple our data from our container.
VOLUME ["/var/data/postgresql"]

# Configure the database to use our data dir
RUN sed -i -e"s/data_directory =.*$/data_directory = '\/var\/data\/postgresql'/" /etc/postgresql/9.3/main/postgresql.conf
# Allow connections from anywhere.
RUN sed -i -e"s/^#listen_addresses =.*$/listen_addresses = '*'/" /etc/postgresql/9.3/main/postgresql.conf
RUN echo "host    all    all    0.0.0.0/0    md5" >> /etc/postgresql/9.3/main/pg_hba.conf

EXPOSE 5432
ADD scripts /scripts
RUN chmod +x /scripts/start*.sh
RUN touch /firstrun

# Clean up APT when done.
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
RUN cp -f /scripts/start.sh /etc/my_init.d

# SSH stuffs
# add all the authorisation key from the machine where the build is done
RUN mkdir -p /root/.ssh
ADD my_keys /tmp/my_keys
RUN cat /tmp/my_keys >> /root/.ssh/authorized_keys && rm -f /tmp/my_keys
RUN chmod 700 /root/.ssh
RUN chmod 600 /root/.ssh/authorized_keys
```

execute this Dockerfile:
```
docker build -t confluence-img .
```

now that the image is built, we can run the container for the first time:
```
docker run -d --name="confluence-container" \
           -p 8090:8090 -p 5432:5432 \
           -v /var/local/data/confluence-postgresql:/var/data/postgresql \
           -e USER="confluenceuser" \
           -e DB="confluencedb" \
           -e PASS="yourpassword" \
           confluence-img
```

- -d means the container is detached
- -p does the port mapping from the container to the host
- -v does the volume mapping from the container to the host
- -e are the variables that drives the postgresql inits

The container should now run, verify with:
```
docker ps
```

you should see something like 
```
CONTAINER ID        IMAGE                   COMMAND             CREATED             STATUS              PORTS                                            NAMES
178b56498780        confluence-img:latest   /sbin/my_init       2 hours ago         Up 2 hours          0.0.0.0:5432->5432/tcp, 0.0.0.0:8090->8090/tcp   confluence-container
```

# configure Nginx
Now, with your container started and its port exposed to the host, we are able to reverse proxy confluence with nginx.
Add a www.mydomain.com file in /etc/nginx/sites-available with:
```
server {
    server_name confluence.mydomain.com;
    location / {
        proxy_pass http://localhost:8090;
    }
}
```

# configure Confluence
The first time you'll visit the confluence installation URL, it will ask you to fill the licence data, setup the database, etc.:


