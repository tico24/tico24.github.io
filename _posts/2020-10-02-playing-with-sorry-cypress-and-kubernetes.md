---
published: true
---
Here at ${job} we use Cypress for our test automation and I wanted to integrate it more deeply with our CI/CD stack which sits on top of Kubernetes.

_Updated 12 October 2020_

## **Before we get into the more technical stuff, let's wind things back a little bit:**

- [Cypress](https://www.cypress.io/) is a test automation tool for testing websites.
[Cypress itself is free and open source](https://github.com/cypress-io/cypress). They make their money by offering a tool called Cypress Dashboard where you can view test results, plan parallel testing and store screenshots/videos of failed tests. I absolutely don't have a problem with paying for good software, however I do have a problem sharing screenshots and videos of my failed system, regardless of how secure Cypress might be.

- [Sorry Cypress](https://sorry-cypress.dev/) is an open source tool that aims to replace the Dashboard aspect of Cypress and it seems to do a decent job. You need to self-host it and are therefore responsible for its upkeep and the storage of potentially large screenshots/images. If you’re looking at Sorry-Cypress from a pure cost-saving perspective I wouldn’t expect to save much, especially when you factor in the time to install and maintain it. 

- [Kubernetes](https://kubernetes.io) is a container orchestration tool... but hopefully you’ve heard of that one. If you haven’t, I imagine this will be an even more boring article than I expected.

---
**_tl;dr: [The YAML to deploy Sorry Cypress to Kubernetes can be found here](https://raw.githubusercontent.com/tico24/sorry-cypress/master/kubernetes-full.yml)_**

---

## Hosting Sorry Cypress on Kubernetes

Sorry Cypress has already been containerised by the developer and he has included some docker-compose examples. Unfortunately there arent any Kubernetes examples - this will hopefully be addressed soon as I'll be including the examples in this post in a [pull request](https://github.com/sorry-cypress/sorry-cypress/pull/151). While docker compose and Kubernetes share the same language in YAML, there is no direct correlation between the two so a small amount of work is needed to translate from one to the other.

**You will need:**
- A Kubernetes cluster
- Access to S3 (or equivalent, eg [minio](https://min.io/) or [Backblaze b2](https://www.backblaze.com/b2/cloud-storage.html))

**Caveat time!**

This is some pretty basic yaml. If you're going to host this in production with important data, I'd strongly recommend you embelish it in certain areas. I'll try and point some of these out as I explain the yaml. But here's a couple of caveats to get us started:

- The pod request and limit values are completely arbitrary. Please do your own research to find out values that suit your needs.
- Readiness probe timings are also completely arbitrary and are almost certainly not the most efficient.

## The Kubernetes Yaml

1. A persistent Volume:
If you want your mongo database to persist when the pod gets removed, you'll want a persistent volume. This one's pretty self explanatory, but to give a little context, I host on EKS so I'll be using the default gp2 storage class:

``` yml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mongo-storage-claim
  labels:
    app: mongo-storage-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "gp2"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

2. The database deployment:
If you're going to plop this into production, I'd recommend you use a more robust mongo deployment than this, maybe the [Bitnami Mongo Helm Chart](https://github.com/bitnami/charts/tree/master/bitnami/mongodb) or a hosted solution such as AWS' [DocumentDB](https://aws.amazon.com/documentdb/).

I'm using a slightly older mongo version than is available as this aligns with the developer's docker compose examples.

``` yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      name: mongo
      labels: 
        app: mongo
    spec:
      containers:
      - image: mongo:4.0
        imagePullPolicy: "Always"
        name: mongo
        ports:
        - containerPort: 27017
        readinessProbe:
          exec:
            command:
            - mongo
            - --eval
            - db.adminCommand('ping')
          failureThreshold: 6
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
        resources:
          requests:
            memory: "128M"
            cpu: 0.2
          limits:
            memory: "512M"
            cpu: 0.5
        volumeMounts:
          - name: mongo-storage
            mountPath: /data/db
      restartPolicy: Always
      serviceAccountName: ""
      volumes:
        - name: mongo-storage
          persistentVolumeClaim:
            claimName: mongo-storage-claim
```

3. The API deployment:
Note that the API deployment has no readiness probe. I didn't have a lot of time, but I couldn't get anything to respond positively.

``` yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
      name: api
    spec:
      containers:
      - env:
        - name: MONGODB_DATABASE
          value: sorry-cypress
        - name: MONGODB_URI
          value: mongodb://mongo-service:27017
        image: agoldis/sorry-cypress-api:latest
        imagePullPolicy: "Always"
        name: api
        ports:
        - containerPort: 4000
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 2
          failureThreshold: 5
        resources:
          requests:
            memory: "128M"
            cpu: 0.2
          limits:
            memory: "512M"
            cpu: 0.5
      restartPolicy: Always
      serviceAccountName: ""
      volumes: null
```

4. The Director deployment:
This references a couple of secrets that we'll come back to later.
You'll need to enter the dashboard URL and the S3 details you're expecting.

``` yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: director
spec:
  replicas: 1
  selector:
    matchLabels:
      app: director
  template:
    metadata:
      labels:
        app: director
      name: director
    spec:
      containers:
      - env:
        - name: DASHBOARD_URL
          value: [YOUR-DASHBOARD-URL-HERE]
        - name: EXECUTION_DRIVER
          value: ../execution/mongo/driver
        - name: MONGODB_DATABASE
          value: sorry-cypress
        - name: MONGODB_URI
          value: mongodb://mongo-service:27017
        - name: SCREENSHOTS_DRIVER
          value: ../screenshots/s3.driver
        - name: S3_BUCKET
          value: [YOUR-BUCKET-NAME-HERE]
        - name: S3_REGION
          value: [YOUR-REGION-HERE]
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
              secretKeyRef:
                name: cypress-s3-secrets
                key: AWS_ACCESS_KEY_ID
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
              secretKeyRef:
                name: cypress-s3-secrets
                key: AWS_SECRET_ACCESS_KEY
        image: agoldis/sorry-cypress-director:latest
        imagePullPolicy: "Always"
        name: director
        ports:
        - containerPort: 1234
        readinessProbe:
          httpGet:
            path: /
            port: 1234
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 2
          failureThreshold: 5
        resources:
          requests:
            memory: "128M"
            cpu: 0.2
          limits:
            memory: "512M"
            cpu: 0.5
      restartPolicy: Always
      serviceAccountName: ""
      volumes: null
```

5. The Dashboard Deployment:
You just need to pop your API url under GRAPH_SCHEMA_URL.

``` yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dashboard
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dashboard
  template:
    metadata:
      labels:
        app: dashboard
      name: dashboard
    spec:
      containers:
      - env:
        - name: GRAPHQL_SCHEMA_URL
          value: [YOUR-API-URL-HERE]
        image: agoldis/sorry-cypress-dashboard:latest
        imagePullPolicy: "Always"
        name: dashboard
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 2
          failureThreshold: 5
        resources:
          requests:
            memory: "128M"
            cpu: 0.2
          limits:
            memory: "512M"
            cpu: 0.5
      restartPolicy: Always
      serviceAccountName: ""
      volumes: null
```

6. Services:
I'll lump all the services together here. Fairly self-explainatory.

``` yml
apiVersion: v1
kind: Service
metadata:
  name: dashboard-service
spec:
  ports:
  - name: "8080"
    port: 8080
    targetPort: 8080
  selector:
    app: dashboard
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  ports:
  - name: "4000"
    port: 4000
    targetPort: 4000
  selector:
    app: api
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-service
spec:
  ports:
  - name: "27017"
    port: 27017
    targetPort: 27017
  selector:
    app: mongo
---
apiVersion: v1
kind: Service
metadata:
  name: director-service
spec:
  ports:
  - name: "1234"
    port: 1234
    targetPort: 1234
  selector:
    app: director
```

7. Ingresses:

``` yml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cypress-ingresses
spec:
  rules:
  - host: [YOUR-DASHBOARD-URL-HERE]
    http:
      paths:
      - backend:
          serviceName: dashboard-service
          servicePort: 8080
  - host: [YOUR-API-URL-HERE]
    http:
      paths:
      - backend:
          serviceName: api-service
          servicePort: 4000
  - host: [YOUR-DIRECTOR-URL-HERE]
    http:
      paths:
      - backend:
          serviceName: director-service
          servicePort: 1234
```

8. Secrets:
We want to make our S3 ACCESS_KEY_ID and SECRET_ACCESS_KEY into Kubernetes secrets.

Kubernetes secrets need to be base64 encoded, so in your terminal, you'll want to do the following for your access key ID and your secret access key:

``` terminal
$ echo -n YOUR-TEXT-TO-ENCODE | base64
```
        
You then insert these into your yaml:

``` yml
apiVersion: v1
kind: Secret
metadata:
  name: cypress-s3-secrets
data:
  AWS_SECRET_ACCESS_KEY: VGhpcyBpcyB0aGUgY3J1bWJob2xl
  AWS_ACCESS_KEY_ID: Tm8gcGVla2luZw==
```

And that's it. 

To deploy, you can simply put it all in one .yaml file ([as I have done here](https://raw.githubusercontent.com/tico24/sorry-cypress/master/kubernetes-full.yml)) and use kubectl to apply it.

Once your DNS has propagated you should be able to visit your dashboard URL. Point your runner at your director URL and it'll use Sorry Cypress instead of the hosted offering.