---

sectionid: apidb
sectionclass: h2
parent-id: upandrunning
title: Deploy the Order Capture API
---

You need to deploy the **Order Capture API** ([azch/captureorder](https://hub.docker.com/r/azch/captureorder/)). This requires an external endpoint, exposing the API on port 80 and needs to write to MongoDB.

### Container images and source code

In the table below, you will find the Docker container images provided by the development team on Docker Hub as well as their corresponding source code on GitHub.

| Component                    | Docker Image                                                     | Source Code                                                       | Build Status |
|------------------------------|------------------------------------------------------------------|-------------------------------------------------------------------|--------------|
| Order Capture API            | [azch/captureorder](https://hub.docker.com/r/azch/captureorder/) | [source-code](https://github.com/Azure/azch-captureorder)         | [![Build Status](https://dev.azure.com/theazurechallenge/Kubernetes/_apis/build/status/Code/Azure.azch-captureorder)](https://dev.azure.com/theazurechallenge/Kubernetes/_build/latest?definitionId=10) |

### Environment variables

The Order Capture API requires certain environment variables to properly run and track your progress. Make sure you set those environment variables.

  * `TEAMNAME="[YourTeamName]"`
    * Track your team's progress. Use your assigned team name
  * `CHALLENGEAPPINSIGHTS_KEY="[AsSpecifiedAtTheEvent]"`
    * Application Insights key provided by proctors
  * `APPINSIGHTS_KEY="[YourOwnKey]"`
    * Your own Application Insights key, if you want to track application metrics
  * `MONGOHOST="<hostname of mongodb>"`
    * MongoDB hostname.
  * `MONGOUSER="<mongodb username>"`
    * MongoDB username.
  * `MONGOPASSWORD="<mongodb password>"`
    * MongoDB password.

> **Hint:** The Order Capture API exposes the following endpoint for health-checks: `http://[PublicEndpoint]:[port]/healthz`

### Tasks

#### Provision the `captureorder` deployment and expose a public endpoint

{% collapsible %}

##### Deployment

Save the YAML below as `captureorder-deployment.yaml` or download it from [captureorder-deployment.yaml](yaml-solutions/01. challenge-02/captureorder-deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: captureorder
spec:
  selector:
      matchLabels:
        app: captureorder
  replicas: 2
  template:
      metadata:
        labels:
            app: captureorder
      spec:
        containers:
        - name: captureorder
          image: azch/captureorder
          imagePullPolicy: Always
          readinessProbe:
            httpGet:
              port: 8080
              path: /healthz
          livenessProbe:
            httpGet:
              port: 8080
              path: /healthz
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "500m"
          env:
          - name: TEAMNAME
            value: "team-azch"
          - name: APPINSIGHTS_KEY
            value: ""
          - name: MONGOHOST
            value: "orders-mongo-mongodb.default.svc.cluster.local"
          - name: MONGOUSER
            value: "orders-user"
          - name: MONGOPASSWORD
            value: "orders-password"
          ports:
          - containerPort: 80
```

And deploy it using

```sh
kubectl apply -f captureorder-deployment.yaml
```

##### Verify that the pods are up and running

```sh
kubectl get pods -l app=captureorder
```

> **Hint** If the pods are not starting, not ready or are crashing, you can view their logs using `kubectl logs <pod name>` and `kubectl describe pod <pod name>`.

##### Service

Save the YAML below as `captureorder-service.yaml` or download it from [captureorder-service.yaml](yaml-solutions/01. challenge-02/captureorder-service.yaml)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: captureorder
spec:
  selector:
    app: captureorder
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: LoadBalancer
```

And deploy it using

```sh
kubectl apply -f captureorder-service.yaml
```

##### Retrieve the External-IP of the Service

Use the command below. Make sure to allow a couple of minutes for the Azure Load Balancer to assign a public IP.

```sh
kubectl get service captureorder -o jsonpath="{.status.loadBalancer.ingress[*].ip}"
```

{% endcollapsible %}

#### Ensure orders are successfully written to MongoDB

{% collapsible %}
Send a `POST` request using [Postman](https://www.getpostman.com/) or curl to the IP of the service you got from the previous command

```sh
curl -d '{"EmailAddress": "email@domain.com", "Product": "prod-1", "Total": 100}' -H "Content-Type: application/json" -X POST http://[Your Service Public LoadBalancer IP]/v1/order
```

You should get back the created order ID

```json
{
    "orderId": "5beaa09a055ed200016e582f"
}
```

{% endcollapsible %}

> **Resources**
> * <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>
> * <https://kubernetes.io/docs/concepts/services-networking/service/>