---
published: true
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

Sorry Cypress allows us to parallelise our jobs, having multiple executors share the load of executing the tests, which results in a faster feedback loop to our development team, so we want to capitalise on that. The fly in the ointment is that Jenkins wants to run any containers for a given job in the same pod.... this means they have to run on the same physical (or virtual) server, which would easily become a constraint.

To overcome this, we use a 'leader' job to kick off a number of parallel 'worker' jobs that each run in their own pod. This way we can spread the load out across servers and we aren't resource constrained. The downside is it takes a little more work to set up, and to get your head around.

Here's something that resembles our leader job. It checks out the tests from git, does some database prep (not shown because 🤫) and then in parallel triggers additional jenkins jobs.

    def CI_ID = "cy-k8s-${BUILD_ID}"

    pipeline {
        agent{
            kubernetes {
                label 'cypress-internal'
                yaml """
    kind: Pod
    metadata:
      name: cypress-internal
    spec:
      containers:
      - name: cypress-internal
        image: [our-repo/container:latest]
        imagePullPolicy: Always
        command: ['cat']
        tty: true
        workingDir: /home/jenkins/agent/
        resources:
          requests:
            memory: "128M"
            cpu: 0.2
          limits:
            memory: "1Gi"
            cpu: 1
    """
            }
        }
        parameters {
            string defaultValue: '999.999.999.999', description: 'The test environment IP you wish to run the automation system on', name: 'test_env_IP', trim: true
        }

        stages {
            stage('SCM Checkout') {
                steps {
                    git branch: 'master',
                    credentialsId: '[jenkins credentials to access git]',
                    url: '[our git repo here].git'
                }
            }    

            stage('Testify!') {
                parallel{
                    stage('Invoke Worker 1') {
                        steps{
                            echo "Build ID is ${CI_ID}"
                            build job: 'cypress-workers', wait: true, propagate: true, parameters: [
                                string(name: 'CI_ID',  value: "${CI_ID}"),
                                string(name: 'test_env_IP',  value: "${test_env_IP}"),
                                string(name: 'workernum', value: "1"),
                            ]
                        }
                    }
                    stage('Invoke Worker 2') {
                        steps{
                            build job: 'cypress-workers', wait: true, propagate: true, parameters: [
                                string(name: 'CI_ID',  value: "${CI_ID}"),
                                string(name: 'test_env_IP',  value: "${test_env_IP}"),
                                string(name: 'workernum', value: "2"),
                            ]
                        }
                    }
                    stage('Invoke Worker 3') {
                        steps{
                            build job: 'cypress-workers', wait: true, propagate: true, parameters: [
                                string(name: 'CI_ID',  value: "${CI_ID}"),
                                string(name: 'test_env_IP',  value: "${test_env_IP}"),
                                string(name: 'workernum', value: "3"),
                            ]
                        }
                    }
                    stage('Invoke Worker 4') {
                        steps{
                            build job: 'cypress-workers', wait: true, propagate: true, parameters: [
                                string(name: 'CI_ID',  value: "${CI_ID}"),
                                string(name: 'test_env_IP',  value: "${test_env_IP}"),
                                string(name: 'workernum', value: "4"),
                            ]
                        }
                    }
                }
            }
        }
    }


Finally, our workers get instructed by Sorry Cypress which tests to run, and then crack on with it. Our worker job looks a little like this:

    pipeline {
        agent{
            kubernetes {
                label 'cypress-internal'
                yaml """
    kind: Pod
    metadata:
      name: cypress-internal
    spec:
      containers:
      - name: cypress-internal
        image: [our-repo/container:latest]
        imagePullPolicy: Always
        command: ['cat']
        tty: true
        workingDir: /home/jenkins/agent/
        resources:
          requests:
            memory: "2Gi"
            cpu: 2.5
          limits:
            memory: "4Gi"
            cpu: 3
    """
            }
        }
        parameters {
            string defaultValue: '', description: 'The test environment IP you wish to run the automation system on', name: 'test_env_IP', trim: true
            string defaultValue: '', description: 'The CI ID to pass to Sorry Cypress', name: 'CI_ID'
            string defaultValue: '', description: 'workernum', name: 'workernum'
            booleanParam defaultValue: false, description: 'Check to record the run to the Cypress Dashboard', name: 'record_run'
        }

        environment {
              JENKINS_VAULT_TOKEN = credentials('jenkins-token')
              APPROLE_NAME        = "cypress-automation"
              APPROLE_ID          = "89ebc42c-8b0f-677e-ac65-f1ec1b87f5b6"
        }

        stages {
            stage('SCM Checkout') {
                steps {
                    git branch: 'master',
                    credentialsId: '[jenkins credentials to access git]',
                    url: '[our git repo here].git'
                }
            }
        }
            stage('Testify!') {
                environment {
                CYPRESS_RECORD_KEY = "record-key"
                }
                steps{
                    container('cypress-internal') {
                        script{
                          sh 'npx cypress@5.3.0 run -e worker=Worker${workernum},IP=${test_env_IP},tag=k8sJenkins --browser chrome --headless --parallel --record --ci-build-id ${CI_ID} --config testFiles=**${features_to_test}/*.feature'
                        }
                    }
                }
            }
        }
    }
  
You'll notice the quite hefty resource request numbers on the worker containers. This value was derived after a lot of trial and error to find the optimum number that worked for us.

We use EKS and make use of autoscaling, so if we have a large number of test requests, we will automatically scale up the number of available servers to accommodate the load. This also means we will drop down the number of servers when the load reduces, thereby saving a bit of money.

Hopefully this was useful. If you want any more information, do get in touch.
