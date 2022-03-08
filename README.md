
###### GLOUD CONNECTOR, SYNC
# DOC: https://cloud.google.com/community/tutorials/infra-automation-kcc-config-sync-gatekeeper

# ENV's:
export HOST_PROJECT_ID=host-343500 
export DEV_PROJECT_ID=dev-proj-343500 
export PROD_PROJECT_ID=prod-proj-343500 
export CLUSTER_NAME=cc-host-cluster
export ZONE=australia-southeast1-c

# Enable API
gcloud services enable container.googleapis.com --project $HOST_PROJECT_ID

# Create GKE
gcloud container clusters create ${CLUSTER_NAME} --project=$HOST_PROJECT_ID --zone=${ZONE} --machine-type=e2-standard-4 \
  --workload-pool=${HOST_PROJECT_ID}.svc.id.goog

# Get Credentials
gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE --project $HOST_PROJECT_ID

# Download and install the Config Connector operator
gsutil cp gs://configconnector-operator/latest/release-bundle.tar.gz release-bundle.tar.gz
tar zxvf release-bundle.tar.gz
kubectl apply -f operator-system/configconnector-operator.yaml

# Create a Config Connector configuration to run in namespaced mode
cat>configconnector.yaml<<EOF
apiVersion: core.cnrm.cloud.google.com/v1beta1
kind: ConfigConnector
metadata:
  name: configconnector.core.cnrm.cloud.google.com
spec:
 mode: namespaced
EOF
kubectl apply -f configconnector.yaml

# Create NS, Annotation, IAM service account, binding, permissions,etc.
kubectl create namespace cc-tutorial-dev
kubectl create namespace cc-tutorial-prod
kubectl annotate namespace cc-tutorial-dev cnrm.cloud.google.com/project-id=$DEV_PROJECT_ID
kubectl annotate namespace cc-tutorial-prod cnrm.cloud.google.com/project-id=$PROD_PROJECT_ID
gcloud iam service-accounts create cc-tutorial-dev --project=$HOST_PROJECT_ID
gcloud iam service-accounts create cc-tutorial-prod --project=$HOST_PROJECT_ID
gcloud projects add-iam-policy-binding $DEV_PROJECT_ID \
  --member="serviceAccount:cc-tutorial-dev@${HOST_PROJECT_ID}.iam.gserviceaccount.com" --role="roles/editor" 
gcloud projects add-iam-policy-binding $PROD_PROJECT_ID \
  --member="serviceAccount:cc-tutorial-prod@${HOST_PROJECT_ID}.iam.gserviceaccount.com" --role="roles/editor"
gcloud iam service-accounts add-iam-policy-binding cc-tutorial-dev@${HOST_PROJECT_ID}.iam.gserviceaccount.com \
  --member="serviceAccount:${HOST_PROJECT_ID}.svc.id.goog[cnrm-system/cnrm-controller-manager-cc-tutorial-dev]" \
  --role="roles/iam.workloadIdentityUser" \
  --project=$HOST_PROJECT_ID
gcloud iam service-accounts add-iam-policy-binding cc-tutorial-prod@${HOST_PROJECT_ID}.iam.gserviceaccount.com \
  --member="serviceAccount:${HOST_PROJECT_ID}.svc.id.goog[cnrm-system/cnrm-controller-manager-cc-tutorial-prod]" \
  --role="roles/iam.workloadIdentityUser" \
  --project=${HOST_PROJECT_ID}
gcloud projects add-iam-policy-binding $HOST_PROJECT_ID \
  --member="serviceAccount:cc-tutorial-dev@${HOST_PROJECT_ID}.iam.gserviceaccount.com" --role="roles/monitoring.metricWriter"
gcloud projects add-iam-policy-binding $HOST_PROJECT_ID \
  --member="serviceAccount:cc-tutorial-prod@${HOST_PROJECT_ID}.iam.gserviceaccount.com" --role="roles/monitoring.metricWriter"

# Create Config Connector context
cat <<EOF > configconnectorcontext.yaml
apiVersion: core.cnrm.cloud.google.com/v1beta1
kind: ConfigConnectorContext
metadata:
  name: configconnectorcontext.core.cnrm.cloud.google.com
  namespace: cc-tutorial-dev
