# Creating a cluster with rancher

## Why Rancher ?

Rancher is a complete software stack for teams adopting containers. It addresses the operational and security challenges of managing multiple Kubernetes clusters across any infrastructure, while providing DevOps teams with integrated tools for running containerized workloads.

## Get started with Rancher in 2 Easy Steps.

1. Prepare a Linux Host: Prepare a Linux host with 64-bit Ubuntu 16.04 or 18.04 (or another [supported Linux distribution
](https://rancher.com/support-maintenance-terms/all-supported-versions/) ), and at least 4GB of memory. Install a [supported version of Docker](https://rancher.com/support-maintenance-terms/all-supported-versions/) on the host.
2. To install and run Rancher, execute the following Docker command on your host:
   ```shell
    $ sudo docker run -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
   ```
   To access the Rancher server UI, open a browser and go to the hostname or address where the container was installed. You will be guided through setting up your first cluster.

That's it. Rancher should be up and running for now.

## Configure your cluster

Now let's move on configuring your kubernetes cluster with rancher. We will configure a cluster on bare metal without any cloud provider.

Rancher should be available at https://localhost. Go there and configure your password for rancher dashboard.

1. Click on `Add cluster`
2. From the available cluster types select "From existing nodes (Custom)"
3. Fill in the cluster name. You may also configure some needed options in the above dashboard.
4. Click `Next`.
5. Now you have to configure your nodes. We will make one master node and one worker node. For this select `etcd` and `Control Plane` checkboxes, this will be our master node. A command line is generated for the selected options copy it.
6. Execute this command on the machine where rancher is hosted or other machine of your choice. 

After this cluster will be created and a master node will be added to the cluster. Give it some time in order to configure and run the newly created cluster and node. After everything has finished you should be able to see your cluster up and running in the list of clusters.

Currently your cluster has only one node. For scalability purposes let's set up 2 more worker nodes.

1. Select your newly created cluster and click on the options button (3 vertical dots) and click Edit.
2. Scroll down to the "Customize Node Run Comman" menu.
3. Check only the "Worker" checkbox.
4. The command is now generated. Copy it and run it on the desired node machines.
5. The nodes should automatically configure and run.

Congratulations, you now have a kubernetes cluster with 2 worker nodes all administered by rancher. 

This cluster can now be used to deploy container based apps in a staging environment. A devops pipeline can be set using this cluster for deployment.

For more information and customization please refer to the [Rancher docs.](https://rancher.com/docs/rancher/v2.x/en/)


# Deploying a docker registry

Docker images are stored in a registry. A registry is a storage and content delivery system, holding named Docker images, available in different tagged versions. Usually in a development environment custom private registries are created where devlopers push and pull images. It's private because some apps are not intended for public access. Here we will learn how to setup and run a private docker registry that can be then used with a kubernetes cluster for a complete development/deployment cycle.

Docker registry is running on a container in docker (oh the irony). For this make sure you have docker installed.

## Understanding image naming

Image names as used in typical docker commands reflect their origin:

 * `docker pull ubuntu` instructs docker to pull an image named `ubuntu` from the official Docker Hub. This is simply a shortcut for the longer `docker pull docker.io/library/ubuntu` command
 * `docker pull myregistrydomain:port/foo/bar` instructs docker to contact the registry located at `myregistrydomain:port` to find the image `foo/bar`

You can find out more about the various Docker commands dealing with images in
the [official Docker engine
documentation](/engine/reference/commandline/cli.md).

## Use cases

Running your own Registry is a great solution to integrate with and complement
your CI/CD system. In a typical workflow, a commit to your source revision
control system would trigger a build on your CI system, which would then push a
new image to your Registry if the build is successful. A notification from the
Registry would then trigger a deployment on a staging environment, or notify
other systems that a new image is available.

It's also an essential component if you want to quickly deploy a new image over
a large cluster of machines.

Finally, it's the best way to distribute images inside an isolated network.

## Run a local registry

Use a command like the following to start the registry container:

```bash
$ docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

The registry is now ready to use.

## Using the local registry

Now that your local registry is up and running let's try to push an image. We will copy an image from docker hub and push it to our local registry in this example.

1. Pull the `nginx:latest` image from Docker Hub.

    ```bash
    $ docker pull nginx:latest
    ```
2. Tag the image as `localhost:5000/my-nginx`. The first part of the tag is the hostname and port. When pushing docker interprets this part as the location of the registry to push to. In our case the registry is running locally so the host is `localhost` and the port is `5000` which was published above.

    ```bash
    docker tag nginx:latest localhost:5000/my-nginx
    ```

3. Push the image to the local registry running at `localhost:5000`:

    ```bash
    $ docker push localhost:5000/my-nginx
    ```
4. Remove the locally cashed `nginx:latest` and `localhost:5000/my-nginx` images so that you can test pulling from your registry. This does not remove `localhost:5000/my-nginx` from your registry.

    ```bash
    $ docker image remove nginx:latest
    $ docker image remove localhost:5000/my-nginx
    ```
5. Pull the `localhost:5000/my-nginx` from your local registry

    ```bash
    $ docker pull localhost:5000/my-nginx
    ```
The image should be pulled successfuly from the registry now.

Great. The registry works, but locally which means it will not be accessible remotely. We need to make this registry remotely accessible and configure an authentication with user name and password.

## Run an external-accessible registry

In order to make your registry accessible to external hosts, you must first secure it using TLS.

### Get a certificate

This example assumes that:

* Your registry URL is `https://myregistry.domain.com`
* Your DNS, routing and firewall settings allow access to the registry's host on port 443.
* You already obtained a certificate from a certificate authority (CA)

1. Create a certs directory
    
    ```bash
    $ mkdir -p certs
    ```
2. Stop the registry if it's currenlty running.

    ```bash
    $ docker container stop registry
    ```

3. Restart the registry, directing it to use the TLS certificate. This command bind-mounts the `certs/` directory into the container at `/certs` and sets environment variables that tell the container where to find the domain.crt and domain.key file. The registry runs on port 443, the default HTTPS port.

    ```bash
    $ docker run -d --restart=always --name registry -v "$(pwd)"/certs:/certs -e REGISTRY_HTTP_ADDR=0.0.0.0:443 -e REGISTR_HTTP_TLS_CERTIFICATE=/certs/domain.crt -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key -p 443:443 registry:2
    ```
4. Docker clients can now pull and push from your registry using provided external address. Here is an example

    ```bash
    $ docker pull nginx:latest
    $ docker tag nginx:latest myregistry.domain.com/my-nginx
    $ docker push myregistry.domain.com/my-nginx
    $ docker pull myregistry.domain.com/my-nginx
    ```

## Registry authentication

The simples way to achieve access restriction is through basic authentication. This example uses native basic authentication using htpasswd to store the secrets.

1. Create a password file with one entry for the user `testuser` with the password `testpassword`:

    ```bash
    $ mkdir auth
    $ docker run --entrypoint htpasswd registry:2 -Bbn testuser testpassword > auth/passwd
    ```
2. Stop the registry if currently running

    ```bash
    $ docker container stop registry
    ```
3. Start the registry with basic authentication

    ```bash
    $ docker run -p 443:443 --restart=always --name registry -v "$(pwd)"/auth:/auth -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd -v "$(pwd)"/certs:/certs -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key registry:2
    ```
4. Now you if you try to push or to pull images from this registry the command will fail.
5. Log in to the registry

    ```bash
    $ docker login myregistry.domain.com
    ```

    When asked provide the username and password you've set in step one.

    Now you should be able to push or to pull from the registry.

## Deployment using docker compose

An easier way to run and configure a docker registry would be using docker compose. All the configurations above can now be writtne in a `docker-compose.yml` file as shown below:

```yaml
registry:
  restart: always
  image: registry:2
  ports:
    - 443:443
  environment:
    REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
    REGISTRY_HTTP_TLS_KEY: /certs/domain.key
    REGISTRY_AUTH: htpasswd
    REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
    REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
  volumes:
    - /path/data:/var/lib/registry
    - /path/certs:/certs
    - /path/auth:/auth
```

Make sure to replace /path with the directory containing the `certs/` and `auth/` directories.

Start and configure your registry by simply running the following command in the directory containing the `docker-compose.yml` file:

```bash
$ docker-compose up -d
```

This is it, the registry is now configured to use https with provided certificate and key have authentication and running currently. This registry can now be used externally to push or pull images securely.