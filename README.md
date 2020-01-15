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