### build

```
export PROJECT=nf-demo-gkee
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT} --format="value(projectNumber)")
gcloud config set project ${PROJECT}

mkdir -p ${HOME}/edge-to-mesh-multi-region
cd ${HOME}/edge-to-mesh-multi-region
export WORKDIR=`pwd`

touch edge2mesh_mr_kubeconfig
export KUBECONFIG=${WORKDIR}/edge2mesh_mr_kubeconfig

export CLUSTER_1_NAME=autopilot-demo-01
export CLUSTER_2_NAME=autopilot-demo-02
export CLUSTER_1_REGION=us-central1
export CLUSTER_2_REGION=us-east4
export PUBLIC_ENDPOINT=frontend.endpoints.${PROJECT}.cloud.goog

gcloud services enable \
  container.googleapis.com \
  mesh.googleapis.com \
  gkehub.googleapis.com \
  multiclusterservicediscovery.googleapis.com \
  multiclusteringress.googleapis.com \
  trafficdirector.googleapis.com \
  certificatemanager.googleapis.com

gcloud container clusters create-auto --async \
${CLUSTER_1_NAME} --region ${CLUSTER_1_REGION} \
--release-channel rapid --labels mesh_id=proj-${PROJECT_NUMBER} \
--enable-private-nodes --enable-fleet

gcloud container clusters create-auto \
${CLUSTER_2_NAME} --region ${CLUSTER_2_REGION} \
--release-channel rapid --labels mesh_id=proj-${PROJECT_NUMBER} \
--enable-private-nodes --enable-fleet

gcloud container clusters list

gcloud container clusters get-credentials ${CLUSTER_1_NAME} \
    --region ${CLUSTER_1_REGION}

kubectl config rename-context gke_${PROJECT}_${CLUSTER_1_REGION}_${CLUSTER_1_NAME} ${CLUSTER_1_NAME}
kubectl config rename-context gke_${PROJECT}_${CLUSTER_2_REGION}_${CLUSTER_2_NAME} ${CLUSTER_2_NAME}

# just use the e2m guide for the rest
```