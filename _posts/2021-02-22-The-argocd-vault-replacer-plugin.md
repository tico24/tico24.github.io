---
published: true
---
I recently collaborated on an [Argo CD plugin](https://argoproj.github.io/argo-cd/) called ArgoCD-Vault-Replacer. It allows you to merge your code in Git with your secrets in [Hashicorp Vault](https://www.vaultproject.io/) to deploy into your Kubernetes cluster(s). It supports ‘normal’ Kubernetes yaml (or yml) manifests (of any type) as well as [argocd-managed Kustomize](https://argoproj.github.io/argo-cd/user-guide/kustomize/) and [Helm charts](https://argoproj.github.io/argo-cd/user-guide/helm/).

# TL;DR
You can [find the plugin here](https://github.com/crumbhole/argocd-vault-replacer), complete with instructions and some samples so you can test the installation.

![argocd-vault-replacer-diagram.png]({{site.baseurl}}/images/2021-02-22/argocd-vault-replacer-diagram.png)

# Why?
GitOps is a great thing to have (or at least be aiming for)... ensuring all your infrastructure and configuration is stored as code, and ensuring that your Git repo is the source of truth for your environment. Argo CD Happily sits there and ensures that your environment(s) match the configuration in Git.

The painful reality of GitOps is that it becomes all too easy to store secrets in your Git repo alongside your code, combined with the distributed nature of Git, it can become quite easy to lose those secrets to a bad actor, or simply to more people in your organisation who really should have those secrets. One solution is to store your secrets in a secrets manager, but then extracting those secrets in a programatic way can become tricky.

# So how does this work?
I'm not going to go into super detail here, because the readme is relatively comprehensive already (and if it isn't, PRs are more than welcome!). Firstly you have to create a Kubernetes serviceAccount, and then give the serviceAccount permission to look at (some of your) secrets in Vault. [Hashicorp's own documentation](https://www.vaultproject.io/docs/auth/kubernetes) covers this well.

Then you append that serviceAccount to your installation of ArgoCD and install the plugin (using an init container).

Lastly, you need to modify your yaml (or yml, or Helm, or Kustomize scripts) to point it at the relevant path(s) and key(s) in vault that you wish to add to your code.

In the following example, we will populate a Kubernetes Secret with the key `secretkey` on the path `path/to/your/secret`. As we are using a Vault kv2 store, we must include ..`/data/`.. in our path. Kubernetes secrets are base64 encoded, so we add the modifier `|base64` and the plugin handles the rest.

``` yml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-vault-replacer-secret
data:
  sample-secret: <vault:path/data/to/your/secret~secretkey|base64>
type: Opaque
```

When Argo CD runs, it will pull your yaml from Git, find the secret at the given path and will merge the two together inside your cluster. The result is exactly what you'd expect, a nicely populated Kubernetes Secret.

# Try it out
If you're already using Argo CD and Vault, then this is really simple to set up and start using. Please do try it out, and issues and comments are more than welcome: https://github.com/crumbhole/argocd-vault-replacer