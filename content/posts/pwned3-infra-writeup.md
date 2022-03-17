+++
date = "2022-03-17"
title = "pwnED 3 infrastructure writeup"
description = "Overview of infrastructure architecture for pwnEd 3 CTF"
math = true
authors = ["Atte Niemi"]
+++

This post is going to be about the infrastructure behind pwnEd 3, the third installment of the annual CTF competition hosted by SIGINT.
45 teams or approximately 155 users signed up. We used Google Cloud Platform to host CTFd, and DigitalOcean to host our challenges on a Kubernetes cluster.
#### CTFd
The complete CTFd terraform deployment is available [here](https://github.com/hur/ctfd-gcp). CTFd is deployed on App Engine Flex, which allows for scaling the number of replicas up and down very easily, with load balancing and health checks and other nice-to-haves. We used Redis Memorystore as a cache for CTFd. The smallest available Redis instance was more than enough for our CTF, and CTFd mkes efficient use of a cache with a ~99% cache hit rate. We used Cloud SQL as the database for CTFd. The combination of App Engine Flec, Cloud SQL and Redis Memorystore allows for easy scaling up of all the components of CTFd. Before the competition, we had a single instance of CTFd for uploading challenges and other administrative tasks. The night before the CTF we bumped the number of replicas up to 3. 

We also used a Google Storage Bucket for CTFd's challenge storage. This required setting up a service account with a HMAC key for S3 interoperability, since CTFd expects a S3 bucket. Public access to the bucket is disabled but users can access files via CTFd, making our challenge files safe from pre-CTF enumeration. 

Everything in this deployment was in a private VPC. Well, the google-managed Cloud SQL and Redis don't technically exist in the VPC, but with a VPC access connector for Cloud SQL and VPC Peering for Redis and public IP disabled for both, they can only be accessed from the VPC. Then, the only publicly available endpoint is the App Engine Flex application. Access to storage buckets was via IAM permissions on the app engine service account. 

#### Challenges
We hosted challenges on a DigitalOcean Kubernetes cluster with 5 nodes. 
I wrote a helm template to allow deploying challenges easily.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ $challenge.name }}-deployment
  namespace: {{ $challenge.name }}-namespace 
  labels: 
    challenge: {{ $challenge.name }}
    category: {{ $challenge.category }} 
spec:
  replicas: {{ $challenge.replicas }}
  selector: 
    matchLabels:
      challenge: {{ $challenge.name }}
  template:
    metadata:
      annotations:
      labels:
        challenge: {{ $challenge.name }}
        category: {{ $challenge.category }} 
    spec:
      enableServiceLinks: false
      automountServiceAccountToken: false
      # We want pods of a challenge to be evenly spread across nodes for redundancy
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule # Don't schedule 2 of the same pod onto the same node
          labelSelector:
            matchLabels:
              challenge: {{ $challenge.name }} 
      imagePullSecrets:
        - name: {{ $top.Values.doRegistrySecretName }}
      containers:
        - name: {{ $challenge.name }}
          image: {{ $challenge.image }}
          imagePullPolicy: Always
          ports:
            - containerPort: {{ $challenge.containerPort }} 
```
Each pod was behind a simple ClusterIP service.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ $challenge.name }}-service
  namespace: {{ $challenge.name }}-namespace
  labels:
    category: {{ $challenge.category }}
    challenge: {{ $challenge.name }} 
spec:
  selector:
    challenge: {{ $challenge.name }}
  ports:
    - port: {{ $challenge.containerPort }} 
{{ end }}
```

To allow connections to the services from outside the cluster, we deployed nginx-ingress to perform the load balancing, with 3 replicas of the ingress controller for redundancy. 
```yaml
ingress-nginx:
  controller:
    service:
      enabled: true
      externalTrafficPolicy: Local
    replicaCount: 3 
    # spread the ingress controllers across nodes for reliability
    topologySpreadConstraints: 
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule 
        labelSelector:
          matchLabels:
            app.kubernetes.io/name: ingress-nginx
            app.kubernetes.io/component: controller
            # This should be the same as the release name
            app.kubernetes.io/instance: challenges
  config:
    use-proxy-protocol: true
```

This was preferred over having a load balancer for each service as they are $10/month per node, so price can stack up quite fast especially if we want our load balancers to be highly available. 

For routing to different challenges, we considered 2 options. Name based virtual hosting and TCP port based routing.

Name based virtual hosting routes based on subdomain, so web1.challenges.sigint.mx could point to a web challenge. This looks quite nice, but doesn't support TCP challenges and there was concern that people might be able to enumerate our subdomains before the CTF. 

Port based routing simply points challenges.sigint.mx:4773 to a challenge, and it supports general TCP ports so challenges that should be connected to via  `netcat`  can be routed to as well. This is what we used in pwnEd 3, and it worked well. For each challenge, we configured a routing rule to the challenge's service.

##### Challenge upload workflow
With our templates in place, the workflow for deploying the challenge was simple. 
1. Build and push a docker image of the challenge to a private docker repository. 
2. Add the following to our `values.yaml` file 
```yaml
tcp_challenges:
	name: ssh01
    image: registry.digitalocean.com/<registry_name>/ssh01:latest
    containerPort: 1337
    category: pwn
    replicas: *replicaCount # defined earlier, = 3
```
3. Add a TCP routing rule to ingress-nginx
```yaml
ingress-nginx:
  tcp:
    11000: ssh01-namespace/ssh01-service:1337
```
4. Run `helm upgrade challenges .` to deploy the new challenge to the cluster

This was a lot better than copy/pasting yaml files for each challenge, but for next year I plan on improving the workflow more, as this is still quite unpolished. The ideal would be to automatically build and deploy docker containers to the cluster from a git repo. 

Having a declarative deployment is quite nice since the whole deployment is described in a few terraform and yaml files. Makes it very easy to reuse and improve for subsequent CTFs.
a
