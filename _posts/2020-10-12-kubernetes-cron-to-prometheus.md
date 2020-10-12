---
published: true
---
I recently had a requirement to execute a small python script inside a cron job. This script produced a series of metric values that we ultimately wanted to be able to represent on a graph and alert on in the future.

I didn't want to waste a whole EC2 instance on running a small script every hour and I'm not particularly a fan of vendor-lockin so wanted to avoid the serverless route. So as you've probably guessed from the title, I decided to use Kubernetes to manage the cron jobs.

We have Prometheus and Grafana already set up in our cluster, so all I had to do was to work out how to push data from our ephemeral cron pod into Prometheus.

---
## If you don't already have Prometheus and Grafana installed, a very brief interlude:

- I installed Prometheus using [this particular helm chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack). There are a lot of 'competing' charts out there and some seem to have been deprecated, so it was a bit of a minefield to try and source one that I liked.
- The prometheus helm does come with a Grafana instance, but I didn't like certain aspects of inflexibility that came with it. So [I installed Grafana](https://github.com/grafana/helm-charts/tree/main/charts/grafana) in its own namespace using this helm and configured it to use Prometheus as a data source.
---

Prometheus uses a pull method of collecting data. It scrapes data from defined service endpoints (or pods if you set it up to do so). This isn't any use to us. At its slowest, our cron job takes 3 seconds to spin up and execute, and then it destroys itself... this would mean we'd have to turn the Prometheus scrape interval up to 11 to be able to catch the pod while its running, and even then we can't guarantee that Prometheus will collect the generated data in time. Luckily, I'm not the first to encounter this problem.

## Enter Prometheus Pushgateway
The Pushgateway is an additional deployment that acts as a middle man between ephemeral pods (such as our cron job) and Prometheus. Jobs can push data to the Pushgateway which holds it for Prometheus to scrape in the way it knows how.

The Helm Chart for the Pushgateway [can be found here](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-pushgateway). Installation is pretty straightforward and the values.yml file is well documented.

For testing purposes, I would recommend you set up the ingress (or be prepared to forward the service locally so you can access the UI).

After installing the Pushgateway, we will need to tell Prometheus that it exists. This is as simple as adding an additionalServiceMonitor to the Prometheus helm chart and performing an update. Here's a small example with a bit of annotation:

    additionalServiceMonitors:
      - name: pushgateway  # Generic name so you can identify it.
        selector:
          matchLabels:
            app: prometheus-pushgateway  # Match with any service where the label key is 'app' and the value is 'prometheus-pushgateway'.
        namespaceSelector:
          matchNames:
            - prometheus # Only match with services in this namespace.
        endpoints:
          - port: http # Port name in the service.
            interval: 10s # How often to scrape.

## The Python bit
We need to tweak our cron job slightly so it knows how to push the data to the push gateway. I'm not a Pythonmonger, so this bit probably took me longer than it should have done.

The official docs are [here](https://github.com/prometheus/client_python), but I found that the example didn't actually work (YMMV). This didn't matter too much as I already had a python script that generated values, I just needed to punt them somewhere.

The key takeaway was that I needed to use a Gauge in my instance as my values may go up or down. So I just had to add a couple of import lines:

    import prometheus_client as prom
    from prometheus_client import CollectorRegistry, Gauge, push_to_gateway
    
... and then push the values (taken from a dictionary) to the gateway:

      # Push to Prometheus Gateway
      if __name__ == '__main__':
        registry = CollectorRegistry()
        g = Gauge('identifying_name_of_my_metric', 'Metric description to help humans', registry=registry)
        g.set(a_dictionary["metric"])
        push_to_gateway('servicename.namespace.svc.cluster.local:9091', job='Demo-job', registry=registry)
      
'servicename.namespace.svc.cluster.local' is 'the name of the pushgateway service'.'the name of the namespace the service is in'.svc.cluster.local. This will use Kubernetes' internal DNS to route to the correct service, so nothing needs to be exposed.

Once complete you should be able to see your data if you look firstly at your pushgateway url through a browser, and then at your prometheus instance.

## Making the container reusable
I envisage that this won't be our only Python cron job, so I wanted to make something that could be reused with ease.

So the python container is extremely lightweight, just python plus dependencies. Here's the dockerfile:

    FROM python:3-alpine
    RUN pip install requests prometheus_client

From there, my kubernetes yaml looks a little like this:

    apiVersion: batch/v1beta1
    kind: CronJob
    metadata:
      name: demo-cronjob
    spec:
      concurrencyPolicy: Forbid
      jobTemplate:
        spec:
          template:
            metadata:
              labels:
                app: demo-cronjob
            spec:
              containers:
              - name: demo-cronjob
                image: location-of-our-container
                imagePullPolicy: Always
                command:
                - python
                args:
                - /tmp/demo.py
                volumeMounts:
                  - name: scripts-mount
                    mountPath: /tmp
              volumes:
                - name: scripts-mount
                  configMap:
                    name: scripts-mount
              restartPolicy: Never
      schedule: '*/5 * * * *'
      successfulJobsHistoryLimit: 3
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: scripts-mount
    data:
      demo.py: |-
        import requests
        import prometheus_client as prom
        from prometheus_client import CollectorRegistry, Gauge, push_to_gateway
        
    	...
        
        # Push to Prometheus Gateway
        if __name__ == '__main__':
          registry = CollectorRegistry()
          g = Gauge('identifying_name_of_my_metric', 'Metric description to help humans', registry=registry)
          g.set(a_dictionary["metric"])
          push_to_gateway('servicename.namespace.svc.cluster.local:9091', job='Demo-job', registry=registry)
          
This allows me to add additional python scripts easily by appending the configmap and just adding additional cron jobs as required.
