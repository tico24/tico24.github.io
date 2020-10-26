---
published: true
---
So you've gone and set up Kubernetes backups... Nice one. You've probably used Velero if you haven't used something native to your cloud provider. Velero is a solid tool, admittedly with a few quirks, but it gets the job done and can be relied upon. But how can you be certain the backups worked? You could manually restore every now and then, but that's not very DevOpsy.. plus it's really boring. Here's how I did it.

First, I created a namespace called "backup-canary" and deployed a simple deployment containing an nginx pod with a service. This pod has a 1GB Persistent Volume mounted at /usr/share/nginx/html.

I then manually wrote a small index.html file inside the volume, containing a known phrase:

``` yml
// Get the name of the pod
kubectl -n backup-canary get pods -l=app=backup-canary -o jsonpath='{.items[-1:].metadata.name}'

// Write to the index.html file
kubectl exec -n backup-canary -it ${podName} -- bash -c "echo hello-world > /usr/share/nginx/html/index.html" y
```
If we were to put an ingress on the service (or did magic kubectl port forwarding), we would see a web page with "hello-world" emblazened on it.

I then allowed Velero to back this all up.

I chose to create a Jenkins job to then perform the restore test. Any CI tool should be up for the job, or you could do this natively in Kubernetes, but I wanted an outside tool to do the hard work.

Firstly I created a simple kubectl + velero CLI container:

    FROM dtzar/helm-kubectl:3.3.4
    RUN wget -nv -O /tmp/velero.tar.gz https://github.com/vmware-tanzu/velero/releases/download/v1.5.2/velero-v1.5.2-linux-amd64.tar.gz \
      && tar -xvf /tmp/velero.tar.gz --directory /tmp/ \
      && mv /tmp/velero-v1.5.2-linux-amd64/velero/ /usr/local/bin/ \
      && rm -rf /tmp/velero.*
      
Then I wrote a simple Jenkins pipline to perform the following steps:

1. Read the contents of the current index.html file and store as a variable.
2. Delete the "backup-canary" namespace.*
3. Find out the name of the latest successful backup.
4. Restore this backup to a new namespace ("restore-canary").
5. Test that the restored file matches what we expected.**
6. Assuming we are successful, I then delete the restored namespace.
7. Restore the backup-canary namespace and deployment.
8. Write a new phrase (I use a phrase containing the jenkins job build ID) to the index.html to be backed up next time.

* Velero will not restore a file, (even to a different namespace) if the original remains.

** I test this by using kubectl to connect to the restored pod, and then perform a curl against the service to get the value. This has the benefit of not just testing that the PV restored successfully, but also the pod and service restored well enough that the webserver is accessible.

![Velero-Restore-Pod.svg]({{site.baseurl}}/images/Velero-Restore-Pod.svg)

The jenkins job is then set to run regularly. Velero can produce Prometheus metrics if you set it up to do so, so restores (successes and failures) are recorded on a Grafana dashboard. If restores fail, a Prometheus alert is triggered and I can resolve the problem before I really need the backup.

![grafana-restores.png]({{site.baseurl}}/images/grafana-restores.png)
