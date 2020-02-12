# Run Istio E2E tests on IKS

```
docker pull gcr.io/istio-testing/app:latest
docker tag gcr.io/istio-testing/app:latest rvennam/app:1.4.4
docker push rvennam/app:1.4.4

export TAG=1.4.4
export HUB=docker.io/rvennam
export GOPATH=/Users/rvennam/go

cd ~/go/src/istio.io/istio/tests

https://raw.githubusercontent.com/istio/istio/1.4.4/tests/integration/security/testdata/global-mtls-on.yaml

cd ~/go/src/istio.io/istio/tests

./e2e_istio_preinstalled.sh simple
```
