# Makisu
--------

A docker image build tool that is more flexible, faster at scale. This makes it easy to build
lots of docker images directly from a containerized environment such as Kubernetes. Specifically, Makisu:
* Uses a distributed layer cache to improve performance across a build cluster.
* Provides control over generated layers with keyword #!COMMIT, reducing number of layers in images.
* Requires no elevated privileges, making the build process portable.
* Is Docker compatible. Our dockerfile parser is opinionated in some scenarios, more details can be found [here](lib/parser/dockerfile/README.md).

## Makisu On Kubernetes

Makisu makes it super easy to build images from a github repository inside Kubernetes. The overall design is that a single Pod (or Job) gets
created with a builder/worker container that will perform the build, and a sidecar container that will clone the repo and use the 
`makisu-client` to trigger the build in the sidecar container.

### Creating your registry configuration

Makisu will need to have your registry configuration mounted if you want it to push to your registry. The config format is described at the 
bottom of this Readme. Once you have your configuration on your local filesystem, you will need to create the k8s secret:
```shell
$ kubectl create secret generic docker-registry-config --from-file=./registry.yaml
secret/docker-registry-config created
```

### Building Git repositories

Building your image from a GitHub repo is super easy, we have a `makisu-client` image that has a simple entrypoint which lets you do 
that easily. If your repo is private, make sure to have your github token readable at `/makisu-secrets/github-token/github_token` inside
the client container. 
```shell
kubectl create secret generic github-token --from-file=./github_token
```
You will also need to mount the registry configuration after having created the secret for it. Below is a template to build your private 
github repository and push it to your registry.
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: imagebuilder
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: makisu-worker
        image: makisu-worker:6246b77
        imagePullPolicy: IfNotPresent
        args:
        - listen
        - -s
        - /makisu-socket/makisu.sock
        volumeMounts:
        - name: socket
          mountPath: /makisu-socket
        - name: context
          mountPath: /makisu-context
        - name: registry-config
          mountPath: /makisu-secrets/registry-config
      - name: makisu-manager
        image: makisu-client:6246b77
        imagePullPolicy: IfNotPresent
        args:
        - github.com/<your github repo>
        - --exit
        - build
        - -t=<your tag here>
        - --push=<your registry hostname here (eg: gcr.io)>
        - --registry-config=/makisu-secrets/registry-config/registry.yaml
        volumeMounts:
        - name: socket
          mountPath: /makisu-socket
        - name: context
          mountPath: /makisu-context
        - name: github-token
          mountPath: /makisu-secrets/github-token
      volumes:
      - name: socket
        emptyDir: {}
      - name: context
        emptyDir: {}
      - name: github-token
        secret:
          secretName: github-token
```
Once you have your job spec a simple `kubectl create -f job.yaml` will start your build. The job status will reflect whether or not the build failed.

## Build 

To build a docker image that can perform the builds (makisu-builder/makisu-worker) binary:
```
make image-builder image-worker
```

To get the makisu-builder binary locally and build _some_ images with no need for a containerizer:
```
go get github.com/uber/makisu/cmd/makisu-builder
```

## Run locally

If your dockerfile doesn't have RUN, you can use makisu-builder to build it without chroot, docker daemon or other containerizer.
To build a simple docker image and save it as a tar file:
```
makisu-builder build -t ${TAG} -dest ${TAR_PATH} ${CONTEXT}
```
To build a simple docker image and load into local docker daemon:
```
makisu-builder build -t ${TAG} -load ${CONTEXT}
```
To build a simple docker image and push to a registry:
```
makisu-builder build -t ${TAG} -push ${REGISTRY} ${CONTEXT}
```

## Run locally with Docker

To build a docker image in a docker container and load it into docker daemon:
```
docker run -i --rm --net host -v ${CONTEXT}:/context -v /var/run/docker.sock:/docker.sock -e DOCKER_HOST=unix:///docker.sock makisu-builder:latest build -t ${TAG} -modifyfs=true -load /context
```

## Configuring Docker Registry

Makisu supports TLS and Basic Auth with Docker Registry (Docker Hub, GCR, and private registries). It also contains a list of common root CA certs as default.
Pass your configuration file to Makisu with flag `-registry-config=${PATH_TO_CONFIG}`.
```
// Config contains docker registry client configuration.
type Config struct {
  Concurrency int           `yaml:"concurrency"`
  Timeout     time.Duration `yaml:"timeout"`
  Retries     int           `yaml:"retries"`
  PushRate    float64       `yaml:"push_rate"`
  // If not specify, a default chunk size will be used.
  // Set it to -1 to turn off chunk upload.
  // NOTE: gcr does not support chunked upload.
  PushChunk int64           `yaml:"push_chunk"`
  Security  security.Config `yaml:"security"`
}
```
To configure your own registry endpoint:
```
[registry]:
  [repo]:
    security:
      tls:
        client:
          enabled: true
          cert:
            path: //path to cert//
          key:
            path: //path to key//
          passphrase
            path: //path to passphrase//
        ca:
          cert:
            path: //path to ca certs, appends to system certs. A list of common ca certs are used if empty//
      basic:
        username: //username//
        password: //password//
```
Example:
```
"gcr.io":
  "makisu-project/*":
    push_chunk: -1
    security:
      tls:
        client:
          enabled: true
      basic:
        username: _json_key
        password: //password//
```
