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

## Scale up


## Update the app
