# mesh-demo-march-2024

### setup

```
# env vars
export PROJECT=e2m-private-test-01
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT} --format="value(projectNumber)")
gcloud config set project ${PROJECT}

export CLUSTER_1_NAME=edge-to-mesh-01
export CLUSTER_2_NAME=edge-to-mesh-02
export CLUSTER_1_REGION=us-central1
export CLUSTER_2_REGION=us-east4
export PUBLIC_ENDPOINT=frontend.endpoints.${PROJECT}.cloud.goog

# rename contexts 
kubectl config rename-context gke_${PROJECT}_${CLUSTER_1_REGION}_${CLUSTER_1_NAME} ${CLUSTER_1_NAME}
kubectl config rename-context gke_${PROJECT}_${CLUSTER_2_REGION}_${CLUSTER_2_NAME} ${CLUSTER_2_NAME}

# create gsa for tracing 
# create GSA for writing traces 
gcloud iam service-accounts create whereami-tracer \
    --project=${PROJECT}

gcloud projects add-iam-policy-binding ${PROJECT} \
    --member "serviceAccount:whereami-tracer@${PROJECT}.iam.gserviceaccount.com" \
    --role "roles/cloudtrace.agent"

# map to KSAs
gcloud iam service-accounts add-iam-policy-binding whereami-tracer@${PROJECT}.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:${PROJECT}.svc.id.goog[frontend/whereami-frontend]"

gcloud iam service-accounts add-iam-policy-binding whereami-tracer@${PROJECT}.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:${PROJECT}.svc.id.goog[backend/whereami-backend]"

# editing KSAs to add annotation by adding iam.gke.io/gcp-service-account: whereami-tracer@e2m-private-test-01.iam.gserviceaccount.com

# enable access logging and tracing
cat <<EOF | kubectl --context edge-to-mesh-01 apply -f -
apiVersion: v1
data:
  mesh: |-
    accessLogFile: /dev/stdout
    defaultConfig:
      tracing:
        stackdriver: {}
kind: ConfigMap
metadata:
  name: istio-asm-managed-rapid
  namespace: istio-system
EOF

cat <<EOF | kubectl --context edge-to-mesh-02 apply -f -
apiVersion: v1
data:
  mesh: |-
    accessLogFile: /dev/stdout
    defaultConfig:
      tracing:
        stackdriver: {}
kind: ConfigMap
metadata:
  name: istio-asm-managed-rapid
  namespace: istio-system
EOF

# restart backend and frontend 
for CONTEXT in ${CLUSTER_1_NAME} ${CLUSTER_2_NAME}
do 
    kubectl --context=$CONTEXT -n backend rollout restart deployment whereami-backend
done

for CONTEXT in ${CLUSTER_1_NAME} ${CLUSTER_2_NAME}
do 
    kubectl --context=$CONTEXT -n frontend rollout restart deployment whereami-frontend
done

# apply locality 
for CONTEXT in ${CLUSTER_1_NAME} ${CLUSTER_2_NAME}
do 
    kubectl --context=$CONTEXT apply -f locality/
done

# apply all authz stuff
for CONTEXT in ${CLUSTER_1_NAME} ${CLUSTER_2_NAME}
do 
    kubectl --context=$CONTEXT apply -f authz/
done

# test removal of backend authz policy
for CONTEXT in ${CLUSTER_1_NAME} ${CLUSTER_2_NAME}
do 
    kubectl --context=$CONTEXT delete -f authz/backend.yaml
done

# test explicit denial
for CONTEXT in ${CLUSTER_1_NAME} ${CLUSTER_2_NAME}
do 
    kubectl --context=$CONTEXT apply -f authz-explicit-deny/
done

# remove explicit denial
for CONTEXT in ${CLUSTER_1_NAME} ${CLUSTER_2_NAME}
do 
    kubectl --context=$CONTEXT delete -f authz-explicit-deny/
done

# can check with log filter for labels.response_details = "AuthzDenied"
# check metrics with sum(rate(istio_io:service_server_request_count{monitored_resource="istio_canonical_service",service_authentication_policy="MUTUAL_TLS"}[${__interval}]))

```

### reset environment

```
for CONTEXT in ${CLUSTER_1_NAME} ${CLUSTER_2_NAME}
do 
    kubectl --context=$CONTEXT delete -f traffic-splitting/
    kubectl --context=$CONTEXT delete -f authz-explicit-deny/
    kubectl --context=$CONTEXT delete -f locality/
    kubectl --context=$CONTEXT delete -f authz/
    kubectl --context=$CONTEXT delete -f peer-auth/
    kubectl --context=$CONTEXT -n backend apply -f traffic-splitting/subsets-dr.yaml
    kubectl --context=$CONTEXT -n backend apply -f traffic-splitting/vs-0.yaml
done

```

### edit CM for metadata

- edit CM
- edit deployment

### test from VM

```
hey -n 99999999999999 -c 2 -q 20 https://frontend.endpoints.e2m-private-test-01.cloud.goog
```

### test from Cloud Shell

```
watch -n 0.1 'curl -s https://frontend.endpoints.e2m-private-test-01.cloud.goog | jq'
```

### create test pod for auth 

```
kubectl run whereami-test --image=us-docker.pkg.dev/google-samples/containers/gke/whereami:v1.2.22

kubectl exec --stdin --tty whereami-test -- /bin/sh

while true; do curl http://whereami-frontend.frontend; sleep 0.01; done
```

### create strict mTLS policy

```
# apply
for CONTEXT in ${CLUSTER_1_NAME} ${CLUSTER_2_NAME}
do 
    kubectl --context=$CONTEXT apply -f peer-auth/
done

# remove 
for CONTEXT in ${CLUSTER_1_NAME} ${CLUSTER_2_NAME}
do 
    kubectl --context=$CONTEXT delete -f peer-auth/
done
```

### authz journey

```
for CONTEXT in ${CLUSTER_1_NAME} ${CLUSTER_2_NAME}
do 
    kubectl --context=$CONTEXT apply -f authz/asm-ingress.yaml
    kubectl --context=$CONTEXT apply -f authz/frontend.yaml
    kubectl --context=$CONTEXT apply -f authz/allow-none.yaml
done

for CONTEXT in ${CLUSTER_1_NAME} ${CLUSTER_2_NAME}
do 
    kubectl --context=$CONTEXT apply -f authz/backend.yaml
done
```

### apply locality

```
for CONTEXT in ${CLUSTER_1_NAME} ${CLUSTER_2_NAME}
do 
    kubectl --context=$CONTEXT apply -f locality/
done
```

### test backend service failover

```
# scale to zero
for CONTEXT in ${CLUSTER_1_NAME} 
do 
    kubectl --context=$CONTEXT -n backend scale --replicas=0 deployment/whereami-backend
done

# scale back up 
for CONTEXT in ${CLUSTER_1_NAME} 
do 
    kubectl --context=$CONTEXT -n backend scale --replicas=3 deployment/whereami-backend
done
```

### egress demo 