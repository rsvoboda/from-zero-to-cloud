# From Zero To Cloud
From Zero To Cloud - Minikube setup and application deploy (on macOS machine).

## Minikube
Tools installation via Brew.
```
brew cask install minikube
brew install kubectl

minikube delete
minikube start --memory=4096 --cpus=4 --disk-size=40g
```

Minikube context for kubectl
```
kubectl config use-context minikube
kubectl get pods --context=minikube
```

## First app
Application based on Quarkus compiled to the native binary.
```
mkdir from-zero-to-cloud && cd from-zero-to-cloud

setJdkGraalVM
mvn io.quarkus:quarkus-maven-plugin:0.13.3:create \
   -DprojectGroupId=org.acme \
   -DprojectArtifactId=getting-started \
   -DclassName="org.acme.quickstart.GreetingResource" \
   -Dpath="/hello"
cd getting-started
mvn package && mvn package -Pnative
```

## Push to Minikube
Push the docker image directly to the Docker in Minikube.
```
eval $(minikube docker-env)
docker build -f src/main/docker/Dockerfile.native -t quarkus-quickstart/quickstart .

kubectl run quarkus-quickstart --image=quarkus-quickstart/quickstart:latest --port=8080 --image-pull-policy=IfNotPresent
kubectl expose deployment quarkus-quickstart --type=NodePort

curl $(minikube service quarkus-quickstart --url)/hello
```
Application was deployed but not running, time for investigation.

## Investigation
```
kubectl cluster-info
kubectl get nodes
kubectl describe node minikube

kubectl get pods
kubectl describe pod quarkus-quickstart-6f44484bf6-8r2f5

kubectl -n default logs quarkus-quickstart-6f44484bf6-8r2f5
```

Another way to do investigation is using kubernetes dashboard in your default browser.
```
minikube dashboard
```

Message from the log is:
```
standard_init_linux.go:207: exec user process caused "exec format error"
```
This means minikube can't execute delivered binary.

Native executable was produced during the build but it was native binary for macOS.
Option `-Dnative-image.docker-build=true` must be added to build binary usable for Linux / Docker.


## Second try
```
eval $(minikube docker-env -u)

mvn package -Pnative -Dnative-image.docker-build=true

eval $(minikube docker-env)
docker build -f src/main/docker/Dockerfile.native -t quarkus-quickstart/quickstart .

kubectl delete service quarkus-quickstart
kubectl delete deployment quarkus-quickstart

kubectl run quarkus-quickstart --image=quarkus-quickstart/quickstart:latest --port=8080 --image-pull-policy=IfNotPresent
kubectl expose deployment quarkus-quickstart --type=NodePort

curl $(minikube service quarkus-quickstart --url)/hello
```
The app is running.

## Scale up / down
```
kubectl scale --replicas=20 deployment/quarkus-quickstart

kubectl get pod
kubectl get deployments

kubectl scale --replicas=1 deployment/quarkus-quickstart
```

Or via Deployments section of `minikube dashboard`.

## Update the app
```
# do changes in your application
mvn package -Pnative -Dnative-image.docker-build=true

eval $(minikube docker-env -u)
docker build -f src/main/docker/Dockerfile.native -t quarkus-quickstart/quickstart:v2.1 .
docker images

kubectl set image deployment/quarkus-quickstart quarkus-quickstart=quarkus-quickstart/quickstart:v2.1
kubectl get pod

curl $(minikube service quarkus-quickstart --url)/hello
```

```
mvn package -Pnative -Dnative-image.docker-build=true

docker build -f src/main/docker/Dockerfile.native -t quarkus-quickstart/quickstart:v2.2 .
kubectl set image deployment/quarkus-quickstart quarkus-quickstart=quarkus-quickstart/quickstart:v2.2

curl $(minikube service quarkus-quickstart --url)/hello
```

## Decrease Docker image size
Currently the Dockerfile.native image is based on fedora-minimal which has size ~105MB.
We can go even further using "distroless" image - e.g. https://github.com/quarkusio/quarkus-images/tree/graalvm-1.0.0-rc14/distroless which has 17MB. Final image with getting-started app has around 37MB.

```
docker images | grep quarkus/getting-started
quarkus/getting-started-distroless  latest   f4594f855dde   About a minute ago  36.9MB
quarkus/getting-started             latest   57c659fa7991   3 minutes ago       125MB
```

Custom src/main/docker/Dockerfile.distroless:
```
####
# This Dockerfile is used in order to build a "distroless" container that runs the Quarkus application in native (no JVM) mode
#
# Before building the docker image run:
#
# mvn package -Pnative -Dnative-image.docker-build=true
#
# Then, build the image with:
#
# docker build -f src/main/docker/Dockerfile.distroless -t quarkus/getting-started-distroless .
#
# Then run the container using:
#
# docker run -i --rm -p 8080:8080 quarkus/getting-started-distroless
#
###
FROM cescoffier/native-base:latest
COPY target/*-runner /application
EXPOSE 8080
CMD ["./application", "-Dquarkus.http.host=0.0.0.0"]
```