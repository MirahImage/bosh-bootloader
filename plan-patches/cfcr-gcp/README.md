# Patch: cfcr-gcp

Steps to deploy cfcr with bbl:

1. Follow the normal steps to bbl up with a patch
    ```
    mkdir banana-env && cd banana-env
    bbl plan --name banana-env
    cp -r bosh-bootloader/plan-patches/cfcr-gcp/. .
    bbl up
    eval "$(bbl print-env)"
    ```

1. `bosh upload-release https://storage.googleapis.com/kubo-public/kubo-release-latest.tgz`

1. `bosh upload-stemcell https://bosh.io/d/stemcells/bosh-google-kvm-ubuntu-trusty-go_agent`

1. export KD as your path to kubo-deployment so you can copy-paste from below if you so desire.
   be careful to check out the manifest that matches the kubo-release you downloaded above.
   ```
   git clone git@github.com:cloudfoundry-incubator/kubo-deployment.git
   export KD=$(pwd)/kubo-deployment
   ```

1. Deploy the cfcr manifest.
   ```
   bosh deploy -d cfcr ${KD}/manifests/cfcr.yml \
   -o ${KD}/manifests/ops-files/iaas/gcp/cloud-provider.yml \
   -o cfcr-ops.yml \
   -l <(bosh int cfcr-vars-template.yml -l <(bbl outputs))
   ```

   If you'd like to use compiled releases to speed up your deployment and worry a bit less about matching release+manifest versions check out [cfcr-compiled-deployment](https://github.com/starkandwayne/cfcr-compiled-deployment).
   ```
   export CFCRC=~/go/github.com/starkandwayne/cfcr-compiled-deployment
   bosh deploy -d cfcr ${CFCRC}/cfcr.yml \
   -o ${CFCRC}/ops-files/iaas/gcp/cloud-provider.yml \
   -o cfcr-ops.yml \
   -l <(bosh int cfcr-vars-template.yml -l <(bbl outputs))
   ```

1. Configure kubectl

   Then run the following to mix them together into kubectl-appropriate forms:
   ```
   export director_name=$(bosh int <(bbl outputs) --path=/director_name)
   export kubernetes_master_host=$(bosh int <(bbl outputs) --path=/master_lb_ip_address)
   export address="https://${kubernetes_master_host}:8443"
   export cluster_name="kubo:${director_name}:cfcr"
   export user_name="kubo:${director_name}:cfcr-admin"
   export context_name="kubo:${director_name}:cfcr"

   credhub login
   export admin_password=$(bosh int <(credhub get -n "${director_name}/cfcr/kubo-admin-password" --output-json) --path=/value)

   # right now, we don't support TLS verification of the kubernetes master, so we also don't need to run these commmands.
   # export tmp_ca_file="$(mktemp)"
   # bosh int <(credhub get -n "${director_name}/cfcr/tls-kubernetes" --output-json) --path=/value/ca > "${tmp_ca_file}"
   ```
   ```
   kubectl config set-cluster "${cluster_name}" --server="${address}" --insecure-skip-tls-verify=true
   kubectl config set-credentials "${user_name}" --token="${admin_password}"
   kubectl config set-context "${context_name}" --cluster="${cluster_name}" --user="${user_name}"
   kubectl config use-context "${context_name}"
   ```

 - `kubectl get pods`
 - create, scale, and expose apps with the kubernetes bootcamp docker image:
   ```
   kubectl run kubernetes-bootcamp --image=docker.io/jocatalin/kubernetes-bootcamp:v1 --port=8080
   kubectl get pods
   kubectl expose deployment/kubernetes-bootcamp --type="LoadBalancer"
   kubectl get services
   # get EXTERNAL-IP for kubernetes-bootcamp
   curl http://${EXTERNAL-IP}:8080
   ```

