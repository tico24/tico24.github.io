---
published: true
---
For some reason, the official Cypress Docker images are a bit odd. They seem to be programatically created but there's never any consistency between npm versions and browser versions. They contain repetitions across the layers which cause them to be bulkier than they need to be too. To make things worse, their images contain [critical vulnerabilities that they have no interest in resolving](https://github.com/cypress-io/cypress-docker-images/issues/370).

Frustrated by their images, I have knocked together my own. I don't think it is perfect, but it's easier for you to configure yourself to suit your needs, and it's less vulnerability-ridden!

I have run my image through a Trivy scanner and there are 138 medium (and lower) vulnerabilities. 0 of them are fixable. This is down from over 2000 fixable critical vulnerabilities found on the official Cypress docker images.

It is built off of ubuntu 20.04. You can [modify the Dockerfile](https://github.com/tico24/cypress-images) to specify your specific browser and node versions and build it locally (which I'd recommend).

Alternatively, there's an example containing Firefox 81 and Chrome 86 on [Docker Hub](https://hub.docker.com/r/tico24/cypress-images). 
