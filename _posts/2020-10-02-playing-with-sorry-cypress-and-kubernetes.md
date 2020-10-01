---
published: false
---
Here at ${job} we use Cypress for our test automation and I wanted to integrate it more deeply with our CI/CD stack which sits on top of Kubernetes. 

 ***Before we get into the more technical stuff, let's wind things back a little bit:***

- [Cypress](https://www.cypress.io/) is a test automation tool for testing websites.
[Cypress itself is free and open source](https://github.com/cypress-io/cypress). They make their money by offering a tool called Cypress Dashboard where you can view test results, plan parallel testing and store screenshots/videos of failed tests. I absolutely don't have a problem with paying for good software, however I do have a problem sharing screenshots and videos of my failed system, regardless of how secure Cypress might be.

- [Sorry Cypress](https://sorry-cypress.dev/) is an open source tool that aims to replace the Dashboard aspect of Cypress and it seems to do a decent job. You need to self-host it and are therefore responsible for its upkeep and the storage of potentially large screenshots/images. If you’re looking at Sorry-Cypress from a pure cost-saving perspective I wouldn’t expect to save much, especially when you factor in the time to install and maintain it. 

- [Kubernetes](https://kubernetes.io) is a container orchestration tool... but hopefully you’ve heard of that one. If you haven’t, I imagine this will be an even more boring article than I expected.
