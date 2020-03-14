### Log Analytics

* Create a Log Analytics instance, then enable container insights

### Prometheus

* Install Prometheus
```
helm install prometheus --namespace monitoring stable/prometheus-operator
```

### Baseline
* Use the simulator to start generating a baseline and get a feel for the typical performance profile of your cluster

### Insurance App

* Deploy the app using the YAML file in the OpenHack guide. Modify the container registry to use the instance for your team
* The app appears to have a memory leak. Use the 