
## FIRST TASK GET THE CODE.
#  Create your secrets file (one-time setup — never committed to git)

All the playbooks use variables that are declared in the global-var-secrets.txt file located in this project. 
You have to create this first off. 

```
cp global-var-secrets.example global-var-secrets.txt
```
Presently it is pointing to my public github repo, so to get the rest of the code you will need:

    ansible-playbook clone-all-repos.yml


==============================================================================


## Prerequisites

## GIT
Copy the code into a git repository that you have control of, the pipeline will need to read and write to the repo called whats-the-time-argoconf, but actually you will most likely want to tweak the application code etc. so copy all of it.


## A docker registry you can push images to.
I have used a quay.io account, you can use this for free, just create a robot account - which will 
Copy the code


## Openshift
I have used CRC (Code Ready Containers), its free runs on a laptop, and has a really nice start, stop & destroy cycle - similar to vagrant style development. But you can use any OCP instance.


| Tool | Purpose |
|------|---------|
| [CRC](https://developers.redhat.com/products/openshift-local/overview) | Local OpenShift cluster |
| `oc` CLI | Cluster login |
| `jq` | Parse CRC credentials |
| Python `kubernetes` package | Required by `kubernetes.core` collection |
| Ansible collections (see below) | k8s modules |



## CRC SetUp

crc config set memory 24576
crc config set cpus 8
crc config set disk-size 200


## Ansible 
I am using ansible to standup my openshift configuration, the aim is to allow the developer to open the door, sit on the sofa and watch TV'! There is just one playbook that lays down the necessary configuration, but you will need to have ansible installed.

You can install any Ansible dependencies using the following:

```bash
ansible-galaxy collection install -r requirements.yml
pip install kubernetes
```




## Usage

You need to create a pull secret on cloud.redhat.com
and copy it to somewhere on your local file system


```bash
# 1. Start CRC
crc start -p /Users/msumner/pull-secret.txt



# Edit global-var-secrets.txt and fill in real values:
#   crc_password   — retrieve with: crc console --credentials -o json | jq -r '.clusterConfig.adminCredentials.password'

# 
#   gitea_password — Gitea/GIT user password


#    You will need to change the URL/ip to point to your git server.
#     Make sure your git repos are named identically to mine!

#   You will need to add your kubeadmin password, the script will login to openshift. 

#   Point to the container registry of your choice. 
#   quay_robot_token — robot account token from quay.io


# 3. Run the playbook - to stand up the environment.
ansible-playbook whatsthetime-platform-standup.yaml
```

Ansible playbook that bootstraps a local CRC cluster with everything needed to build and deploy the `whats-the-time` service — OpenShift Pipelines (Tekton), OpenShift GitOps (ArgoCD), namespaces, RBAC, and ArgoCD application configuration. A single playbook run takes a bare CRC cluster to a fully operational CI/CD environment.


The playbook is fully idempotent — safe to re-run on an existing cluster.

## What the playbook does

### OpenShift Pipelines (Tekton)
| Step | Action |
|------|--------|
| 1 | Creates OLM `Subscription` for `openshift-pipelines-operator-rh` |
| 2 | Waits up to 5 min for the CSV to reach `Succeeded` |
| 3 | Verifies the `pipelines.tekton.dev` CRD is registered |

### OpenShift GitOps (ArgoCD)
| Step | Action |
|------|--------|
| 4 | Creates OLM `Subscription` for `openshift-gitops-operator` |
| 5 | Waits up to 7.5 min for the CSV to reach `Succeeded` |
| 6 | Waits for the `openshift-gitops-server` deployment to become available |
| 7 | Creates a `cluster-admins` OpenShift Group containing `kubeadmin` |
| 8 | Patches the ArgoCD RBAC policy to grant `kubeadmin` admin role in the console |

### Namespaces and ArgoCD configuration
| Step | Action |
|------|--------|
| 9 | Applies namespace definitions (`whats-thetime-dev`, `whats-thetime-test`, `whats-thetime-prod`, `whats-the-time-pipelines`) |
| 10 | Labels app namespaces with `argocd.argoproj.io/managed-by: openshift-gitops` |
| 11 | Creates a Gitea repository secret in `openshift-gitops` so ArgoCD can pull from the private config repo |
| 12 | Applies ArgoCD `Application` CRs for dev (auto-sync), test (manual), and prod (manual) |

### Console plugins
| Step | Action |
|------|--------|
| 13 | Patches the cluster `Console` CR to enable `pipeline-console-plugin` and `gitops-plugin` |
| 14 | Waits for the `console` deployment to roll out with the new plugins active |

## Key variables

| Variable | Source | Description |
|----------|--------|-------------|
| `crc_password` | `$CRC_PASSWORD` env var | CRC kubeadmin password |
| `gitea_password` | `$GITEA_PASSWORD` env var | Gitea password for ArgoCD repo secret |
| `gitea_url` | playbook vars | GitOps config repo URL |
| `argocd_namespace` | playbook vars | `openshift-gitops` |
| `cluster_conf_dir` | playbook vars | Path to ArgoCD Application CRs |

## Pipeline secrets

The Tekton pipeline requires two secrets in the `whats-the-time-pipelines` namespace before it can run. These are **not** created by this playbook — apply them manually from the pipeline repo after the playbook completes.

### 1. Git credentials (`whats-the-time-git-secret`)

Used by the `git-clone` task to pull source from the private Gitea server. The secret is pre-built in `whats-the-time-secret.yaml` in the pipeline repo:

```bash
oc apply -f ../pipeline/whats-the-time-pipeline/whats-the-time-secret.yaml \
  -n whats-the-time-pipelines
```

The `.git-credentials` entry uses URL-encoded credentials (`Bl%40ckm00r` → `Bl@ckm00r`).

### 2. Registry credentials (`whats-the-time-registry-secret`)

Used by the `buildah` task to push the built image to Quay.io. The YAML for this secret is commented out in `whats-the-time-secret.yaml` because the robot account token must be kept out of source control.

To create it:

```bash
# 1. Base64-encode the robot account credentials
AUTH=$(echo -n "martin_c_sumner+whatsthetime:<robot-token>" | base64)

# 2. Create the secret directly
oc create secret docker-registry whats-the-time-registry-secret \
  --docker-server=quay.io \
  --docker-username=martin_c_sumner+whatsthetime \
  --docker-password=<robot-token> \
  -n whats-the-time-pipelines
```

Or uncomment and populate the `whats-the-time-registry-secret` block in `whats-the-time-secret.yaml`, regenerating the `.dockerconfigjson` value:

```bash
# Regenerate the auth blob if the token is rotated
echo -n "martin_c_sumner+whatsthetime:<new-token>" | base64
```

Then substitute the output into the `data..dockerconfigjson` field and apply:

```bash
oc apply -f ../pipeline/whats-the-time-pipeline/whats-the-time-secret.yaml \
  -n whats-the-time-pipelines
```

### Verify secrets are present

```bash
oc get secrets -n whats-the-time-pipelines
# Expected:
#   whats-the-time-git-secret       Opaque
#   whats-the-time-registry-secret  kubernetes.io/dockerconfigjson
```

## Post-install verification

```bash
# Check Pipelines operator
oc get csv -n openshift-operators | grep pipelines

# Check GitOps operator
oc get csv -n openshift-operators | grep gitops

# Check ArgoCD is running
oc get pods -n openshift-gitops

# Check ArgoCD applications
oc get applications -n openshift-gitops

# Get ArgoCD console URL
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}'
```

Log into the ArgoCD console via **"Log In Via OpenShift"** using `kubeadmin` credentials.

## Troubleshooting

**CSV stuck in `Installing`**
OLM can be slow on CRC. Both operators have generous retry windows. If the playbook times out, re-run it — it is idempotent.

**ArgoCD console shows no applications after login**
Ensure `kubeadmin` is in the `cluster-admins` OpenShift Group (step 7 above). The RBAC policy maps that group to the ArgoCD admin role.

**Credential assertion fails**
All credentials are validated at the very start of the playbook. Ensure `secrets.txt` exists in the same directory as the playbook and that all three keys (`crc_password`, `gitea_password`, `quay_robot_token`) are populated. Copy `secrets.example` to get started.

**Pipelines console plugin not appearing (GitOps plugin is visible)**
The `ConsolePlugin` resource name is `pipelines-console-plugin` (with an `s`). If you applied an older version of this playbook that used `pipeline-console-plugin`, re-run the playbook — the console plugin tasks are idempotent and will correct the plugin list.

**`kubernetes` Python package not found**
```bash
pip install kubernetes
```

## Related repositories

| Repo | Role |
|------|------|
| `whats-the-time-pipeline` | Tekton build and promotion pipelines |
| `whats-the-time-cluster-conf` | ArgoCD Application CRs and namespace definitions applied by this playbook |
| `argo-whatsthetime-conf` | Kustomize base/overlays watched by ArgoCD |
| `whatsthetime` | .NET 8 application source and Dockerfile |




## Now its Playtime!
=====================


First a Quick Orientation
==========================
The playbook will create namespaces called: whats-thetime-dev, whats-thetime-test, whats-thetime-prod
whats-thetime-dev also whats-thetime-pipelines (where you will find the tekton pipelines!).

The playbook will stand-up pipelines that will build bake a container from the whats-the-time-app repo. 

It also configures argo to look at the whats-the-time-argoconf repo to be its source of truth for deployment, you can see the applications deployed under openshift-gitops. one for dev, test and prod. 


There is a playbook called show-time.yml that will show you the routes exposed from openshift.

ansible-playbook show-time.yaml - keep running this periodically to see the routes exposed dev/test/prod as you promote.

1. go to console>pipelines and you will now see the pipelines > three of them.

 Use the whats-the-time-pipeline and run it via the PR pipeline run, go to the actions>>rerun
 you should now see the pipeline running, check the logs, check the details tab. 

     This will do the build code >> bake container >> push to registry 
     It uses the GIT SHA, to tag the image and update the -argoconf repo, with the sha to deploy.

2. create another browser tab (its easier because you will be back in pipelines shortly), and navigate to the home>projects>openshift-gitops project, then click the openshift-gitops project.

Now click the gitops tab in the left of the webconsole, and select applications. 
click on whats-the-time-dev, then click the little argo octopus logo, this will take you into the argoCD console. Use the 'login via Openshift' button, kubeadmin and password, (allow selected permissions). >> click the applications tab and you will see the argoCD console.

3. You can now use the ansible playbook called show-time.yaml to quickly expose the routes for you.
    ansible-playbook show-time.yaml

   ok: [localhost] => {
    "msg": [
        "==============================================",
        "  Exposed routes",
        "==============================================",
        "  dev  : http://whats-the-time-dev.apps-crc.testing/time",
        "  test : pending — trigger sync in ArgoCD",
        "  prod : pending — trigger sync in ArgoCD",
        "=============================================="
    ]
}

You can take the path for dev and put it into another tab on your browser to see: 

"the time in Tokyo, is presently 2026-05-12 15:18:49 UTC
the time in Sao Paulo, is presently 2026-05-12 15:18:49 UTC
the time in Nairobi, is presently 2026-05-12 15:18:49 UTC"

## Quick quiz: can you figure out how the cities are being passed into the app?

Notice that test is pending in the output of show-time.yaml > because we have not deployed there yet.


4. go back to pipelines and run the promote-to-test-run pipeline, note the pipeline is called promote-to-test >> but you want to use the PipelineRun object, this passes params to the pipeline object and makes it reusable across many projects for build similar applications.

5. Having done this you now need to sync in ArgoCD, so go back to the argoCD tab that you logged into earlier, and sync the project called whats-the-time-test. This is two stage, you click the sync in the console then have to click the sync in the next box. You will see the tab turn green for whats-the-time-test

    You can now use the ansible playbook called show-time.yaml to quickly expose the routes for you.
    ansible-playbook show-time.yaml

  ok: [localhost] => {
    "msg": [
        "==============================================",
        "  Exposed routes",
        "==============================================",
        "  dev  : http://whats-the-time-dev.apps-crc.testing/time",
        "  test : http://whats-the-time-test.apps-crc.testing/time",
        "  prod : pending — trigger sync in ArgoCD",
        "=============================================="
    ]
}

you can now take the route for test and then copy it into a tab and you should see:

the time in Sydney, is presently 2026-05-12 15:32:49 UTC
the time in Mumbai, is presently 2026-05-12 15:32:49 UTC
the time in Toronto, is presently 2026-05-12 15:32:49 UTC


6. Last one, repeat the same process of pipelineRun>>Rerun and then argosync for prod.
rerun the playbook and you should now see the last route exposed:

ok: [localhost] => {
    "msg": [
        "==============================================",
        "  Exposed routes",
        "==============================================",
        "  dev  : http://whats-the-time-dev.apps-crc.testing/time",
        "  test : http://whats-the-time-test.apps-crc.testing/time",
        "  prod : http://whats-the-time.apps-crc.testing/time",
        "=============================================="
    ]
}

the time in New York, is presently 2026-05-12 15:35:21 UTC
the time in Cape Town, is presently 2026-05-12 15:35:21 UTC
the time in Dubai, is presently 2026-05-12 15:35:21 UTC


NOTE
=====
1. We have shown this happening manually > you run the pipeline, you sync the argoCD, but this would probably all happen via webhooks, this way you commit to git and the pipeline is running before you can switch tabs etc. But whilst learning it is useful to see this happening in your own time, and at your command.

2. This little project shows separation of platform for which I am using ansible to automate and application ArgoCD. The standup shows an 'everything as code' style: namespaces are created, operators are configured, pipelines and Argo are fully configured. 

3. Notice how you don't need to do anything in terms of pipeline conf. other than start the things.
   This is one of the lovely things about Tekton as a pipeline tech. 

4. The promotion pipelines would typically be in another project then authorisation via RBAC would be provided to stop people from just running the promotion, it would typically be authorised users only.

5. If you want to see how it is laid down, just read the playbook it is that easy!


Quiz Answer:
============
The array of cities is configured to each environment and passed into the time app via an environment variable, this is synced to the namespace via ArgoCD, you can see these in argoconf, overlays: dev, test and prod. 