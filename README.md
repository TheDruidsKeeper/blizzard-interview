# Development Lifecycle: Blue/Green Deployment

## Overview
A sample python application was provided for the interview challenge of demonstrating a blue/green deployment.

This demo will deploy the sample application to a kubernetes cluster (minikube in this case) using 2 separate deployments, each with their own version number.
A single service is also created, pointing at one version of the application.
Finally, a simple test client deployment is created to continuously make & log requests via the service.
No ingress is required for this test, as the client is running within the cluster.

Implementation Notes:
- Resources are created in the cluster using [kustomize](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/) to apply the manifests found in the `kubernetes` directory.
- The python script provided was stored unchanged in the `sample` directory and the dockerfile will override the `__VERSION__` variable while building the image.
- The main point of this is to demonstrate the blue/green deployment model, which is achieved by using a single service and modifying the `spec.selector.version` to match the unique deployment. This allows new deployments to be created, referencing new builds and/or other modifications to the pod(s), and readiness properly probed before updating the service.

Once the resources are created, you can connect a terminal to follow the logs of the test client while using another terminal to execute the upgrade patch.
The patch will edit the service to use the other app version, and the test client logs will show a seamless migration without delay or failures.

### Sample Logs
```text
[2022-10-01 04:10:26] {"version": "1.0.0", "errors": []} in 0.001979s
[2022-10-01 04:10:26] {"version": "1.0.0", "errors": []} in 0.001960s
[2022-10-01 04:10:26] {"version": "1.0.0", "errors": []} in 0.001767s
[2022-10-01 04:10:26] {"version": "1.0.0", "errors": []} in 0.002055s
[2022-10-01 04:10:26] {"version": "1.0.0", "errors": []} in 0.001994s
[2022-10-01 04:10:26] {"version": "1.0.0", "errors": []} in 0.061776s
[2022-10-01 04:10:26] {"version": "1.0.1", "errors": []} in 0.008438s
[2022-10-01 04:10:26] {"version": "1.0.1", "errors": []} in 0.001969s
[2022-10-01 04:10:26] {"version": "1.0.1", "errors": []} in 0.004019s
[2022-10-01 04:10:26] {"version": "1.0.1", "errors": []} in 0.004008s
[2022-10-01 04:10:26] {"version": "1.0.1", "errors": []} in 0.002081s
```

## Steps to Reproduce
_Note_: All commands are written to run from the repository root.

### Prerequisites
For this project you will need to have [docker](https://www.docker.com/), [kubectl](https://kubernetes.io/docs/tasks/tools/), and [minikube](https://minikube.sigs.k8s.io/docs/start/).

### Build and tag two versions of the app
_Note_: The build-arg used will replace the version string inside the python script
```powershell
docker build --build-arg VERSION=1.0.0 -t blizzard-sample:1.0.0 sample
docker build --build-arg VERSION=1.0.1 -t blizzard-sample:1.0.1 sample
```

### Use minikube to create a test cluster
```powershell
minikube start
```

### Push built images into minikube
```powershell
minikube image load blizzard-sample:1.0.0
minikube image load blizzard-sample:1.0.1
```

### Create the resources in Kubernetes
```powershell
kubectl create namespace blizzard
kubectl apply -k .\kubernetes\
```

### Verify the desired state
The following command should show you 3 running pods, and a service with the selector pointing at version `1.0.0`
```powershell
kubectl get pods,svc -n blizzard -o wide
```

### Monitor the test client logs
_Note_: You'll need to leave this terminal running while using another terminal to follow the next step
```powershell
kubectl -n blizzard logs --tail=0 -f deployment/test-client
```

### Apply the upgrade patch
```powershell
kubectl -n blizzard patch service app -p '{\"spec\":{\"selector\":{\"version\":\"1.0.1\"}}}'
```
_Note_: Immediately after running the above command you should ctrl-c the terminal watching the test client logs, so you can find the moment the version changed (it runs fast!). Also, the above command is formatted to run in powershell - if you'd like to run from bash you'll need to remove the `\`'s.

### Verify the updated desired state
The following command will show the service has been updated with the selector pointing at version `1.0.1`
```powershell
kubectl get pods,svc -n blizzard -o wide
```

### Cleanup
```powershell
minikube delete --all
```

## Next Steps
- Ideas?