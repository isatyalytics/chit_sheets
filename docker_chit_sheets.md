# Multipass

Tool to run Ubuntu VM's easily with command-line interface.

List available instances

* `multipass list`

Create and launch a new instance using the latest LTS release

* `multipass launch --name my_instance`

Access the instance shell

* `multipass shell my_instance`

Mount a shared folder in the instance

* `multipass mount path/to/local/folder my_instance:path/to/instance/folder`

Unmount all mounted folders of instance

* `multipass umount my_instance`

Stop an instance

* `multipass stop my_instance`

Start a previously created instance

* `multipass start my_instance`

Get info on a specific instance

* `multipass info my_instance`

Delete an instance (send it to the recycle bin)

* `multipass delete my_instance`

Recover a deleted instance

* `multipas recover my_instance`

Permanently delete all deleted instances

* `multipass purge`

# Docker

## Terminology

* Container: environment that uns an applications that is not dependent on the OS. Kind of like a lightweight VM. Containers are ***stateless***; if you need to update the components inside, create another container instead.
* Image: template to create a container. Its components are defined by a `Dockerfile`.
* Volume: storage area detached from the container for maintaining state.
* Foreground/interactive vs background/detached: a detached container runs in the background whereas an interactive container will usually have a terminal of some sort for interacting with.

## Commands

List your local images

* `docker images`

Clean up images (many ways)
* `docker images -q -f dangling=true`
* `docker image rm`
* `docker image prune`

List your running containers
* `docker ps`

Run a Docker image inside a container

