---
published: false
---

Continuing on [the Sorry Cypress theme](https://crumbhole.com/playing-with-sorry-cypress-and-kubernetes/). I thought it might be useful to demonstrate how I integrate our cypress tests in a Jenkins pipeline. There are other example of this on the web but they don't cater for Jenkins running in Kubernetes, spinning up kubernetes pods as the executors.

Due to this being used at ${job}, I can't share our full pipelines and Dockerfiles with you, but hopefully this will give you enough information to get started.

## Preparing a Docker Container

Firstly, we tweak the [Cypress docker images](https://github.com/cypress-io/cypress-docker-images) to add in our own dependencies as well as to change the url of the cypress dashboard from the official one to our implementation of Sorry Cypress.

Our Dockerfile looks a little like this:
    
    FROM cypress/browsers:node12.14.1-chrome85-ff81

    # Pull dependencies from repo and install them
    COPY ["package.json", "package-lock.json", "yarn.lock", "/"]
    RUN npm install

    # Set to use Sorry Cypress
    RUN sed -i 's/api.cypress.io/our-internal-sorry-cypress-director-url.com/g' /root/.cache/Cypress/5.3.0/Cypress/resources/app/packages/server/config/app.yml

We use a tool called [Kaniko](https://github.com/GoogleContainerTools/kaniko) which allows us to build and push docker images from within Kubernetes (i.e. 'Docker-in-Docker'). I won't go into too much detail about Kaniko as it's well documented on the web.

The resulting Jenkins pipeline to do this looks a little like this:

    pipeline {
        agent {
            kubernetes {
                label 'kaniko'
                yaml """
    kind: Pod
    metadata:
      name: kaniko
    spec:
      containers:
      - name: kaniko
        image: gcr.io/kaniko-project/executor:debug
        imagePullPolicy: Always
        command:
        - /busybox/cat
        tty: true
        volumeMounts:
          - name: jenkins-docker-cfg
            mountPath: /kaniko/.docker
      volumes:
      - name: jenkins-docker-cfg
        projected:
          sources:
          - secret:
              name: registry-credentials
              items:
                - key: .dockerconfigjson
                  path: config.json
    """
            }
        }
        stages {
            stage('Checkout') {
                steps {
                    git branch: 'master',
                    credentialsId: '[jenkins credentials to access git]',
                    url: '[our git repo here].git'
                }
            }
            stage('Make and push Image') {
                environment {
                    PATH        = "/busybox:$PATH"
                    REGISTRY    = '[out internal docker repo]'
                    REPOSITORY  = 'cypress'
                    IMAGE       = 'cypress'
                }
                steps {
                    container(name: 'kaniko', shell: '/busybox/sh') {
                        sh '''#!/busybox/sh
                        /kaniko/executor -f `pwd`/Dockerfile -c `pwd` --cache=true --destination=${REGISTRY}/${REPOSITORY}/${IMAGE}
                        '''
                    }
                }
            }
        }
    }


We now have a container that has all our dependencies baked into it, as well as the path to our Sorry Cypress installation. We can now look to implement CI.

Sorry Cypress allows us to parallelise our jobs, having multiple exectors share the load of executing the tests, which results in a faster feedback loop to our development team. 

