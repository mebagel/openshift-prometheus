## Prometheus overview
![Prometheus components](diagrams/prometheus.png)

## Install Prometheus
https://docs.openshift.com/container-platform/3.11/install_config/prometheus_cluster_monitoring.html
Set variables in inventory (small cluster)
```
openshift_cluster_monitoring_operator_install=true
openshift_cluster_monitoring_operator_prometheus_storage_enabled=true
openshift_cluster_monitoring_operator_prometheus_storage_capacity=50Gi
openshift_cluster_monitoring_operator_prometheus_storage_class_name=[tbd]
openshift_cluster_monitoring_operator_alertmanager_storage_enabled=true
openshift_cluster_monitoring_operator_alertmanager_storage_capacity=2Gi
openshift_cluster_monitoring_operator_alertmanager_storage_class_name=[tbd]
openshift_cluster_monitoring_operator_alertmanager_config=[tbd]
```

Default retention
```
        - --storage.tsdb.retention=15d
```

Run the installer
```
ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/openshift-monitoring/config.yml
```

## Prepare Prometheus
Let Prometheus scrape service labels in different namespaces
```
oc adm policy add-cluster-role-to-user cluster-reader system:serviceaccount:openshift-monitoring:prometheus-k8s
oc delete pod -l app=prometheus
``` 

### Add custom ruleset and scrape endpoints
```
oc apply -f templates/prometheus-custom-rules.yaml -n openshift-monitoring
```

### Additional targets (etcd)
https://docs.openshift.com/container-platform/3.11/install_config/prometheus_cluster_monitoring.html

### Additional targets (router)
```
oc apply -f templates/servicemonitor-router.yaml
oc apply -f templates/clusterrole-router-metrics.yaml
oc adm policy add-cluster-role-to-user router-metrics system:serviceaccount:openshift-monitoring:prometheus-k8s
```

### Additional targets (logging)
```
oc apply -f templates/servicemonitor-elasticsearch.yaml
oc label svc logging-es-prometheus -n openshift-logging scrape=prometheus
oc apply -f templates/rolebinding_logging.yaml
```

## Alerting
Inventory:
```
openshift_cluster_monitoring_operator_alertmanager_config: |+
....
```

By hand
```
oc delete secret alertmanager-main
oc create secret generic alertmanager-main --from-file=templates/alertmanager.yaml
oc delete pod -l app=alertmanager
```

## Application Monitoring
#### Simple REST Example 
https://labs.consol.de/development/2018/01/19/openshift_application_monitoring.html
```
oc new-project rest-test
oc new-app -f https://raw.githubusercontent.com/ConSol/springboot-monitoring-example/master/templates/restservice_template.yaml -n rest-test
oc label svc restservice scrape="prometheus" --overwrite
oc apply -f templates/servicemonitor-application-rest-test.yaml -n openshift-monitoring
```

#### JMX Example
https://blog.openshift.com/enhanced-openshift-jboss-amq-container-image-for-production/
```
oc new-project amq-tests
oc new-build https://github.com/lbroudoux/openshift-cases.git --context-dir="/jboss-amq-custom/custom-amq" --name=custom-amq --strategy=docker --to=custom-jboss-amq-63
watch oc get pods
oc tag amq-tests/custom-jboss-amq-63:latest amq-tests/custom-jboss-amq-63:1.3
oc process -f https://raw.githubusercontent.com/lbroudoux/openshift-cases/master/jboss-amq-custom/custom-amq63-persistent-nosecret.yml     --param="APPLICATION_NAME=custom-broker"     --param="IMAGE_STREAM_NAMESPACE=amq-tests" | oc create -f -
watch oc get pods
oc edit dc custom-broker-amq 
oc label svc custom-broker-amq-tcp scrape=prometheus
oc apply -f templates/servicemonitor-application-amq-tests.yaml -n openshift-monitoring
```

## Additional configuration
### Add view role for developers
```
oc adm policy add-cluster-role-to-user cluster-monitoring-view [user]
```

### Add metrics reader sa to access Prometheus metrics
```
oc create sa prometheus-metrics-reader -n openshift-monitoring
oc adm policy add-cluster-role-to-user cluster-monitoring-view -z prometheus-metrics-reader -n openshift-monitoring
oc sa get-token prometheus-metrics-reader -n openshift-monitoring
```

### OSE running on ovs-networkpolicy SDN
Allow Prometheus to scrape your metrics endpoints. Create the network-policy.
```
oc create -f templates/networkpolicy.yaml
```

## Known Issues
- add blackbox exporter => https://github.com/coreos/prometheus-operator/issues/1201 not yet implemented
- Application Montoring scrapes always all endpoint ips -> Annotations not yet supported: https://github.com/coreos/prometheus-operator/issues/1547
