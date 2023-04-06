---
title: Getting started with OpenFaaS and Golang on minikube
date: 2019-06-11 00:00:00
tags:
  - Golang
  - Kubernetes
  - k8s
  - OpenFaaS
---

## Set up environment on MacOS

- `brew install go docker-machine-driver-hyperkit faas-cli kubernetes-cli kubernetes-helm`
- `brew cask install minikube docker`

> docker-machine-driver-hyperkit can replace by virtualbox. Use `brew cask install virtualbox` to install it.

> docker-machine-driver-hyperkit and docker both have docker-machine. It may cause some error when homebrew create symbolic link. This error can be ignored since docker-machine will always point to docker's when starting docker for mac.

## Start minikube

### Use virtualbox (Default)

- `minikube start`

### Use hyperkit

- `minikube start --vm-driver=hyperkit`

> Or use `minikube config set vm-driver hyperkit` to modify default value.

## Deploy OpenFaaS to minikube

1. Create a service account for Helm’s server component (tiller): `kubectl -n kube-system create sa tiller && kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller`
2. Install tiller which is Helm’s server-side component: `helm init --skip-refresh --upgrade --service-account tiller`
3. Create namespaces for OpenFaaS core components and OpenFaaS Functions: `kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml`
4. Add the OpenFaaS helm repository: `helm repo add openfaas https://openfaas.github.io/faas-netes/`
5. Update all the charts for helm: `helm repo update`
6. Create a password (Remember this for login to OpenFaaS dashboard): `export PASSWORD=...` Random password example: `export PASSWORD=$(head -c 12 /dev/urandom | shasum| cut -d' ' -f1)`
7. Create a secret for the password: `kubectl -n openfaas create secret generic basic-auth --from-literal=basic-auth-user=admin --from-literal=basic-auth-password="$PASSWORD"`
8. Install OpenFaaS using the chart: `helm upgrade openfaas --install openfaas/openfaas --namespace openfaas --set functionNamespace=openfaas-fn --set basic_auth=true`
9. Finally once all the Pods are started you can login using the faas-cli: `echo -n $PASSWORD | faas-cli login -g http://$(minikube ip):31112 -u admin --password-stdin`
10. You can use `echo http://$(minikube ip):31112` to find dashboard URL and open it by browser (username: admin)

## Use faas-cli

### Generate a Go function

- `faas-cli new go-fn --lang go`

This command will generate a go-fn directory contains a simple template function response like this "Hello, Go. You said: (request body)" and a go-fn.yml file in working directory.

### Build function

1. Create a Docker Hub account
2. Add your Docker Hub username to go-fn.yml file's image tag
```yml
functions:
  go-fn:
    lang: go
    handler: ./go-fn
    image: USERNAME/go-fn:latest
```
3. Build docker image: `faas-cli build -f go-fn.yml`
4. Push to Docker Hub: `faas-cli push -f go-fn.yml`

You can find go-fn in your Docker Hub repository.

### Deploy function

- `faas-cli deploy -f go-fn.yml --gateway http://$(minikube ip):31112`

Then you can see this function in the dashboard. Invoke it by using dashboard UI or using faas-cli.

- `echo test | faas-cli invoke go-fn --gateway http://$(minikube ip):31112`

> Build, push and deploy can use just one command to finish these step `faas-cli up -f go-fn.yml --gateway http://$(minikube ip):31112`

## Develop with third-party package using Go module (Go 1.11+)

1. Init go module in go-fn directory: `go mod init PROJECT_NAME`
2. Import some third-party package in handler.go

example:
```go
package function

import (
	"fmt"

	"gopkg.in/loremipsum.v1"
)

// Handle a serverless request
func Handle(req []byte) string {
	text := loremipsum.New().Words(10)

	return fmt.Sprintf("Request body: %s\nResponse: %s", string(req), string(text))
}
```

3. Create vendor directory: `go mod vendor` (OpenFaaS use vendor to manage dependency)
4. Build and deploy this new function: `faas-cli up -f go-fn.yml --gateway http://$(minikube ip):31112`

## How to use this example repository

1. Deploy OpenFaaS
2. Clone this repository
3. Create vendor in go-fn directory
4. Modify yml file's Docker Hub username in image tag
5. Deploy function 

## Reference

- [OpenFaaS docs](https://docs.openfaas.com/cli/templates/)
- [Getting started with OpenFaaS on minikube](https://medium.com/faun/getting-started-with-openfaas-on-minikube-634502c7acdf)