* `docker run -it --rm image_name:tag`
    * `-it` is a combination of `-i` (interactive mode) and `-t` (allocate a terminal).
    * `--rm` means that the container will be removed when exited.
    * You may find Docker images at the [Docker Hub](https://hub.docker.com/).
    * This command will use the entrypoint defined by the image. It won't necesarily open a terminal inside the container.

Run a Docker image inside a container and override the entrypoint
* `docker run -it --rm --entrypoint=bash image_name:version`
    * This will override the entrypoint of your image and open a bash terminal inside the container instead.

Run a Docker image inside a container and map a port in the container to a port in the host machine
* `docker run -it --rm -p 9696:9696 image_name:tag`

Create a `Dockerfile` with instructions to create a basic custom Docker image.

```Dockerfile
# set base image
FROM python:3.9

# set the working directory in the container
WORKDIR /app

# copy dependencies to the working directory
COPY requirements.txt .

# Install dependencies
RUN pip install -r requirements

# Copy code to the working directory
COPY . /app

# command to run on container start
CMD ["python", "./main.py"]
```

* Docker will process each line as a layer. Some layers are cached, so in order to speed up build time, first copy and run immutable objects and then take care of your code/modules, as shown in this example.
* Base images are useful because they save a lot of work and build time. Choose a lean base image and avoid unnecessary packages.
* Each container should only have one concern. Decouple applications into multiple containers.

Create a slightly more complex `Dockerfile` with pipenv dependencies and specific entrypoints.

```Dockerfile
# set base image
FROM python:3.9

# (pipenv) install pipenv
RUN pip install pipenv

# set the working directory in the container
WORKDIR /app

# (pipenv) copy dependencies to the working directory
COPY ["Pipfile", "Pipfile.lock", "./"]

# (pipenv) Install dependencies
# (pipenv) We don't need a virtualenv in Docker, so we can install dependencies to the system
RUN pipenv install --system --deploy

# Copy the model
COPY ["predict.py", "model.bin", "./"]

# Expose a port on the container
# Remember to map the port to a port in the host when running the container!
EXPOSE 9696

# Specify entrypoint
ENTRYPOINT ["gunicorn", "--bind=0.0.0.0:9696", "predict:app"]
```

* The `COPY` instruction has 2 forms, shown here. The second form (like for pipenv in this example) must be used if any paths may contain whitespaces. The last param is always the destination directoy, which may be `.` or `./` for copying to the directory specified by `WORKDIR`.

Build an image based on a Dockerfile

* `docker build -f Dockerfile -t my_image .`
    * The default Dockerfile that the command will look for is `$PATH/Dockerfile`. If your `Dockerfile` is in the same directory that you will run the command and you have not named it something else, `-f Dockerfile` can be removed from the command.
    * `my_image` will be the name of your image. You may optionally tag it like so: `my_image:my_tag`.

Stop a running container

* `docker stop container_id`

# Docker compose

Example `docker-compose.yaml` file.

```yaml
version: "3.9"
services:
  model-server:
    image: zoomcamp-10-model:v1
  gateway:
    image: zoomcamp-10-gateway:v2
    environment:
      - TF_SERVING_HOST=model-server:8500
    ports:
      - "9696:9696"
```

* `version` is required by `docker-compose``
* The app has 2 components: `model-server` and `gateway`
* Each component must have a Docker `image`.
* You may specify environment variables with `environment` and port mappings with `ports`
    * The dash (`-`) means that the entry is a list. In this example there are 2 lists with a single element each.

Run the app.

```sh
docker-compose up
```

Run the app in detached mode.

```sh
docker-compose up -d
```

Shut down the app

```sh
docker-compose down
```

# Kubernetes

## Kind

Create local cluster

```sh
kind create cluster
```

Delete local cluster

```sh
kind delete cluster
```

Load an image to the local cluster
```sh
kind load docker-image docker-image:tag
```

## eksctl

Create a default cluster on EKS.

```sh
eksctl create cluster
```

Create a cluster with a config YAML file

```sh
eksctl create cluster -f eks-config.yaml
```

Example `eks-config.yaml`

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: mlzoomcamp-eks
  region: eu-west-1

nodeGroups:
  - name: ng-m5-xlarge
    instanceType: m5.xlarge
    desiredCapacity: 1
```

* `metadata` contains both the `name` of the cluster as well as the AWS `region`.
* `nodeGroups` contains a list of node groups. In this example the list has a single entry.
    * `desiredCapacity` contains the amount of nodes inside the node group.
    * `instanceType` is the desired AWS EC2 instance type for the node group. All nodes will be of that instance type.

Delete a cluster

```sh
eksctl delete cluster -f eks-config.yaml
```

## kubectl

[kubectl command cheatsheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

Example `deployment.yaml`file

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <deployment-name>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: <app-name>
  template:
    metadata:
      labels:
        app: <app-name>
    spec:
      containers:
      - name: <my-container>
        image: my-component-image:some-tag
        resources:
          limits:
            memory: "128Mi"
            cpu: "100m"
        ports:
        - containerPort: 9696
        env:
          - name: TF_SERVING_HOST
            value: <service-name>.<namespace>.svc.cluster.local:8500
```

* `kind` must be `Deployment`
* `metadata.name` contains the name of the deployment
* `spec.replicas` states how many pods should be replicated in the deployment. This example file only states 1 replica.
* `spec.selector` defines how the deployment finds which pods to manage. `spec.selector.matchLabels` is a rule that will match a label in the pod template (the label in this case is `app:<app-name>`)
* `spec.template` contains the _blueprint_ for the pods:
    * `metadata` in this example contains the labels we use for the pods so that the deployment can find and manage them.
    * `..spec.containers` contains a plethora of info:
        * `name` is the name of the containers inside the pod.
        * `image` is the Docker image to be used by the containers.
        * `resources` states the physical resource limits
            * For CPU, _`100m`_ means _100 milliCPUs_, or 10% of the available CPU computing time.
        * `ports` contains the ports to use by the containers.
        * `env` contains names and values for nvironment variables, useful for apps to be able to find other containers by their internal cluster URL.
            * When defining a service, Kubernetes publishes a DNS entry inside the Cluster to make it possible for pods to find other pods. These DNS entries follow the `<service-name>.<namespace>.svc.cluster.local:<port>` format.
            * The default namespace is `default`.

Example `service.yaml` file.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: <service-name>
spec:
  type: LoadBalancer
  selector:
    app: <app-name>
  ports:
  - port: 80
    targetPort: 9696
```

* `kind` must be `Service`
* `metadata.name` contains the name of the service
* `spec.type` specifies the type of Service.
    * Internal services are of type `ClusterIP`. This is the default service type if this field is not stated in the file.
    * External services are of type `LoadBalancer` and are assigned an external IP.
* `spec.selector` contains the label to find the deployment to which it belongs to.
* `spec.ports` contains both the port of the service (`port`) as well as the port of the deployment (`targetPort`).