spec:
  googleServiceAccount: "cc-tutorial-dev@${HOST_PROJECT_ID}.iam.gserviceaccount.com"
---
apiVersion: core.cnrm.cloud.google.com/v1beta1
kind: ConfigConnectorContext
metadata:
  name: configconnectorcontext.core.cnrm.cloud.google.com
  namespace: cc-tutorial-prod
spec:
  googleServiceAccount: "cc-tutorial-prod@${HOST_PROJECT_ID}.iam.gserviceaccount.com"
EOF
kubectl apply -f configconnectorcontext.yaml
kubectl wait -n cnrm-system --for=condition=Ready pod --all

#### Config Sync
gsutil cp gs://config-management-release/released/latest/config-sync-operator.yaml config-sync-operator.yaml
kubectl apply -f config-sync-operator.yaml
kubectl create secret generic git-creds \
  --namespace=config-management-system \
  --from-file=ssh=../.ssh/id_rsa # ~ is not recognised here.

gcloud components install nomos
ln -s /usr/local/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/bin/nomos /usr/local/bin/nomos

nomos init --path=sync
mkdir sync/namespaces/cc-tutorial-dev
mkdir sync/namespaces/cc-tutorial-prod
cat>sync/namespaces/cc-tutorial-dev/namespace.yaml<<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: cc-tutorial-dev
EOF
cat>sync/namespaces/cc-tutorial-prod/namespace.yaml<<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: cc-tutorial-prod
EOF
cd ..

export GIT_REPOSITORY_URL=git@github.com:siaomingjeng/gcloud.git
export BRANCH_NAME=main
export DIRECTORY_NAME=sync

cat>config-management.yaml<<EOF
apiVersion: configmanagement.gke.io/v1
kind: ConfigManagement
metadata:
  name: config-management
spec:
  clusterName: cc-host-cluster
  git:
    syncRepo: $GIT_REPOSITORY_URL
    syncBranch: $BRANCH_NAME
    secretType: ssh
    policyDir: $DIRECTORY_NAME
EOF
kubectl apply -f config-management.yaml

cat>sync/namespaces/cc-tutorial-dev/storagebucket.yaml<<EOF
apiVersion: storage.cnrm.cloud.google.com/v1beta1
kind: StorageBucket
metadata:
  annotations:
    cnrm.cloud.google.com/force-destroy: "false"
  name: cc-tutorial-bucket-dev
spec:
  bucketPolicyOnly: true
  lifecycleRule:
    - action:
        type: Delete
      condition:
        age: 7
  versioning:
    enabled: true
EOF

#### Policy Enforcement
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.3/deploy/gatekeeper.yaml
kubectl -n gatekeeper-system describe svc gatekeeper-webhook-service

cat>policies.yml<<EOF
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: pubsubrequiredlabels
spec:
  crd:
    spec:
      names:
        kind: PubSubRequiredLabels
      validation:
        openAPIV3Schema:
          properties:
            labels:
              type: array
              items: 
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package pubsubrequiredlabels

        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          apiVersion := input.review.object.apiVersion
          isKccType := contains(apiVersion, "cnrm.cloud.google.com")
          missing := required - provided
          isKccType; count(missing) > 0
          msg := sprintf("you must provide labels: %v", [missing])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: PubSubRequiredLabels
metadata:
  name: must-contains-labels
spec:
  match:
    kinds:
    - apiGroups: ["pubsub.cnrm.cloud.google.com"]
      kinds: ["PubSubTopic"]
  parameters:
    labels: ["env", "owner", "location"]
EOF

# Show Error
cat>sync/namespaces/cc-tutorial-prod/pubsub.yaml<<EOF
apiVersion: pubsub.cnrm.cloud.google.com/v1beta1
kind: PubSubTopic
metadata:
  labels:
    env: prod
    location: australia-southeast1
  name: pubsubtopic-sample
EOF
nomos status

# Correct Error
cat>sync/namespaces/cc-tutorial-prod/pubsub.yaml<<EOF
apiVersion: pubsub.cnrm.cloud.google.com/v1beta1
kind: PubSubTopic
metadata:
  labels:
    env: prod
    location: us-east4
    owner: john-doe
  name: pubsubtopic-sample
EOF

















