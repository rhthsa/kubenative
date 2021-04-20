# Kubernetes Application Best Practices 

<!-- TOC -->

- [Kubernetes Application Best Practices](#kubernetes-application-best-practices)
  - [Setup](#setup)
    - [Prerequisites](#prerequisites)
    - [Demo Application](#demo-application)
    - [Service Mesh](#service-mesh)
    - [Cluster Logging](#cluster-logging)
    - [Web Terminal](#web-terminal)
    - [Bad App start with Root](#bad-app-start-with-root)
  - [Demo](#demo)
    - [Inject Config Map](#inject-config-map)
      - [Environment](#environment)
      - [Mount ConfigMap to file](#mount-configmap-to-file)
    - [Resources Request/Limit](#resources-requestlimit)
    - [Health Check](#health-check)
    - [Pod Disruption Budget](#pod-disruption-budget)
    - [Application Monitoring](#application-monitoring)
    - [Cluster Logging](#cluster-logging-1)
    - [Circuit Breaker](#circuit-breaker)
    - [mTLS](#mtls)
    - [SCC & PSP](#scc--psp)

<!-- /TOC -->

## Setup
### Prerequisites
- Install following operators from Operator Hub
  - ElasticSearch
  - Jaeger
  - Kiali
  - OpenShift Service Mesh
  - Vertical Pod Autosacler

### Demo Application 
- Setup user workload monitoring
  
  ```bash
   oc apply -f  https://raw.githubusercontent.com/rhthsa/openshift-demo/main/manifests/user-workload-monitoring.yaml
   oc apply -f  https://raw.githubusercontent.com/rhthsa/openshift-demo/main/manifests/cluster-monitoring-config.yaml
  ```
  
- Label 2 workers node with *tier=backend*
  
  ```bash
  for node in $(oc get nodes | grep worker|awk '{print $1}'| head -n 2)
  do
    echo "Label tier=backend to $node"
    oc label node $node tier=backend
  done
  ```

- Create demo project and deploy demo app
  
  ```bash
  oc new-project demo
  curl -s https://gist.githubusercontent.com/voraviz/bf28bb1e41cae30e887f482299abd94f/raw/a5c79670f669cdbd563f481e9d3547588335b0c9/demo-app-with-node-selector.yaml \
  | oc apply -n demo -f -
  ```
- Create Service Monitoring

  ```bash
  curl -s https://raw.githubusercontent.com/rhthsa/openshift-demo/main/manifests/backend-service-monitor.yaml \ 
  | sed 's/backend/demo/g' \
  | oc apply -n demo -f -
  ```

- Create Alert Rule
- 
  ```bash
  curl -s https://raw.githubusercontent.com/rhthsa/openshift-demo/main/manifests/backend-custom-alert.yaml \
  | sed 's/project1/demo/' \
  | oc apply -f -
  ```

- Custom Grafana
   ```bash
   oc new-project application-monitor \
   --display-name="Custom Grafana" \
   --description="Custom Grafana"
   ```
- Install Grafana Operator to project application-monitor
- Create CRD
  ```bash 
   oc apply -f https://raw.githubusercontent.com/rhthsa/openshift-demo/main/manifests/grafana.yaml \
   -n application-monitor
   watch oc get pods -n application-monitor
   oc adm policy add-cluster-role-to-user cluster-monitoring-view \
   -z grafana-serviceaccount -n application-monitor
   TOKEN=$(oc serviceaccounts get-token grafana-serviceaccount -n application-monitor)
   curl -s https://raw.githubusercontent.com/rhthsa/openshift-demo/main/manifests/grafana-datasource.yaml \
   | sed 's/Bearer .*/Bearer '"$TOKEN""'"'/' \
   | oc apply -n application-monitor -f - 
   oc apply -f https://raw.githubusercontent.com/rhthsa/openshift-demo/main/manifests/grafana-dashboard.yaml \
   -n application-monitor 
   echo "Grafana URL => https://$(oc get route grafana-route -o jsonpath='{.spec.host}' -n application-monitor)"
   ```
### Service Mesh
- Setup Service Mesh

  ```bash
  oc new-project istio-system
  oc new-project project1 
  oc create -f \
  https://raw.githubusercontent.com/rhthsa/openshift-demo/main/manifests/smcp.yaml \
  -n istio-system
  watch oc get smcp/basic-install -n istio-system
  sleep 5
  oc apply -f \
  https://raw.githubusercontent.com/rhthsa/openshift-demo/main/manifests/smmr.yaml \
  -n istio-system
  oc describe smmr/default -n istio-system | grep -A2 Spec: 
  ```

- Deploy application for test Circuit Breaker
  ```bash
  curl -s https://raw.githubusercontent.com/rhthsa/openshift-demo/main/manifests/frontend.yaml \
  | sed 's/frontend-v1/frontend/' \
  | oc apply -n project1 -f -
  oc patch deployment/frontend \
  -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' \
  -n project1
  oc apply -f https://raw.githubusercontent.com/rhthsa/openshift-demo/main/manifests/backend.yaml \
  -n project1
  oc delete deployment frontend-v2 -n project1
  oc delete svc frontend-v2 -n project1
  oc delete deployment backend-v2 -n project1 
  oc set env deployment/frontend BACKEND_URL=http://backend:8080/ -n project1
  oc annotate deployment frontend \
  'app.openshift.io/connects-to=[{"apiVersion":"apps/v1","kind":"Deployment","name":"backend-v1"},{"apiVersion":"apps/v1","kind":"Deployment","name":"backend-v2"}]' \
  -n project1
  oc delete route frontend -n project1
  oc scale deployment backend-v1 --replicas=3 -n project1
  oc apply -f \
  https://raw.githubusercontent.com/rhthsa/openshift-demo/main/manifests/backend-destination-rule-v1-only.yaml \
  -n project1
  oc apply -f \
  https://raw.githubusercontent.com/rhthsa/openshift-demo/main/manifests/backend-virtual-service.yaml \
  -n project1
  oc apply -f https://raw.githubusercontent.com/rhthsa/openshift-demo/main/manifests/frontend-destination-rule-v1-only.yaml -n project1
  SUBDOMAIN=$(oc whoami --show-console|awk -F'apps.' '{print $2}')
  curl -s https://raw.githubusercontent.com/rhthsa/openshift-demo/main/manifests/frontend-virtual-service.yaml | sed 's/SUBDOMAIN/'$SUBDOMAIN'/'|oc apply -n project1 -f -
  SUBDOMAIN=$(oc whoami --show-console|awk -F'apps.' '{print $2}')
  curl -s https://raw.githubusercontent.com/rhthsa/openshift-demo/main/manifests/frontend-gateway.yaml | sed 's/SUBDOMAIN/'$SUBDOMAIN'/'|oc apply -n istio-system -f -
  oc get svc -n project1
  oc get deployment -n project1
  sleep 3
  watch oc get pods -n project1
  FRONTEND_URL=$( oc get route $(oc get route -n istio-system|grep istio-system-frontend|awk '{print $1}') -o jsonpath='{.spec.host}' -n istio-system)
  printf "URL: http://%s" $FRONTEND_URL
  curl -v http://$FRONTEND_URL
  ```
- Set user1 to admin of demo,project1 and istio-system
  
  ```bash
  oc adm policy add-role-to-user admin user1 -n demo
  oc adm policy add-role-to-user admin user1 -n project1
  oc adm policy add-role-to-user admin user1 -n istio-system
  oc adm policy add-role-to-user admin user1 -n application-monitor
  ``` 
- Set user1 for monitor

  ```bash
  oc adm policy add-role-to-user  monitoring-edit user1 -n demo
  oc adm policy add-role-to-user  monitoring-rules-view user1 -n demo
  oc adm policy add-role-to-user  monitoring-rules-edit user1 -n demo
  ```
### Cluster Logging
- Install ElasticSerach and Cluster Logging Operator
- Create cluster logging
  ```bash
  oc apply -f cluster-logging.yaml
  watch oc get deployment -n openshift-logging
  oc get pods -n openshift-logging
  ``` 

### Web Terminal
- Install Web Terminal Operator
- User user1 to create project user1-terminal
- Start Web Terminal in project user1-terminal

### Bad App start with Root
- Clone and build container that start with root [hello-world](https://github.com/root-project/docker-examples.git)
- Import into internal registry
  
  ```bash
  oc project demo
  oc import-image --confirm quay.io/voravitl/run-with-root:latest
  ```
  
## Demo

### Inject Config Map
#### Environment
- Dev Console, use project Demo
- Call demo app from Dev Console
- Show source code of backend quarkus
- Show environments already set at pod
- Edit deployment, YAML
- Create Config Map from Dev Console
  
  ```yaml
  data:
    APP_BACKEND: 'https://httpbin.org/status/201'
  ```
- Edit Deployment => Environment
  - Delete Single values (env)
  - Select All values from existing config maps
  - Save
- Show deployment, YAML => Search envFrom
- Call demo app from Dev Console

#### Mount ConfigMap to file
- Create ConfigMap from file
  ```bash
  curl -s \
  https://gitlab.com/ocp-demo/backend_quarkus/-/raw/master/code/config/application.properties \
  > application.properties
  
  oc create configmap example-2 \
  --from-file=application.properties \
  -n demo
  ```
- Set ConfigMap
  ```bash
    oc set volume deployment/demo --add --name=config \
    --mount-path=/work/config/application.properties \
    --sub-path=application.properties \
    --configmap-name=example-2 \
    -n demo
  ```
- Update configmap
  
  ```bash
   oc create configmap example-2 \
  --from-file=application.properties \
  -o yaml  -n demo \
  --dry-run | \
  oc replace -f -
  ```
  Remark: For Quarkus, environment variables take precedance of config file
  
- Rollback to original version
  
  ```bash
  oc rollout history deployment/demo
  # Back 2 Revision
  oc rollout undo deployment/demo --to-revision=1
  ```
  
### Resources Request/Limit
- Show request/limit in deployment
- Show class by
  ```bash
  kubectl pod
  kubectl describe pod <pod name>
  # Check for QoS
  ```
- Set request = limit by Dev Console
- Show class agin
- Delete request/limit
- Describe pod that show Limits, Request 
- Admin Console-> Project Demo-> Limit Ranges

### Health Check
- Developer Console, Project Demo, Click Donut
- Terminal
- 
  ```bash
  curl -L http://localhost:8080/health/live
  curl -L http://localhost:8080/health/ready
  ```
- Add Health Checks with Dev Console
- Show Deployment YAML
- Terminal -> set not_ready, check service that pod is removed from service
- Terminal -> set stop, check that pod is restarted
- Sample CLI
  
  ```bash
  oc set probe deployment/demo --readiness --get-url=http://:8080/health/ready --initial-delay-seconds=5 --failure-threshold=1 --period-seconds=3 --timeout-seconds=5 -n demo
  oc set probe deployment/demo --liveness --get-url=http://:8080/health/live --initial-delay-seconds=5 --failure-threshold=1 --period-seconds=5 --timeout-seconds=5  -n demo  
  ```
  
### Pod Disruption Budget
- Scale demo pod to 10
- Show pod with worker node
  ```bash
  kubectl get pod -l app=demo -o wide --field-selector status.phase=Running
  ```
- Create PodDisruptionBudget
  ```bash
  curl -s https://gist.githubusercontent.com/voraviz/49b5b97c615da4661dd2dfa22d0aad7f/raw/992e5b48ecdd2289b0aba7f969d447902d695718/demo-pdb.yaml | oc apply -f -
  ```
- Check status
  ```bash
  kubectl get poddisruptionbudget/demo-pdb
  ```
- Run script to show pod with Running state
  ```bash
   while [ 1 ];
   do
      NUM=$(oc get pods -l app=demo -n demo |grep Running| wc -l| sed 's/^[ \t]*//')
      printf "Running pods=%s" $NUM
      sleep 2
      clear
  done
  ```
- Drain node

  ```bash
  kubectl get pods -l app=demo -o wide
  kubectl cordon ip-10-0-134-53.ap-southeast-1.compute.internal
  kubectl drain ip-10-0-134-53.ap-southeast-1.compute.internal --pod-selector=app=demo
  ```
  
  with **oc**

  ```bash
  oc adm cordon <node>
  oc adm drain <node>
  ```
  
- check that number of available pod is not less than maxUnavailable
- Check that all pods run on one worker node
  
  ```bash
  kubectl get pods -l app=demo -o wide
  oc get pods -o wide | awk '{print $(NF-2)}' | sort -u
  ```
  
- Set node back
  
  ```bash
  kubectl uncordon ip-10-0-134-53.ap-southeast-1.compute.internal
  ```
  
### Application Monitoring
- Show backend quarkus code for MicroProfile Metrics
- Dev Console -> Monitoring -> Custom -> ...Heap
- Loop
  ```bash
  FRONTEND_URL=https://$(oc get route demo -n demo -o jsonpath='{.spec.host}')
  while [ 1 ];
  do
    curl $FRONTEND_URL
    printf "\n"
    sleep .2
  done
  ```
- Test Latency Alert with Dev Console or CLI
  
  ```bash
  oc set env deployment/backend-v1 APP_BACKEND=https://httpbin.org/delay/6 -n project1
  ```

- Test Concurrent Alert
  
  ```bash
  curl -s https://raw.githubusercontent.com/rhthsa/openshift-demo/main/manifests/load-test-k6.js > load-test-k6.js
  DEMO_URL=https://$(oc get route demo -n demo -o jsonpath='{.spec.host}')
  oc run load-test -n demo -i \
  --image=loadimpact/k6 --rm=true --restart=Never \
  --  run -  < load-test-k6.js \
  -e URL=$DEMO_URL -e THREADS=40 -e DURATION=2m -e RAMPUP=30s -e RAMPDOWN=30s
  ```  
### Cluster Logging
- Show log in code
- Log in Dev Console
- Check Pod's log
- Open Kibana from link *"Show in Kibana"*

### Circuit Breaker
- Switch to project project1
- Show that backend has 3 pods
- Show kiali
- Create bash function
  ```bash
  function loop_frontend(){
  FRONTEND_ISTIO_ROUTE=$(oc get route -n istio-system|grep istio-system-frontend-gateway |awk '{print $2}')
  COUNT=0
  MAX=$1
  while [ $COUNT -lt $MAX ];
  do
    curl -s http://$FRONTEND_ISTIO_ROUTE | awk -F',' '{print $5 "=>" $6}'
    COUNT=$(expr $COUNT + 1 )
  done
  }
  ```
- Stop one pod with /not_ready to set pod to return 503
- Show that all requests are 200 OK
- Apply CB
  ```yaml
  spec:
    host: backend.project1.svc.cluster.local
  trafficPolicy:
    connectionPool:
      http: {}
      tcp: {}
    loadBalancer:
      simple: ROUND_ROBIN
    outlierDetection:
      baseEjectionTime: 15m
      consecutiveErrors: 1
      interval: 15m
      maxEjectionPercent: 100
  ```
- Set one pod with /stop to setup to return 504
- Show kiali

### mTLS
- Configure Destination Rule
  ```yaml
  spec:
    host: backend.project1.svc.cluster.local
    trafficPolicy:
      tls:
        mode: ISTIO_MUTUAL
  ```
- Check Kiali Graph with enable show security
- cURL from pod in demo project
- Required mTLS
  ```bash
  curl -s https://raw.githubusercontent.com/voraviz/openshift-service-mesh-ingress-mtls/main/config/backend-peer-authentication.yaml | sed 's/data-plane/project1/' | oc apply -f -
  ``` 
- cURL from pod in demo project again

### SCC & PSP
- Create new project and deploy from internal container registry
- Pod will crash loop backoff
- Use command to check
  ```bash
  oc status --suggest
  ```