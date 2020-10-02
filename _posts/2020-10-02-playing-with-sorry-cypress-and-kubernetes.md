---
published: true
---
Here at ${job} we use Cypress for our test automation and I wanted to integrate it more deeply with our CI/CD stack which sits on top of Kubernetes. 

## **Before we get into the more technical stuff, let's wind things back a little bit:**

- [Cypress](https://www.cypress.io/) is a test automation tool for testing websites.
[Cypress itself is free and open source](https://github.com/cypress-io/cypress). They make their money by offering a tool called Cypress Dashboard where you can view test results, plan parallel testing and store screenshots/videos of failed tests. I absolutely don't have a problem with paying for good software, however I do have a problem sharing screenshots and videos of my failed system, regardless of how secure Cypress might be.

- [Sorry Cypress](https://sorry-cypress.dev/) is an open source tool that aims to replace the Dashboard aspect of Cypress and it seems to do a decent job. You need to self-host it and are therefore responsible for its upkeep and the storage of potentially large screenshots/images. If you’re looking at Sorry-Cypress from a pure cost-saving perspective I wouldn’t expect to save much, especially when you factor in the time to install and maintain it. 

- [Kubernetes](https://kubernetes.io) is a container orchestration tool... but hopefully you’ve heard of that one. If you haven’t, I imagine this will be an even more boring article than I expected.

### **_tl;dr: The YAML to deploy Sorry Cypress to Kubernetes can be found here:_**

## Hosting Sorry Cypress on Kubernetes

Sorry Cypress has already been containerised by the developer and he has included some docker-compose examples. Unfortunately there arent any Kubernetes examples - this will hopefully be addressed soon as I'll be including the examples in this post in a pull request. While docker compose and Kubernetes share the same language in YAML, there is no direct correlation between the two so a small amount of work is needed to translate from one to the other.

**You will need:**
- A Kubernetes cluster
- Access to S3 (or equivalent, eg [minio](https://min.io/) or [Backblaze b2](https://www.backblaze.com/b2/cloud-storage.html)

**Caveat time!**

This is some pretty basic yaml. If you're going to host this in production with important data, I'd strongly recommend you embelish it in certain areas. I'll try and point some of these out as I explain the yaml.

## The Kubernetes Yaml

1. A persistent Volume:
If you want your mongo database to persist when the pod gets removed, you'll want a persistent volume. This one's pretty self explanatory, but to give a little context, I host on EKS so I'll be using the default gp2 storage class:

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