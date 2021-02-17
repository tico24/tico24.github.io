---
published: true
---
Yes, it's another Sorry Cypress post, sorry!

Ever since [I posted information on how to deploy Sorry Cypress to Kubernetes](https://crumbhole.com/playing-with-sorry-cypress-and-kubernetes/), there's been quite a lot of noise from people asking for a helm chart version. So I finally found some time to make one.


# tl;dr
The chart [can be found here](https://github.com/sorry-cypress/charts)

## Installing

Install the chart using:

```bash
$ helm repo add sorry-cypress https://sorry-cypress.github.io/charts
$ helm install my-release sorry-cypress/sorry-cypress
```

## Upgrading

Upgrade the chart deployment using:

```bash
$ helm upgrade my-release sorry-cypress/sorry-cypress
```

## Uninstalling

Uninstall the my-release deployment using:

```bash
$ helm uninstall my-release
```

# More words
By default, the chart deploys everything to Kubernetes (as you'd expect), but there's no persistence, this is so that you can just get up and running as quickly as possible.

By default, the director uses the in-memory executionDriver, and the dummy screenshotsDriver. However, You can choose to enable a persistent mongo database for execution, and you can use S3 for screenshots.

Do have a play and if you have any feedback, it's probably best to [open an issue](https://github.com/sorry-cypress/charts/issues).