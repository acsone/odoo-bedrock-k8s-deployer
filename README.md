odoo-bedrock-k8s-deployer
============

<!--ts-->
  * [Introduction](#introduction)
  * [Getting Started](#getting-started)
    * [Prerequisites](#prerequisites)
    * [Variables](#variables)
    * [Passing Your Own Variables](passing-your-own-variables)
  * [Use Case Example](#use-case-example)
    * [Image Build](#image-build)
    * [Deployment](#deployment)
  * [Credits](#credits)
    * [Contributors](#contributors)
<!--te-->

Introduction
============

This project will help you automatically deploy (and clean afterwards) an Odoo application to your Kubernetes cluster and create associated database to this deployment. 
The aim of this project is to facilitate the deployment of test environments with GitLab CI by using Helm to deploy your application into your Kubernetes cluster.

The `obk_deploy` script makes sure that all necessary variables are set and that a namespace exists inside your cluster for your project (if not creates it). Then it creates a secret in order to connect to your GitLab Container Registry where you should previously have uploaded the image to deploy. Finally, your application is deployed thanks to the Helm chart available in the chart/ folder.

The second script, `obk_undeploy`, deletes and purges the deployment. Note that the associated database may also be droped at teardown by setting a variable at deployment time, see **Variables** section.

For detailed information about Helm charts, please refer to the official [Helm's documentation](https://docs.helm.sh/).

Getting Started
============
You can simply download this project from your `.gitlab-ci.yml` file (for instance in the before_script stage) with git clone, scripts are directly executable.
```
git clone --depth=1 https://github.com/acsone/odoo-bedrock-k8s-deployer.git
./obk_deploy
```

Prerequisites
-----

* **GitLab Runner** - Your runner needs to have `kubectl` and `helm` installed. 
* **Kubernetes** - You obviously need a Kubernetes cluster to enable deployment. GitLab is particularly well integrated with Kubernetes Engine from Google Cloud, take a look at [GKE Quickstart](https://cloud.google.com/kubernetes-engine/docs/quickstart) if you wish to set up such a cluster.
* **Odoo Image** - You have to previously upload your application's image to your GitLab Container Registry with `$CI_REGISTRY_IMAGE/$CI_COMMIT_REF_SLUG:$CI_COMMIT_SHA` as name. An example of how to do so is provided in [Image Build](#image-build) section.
* **Base Domain** - You will need a domain configured with wildcard DNS. It can be defined either in instance-wide settings in the *Settings > CI/CD* under the "Variables" section or at the project or group level as a variable `APP_DEPLOY_DOMAIN` in your `gitlab-ci.yml` file.
A wildcard DNS A record matching the base domain is required, you need a DNS entry like:
  ```
  *.example.com    3600    A    1.2.3.4
  ```
  Use this domain for the url of the environment you deploy, for instance:
  ```
  environment:
    name: review/$CI_COMMIT_REF_NAME
    url: http://$CI_PROJECT_PATH_SLUG-$CI_ENVIRONMENT_SLUG.$APP_DEPLOY_DOMAIN
  ```

* **PostgreSQL** - You will need a PostgreSQL instance. It may run on a pod into your cluster (you can find many ressources online about how to set up PostgreSQL on Kubernetes) or outside of it. Moreover, a temporary PostgreSQL image may be launch for the environment by setting the `POSTGRES_ENABLED` variable *(feature unavailable in this version)*.
* **Persistent Volume** - In order to keep the state of the Odoo application deployed in your environment persistent, you need to make the filestore reachable from your deployment. The default configuration requires NFS to run on a pod in the kube-public namespace of your cluster accessible with a service named "nfs-service", and will store the data in your NFS at `/data/odoo`.
* **Tiller** - You will need a Tiller deployment inside your cluster. The easiest way to install Tiller on Kubernetes is to run ``helm init`` which will connect to whatever cluster kubectl is connected to by default and install Tiller into the `kube-system` namespace. Thus, you should take a look at [Helm](https://docs.helm.sh/using_helm/#quickstart).

You may also want to implement a mechanism for the deletion of images pushed to your GitLab Container Registry. Unfortunately the GitLab API does not provide a way to easliy manage them. We advice you to follow [this topic](https://gitlab.com/gitlab-org/gitlab-ce/issues/21608) for workarounds.

Variables
-----

The following environment variables may or must be set in order to configure the deployment.

* **APP_DEPLOY_DOMAIN** - Is the application deployment domain and should be set as a variable at the group or project level.
* **POSTGRES_ENABLED** - *Disabled in this version* Must be set to "false" except if you wish to create a new DB within the namespace of your app *(optional, default false)*.
* **OBK_DROP_DB** - Set to "true" if the DB created by this script must be droped when environment is deleted *(optional, default false)*.
* **OBK_VOLUME** - The volume where your filestore is stored. Default is NFS running into your cluster *(optional)*.
   ```
   nfs:
     server: "nfs-service.kube-public.svc.cluster.local"
     path: "/data/odoo"
   ```
* **OBK_ODOO_DATA_DIR** - The mount path for the volume *(optional, default /data/odoo)*.  
* **OBK_SECRET_ODOO_DB_USER** - Your PostgreSQL DB credentials encoded in base64.
* **OBK_SECRET_ODOO_DB_PASSWORD** - Your PostgreSQL DB credentials encoded in base64.
* **OBK_ODOO_DB_NAME** - The name of the PostgresSQL DB that will be created *(optional, default postgres-$CI_ENVIRONMENT_SLUG)*.
* **OBK_ODOO_DB_HOST** - The host where your PostgreSQL is running *(optional, default postgres-service.kube-public.svc.cluster.local)*.
* **OBK_ODOO_DB_PORT** - The port which your PostgreSQL is accessible at *(optional, default 5432)*.

GitLab CI exposes many environment variables, see [GitLab CI/CD Variables](https://docs.gitlab.com/ee/ci/variables/) for full list. Following are the CI/CD Variables used in this project.

* **KUBE_NAMESPACE** - The Kubernetes namespace is auto-generated if not specified. The default value is <project_name>-<project_id>.
* **CI_ENVIRONMENT_SLUG** - A simplified version of the environment name, suitable for inclusion in DNS, URLs, Kubernetes labels, etc.
* **CI_PROJECT_PATH** - The namespace with project name.
* **CI_PROJECT_PATH_SLUG** -  `$CI_PROJECT_PATH` lowercased and with everything except 0-9 and a-z replaced with -. Use in URLs and domain names.
* **CI_PROJECT_VISIBILITY** - The project visibility (internal, private, public).
* **CI_REGISTRY** - If the Container Registry is enabled it returns the address of GitLab's Container Registry.
* **CI_DEPLOY_USER** - Authentication username of the GitLab Deploy Token, only present if the Project has one related.
* **CI_REGISTRY_USER** - The username to use to push containers to the GitLab Container Registry.
* **CI_DEPLOY_PASSWORD** - Authentication password of the GitLab Deploy Token, only present if the Project has one related.
* **CI_REGISTRY_PASSWORD** - The password to use to push containers to the GitLab Container Registry.
* **GITLAB_USER_EMAIL** - The email of the user who started the job.
* **CI_PIPELINE_ID** - The unique id of the current pipeline that GitLab CI uses internally.
* **CI_JOB_ID** - The unique id of the current job that GitLab CI uses internally.
* **CI_REGISTRY_IMAGE** - If the Container Registry is enabled for the project it returns the address of the registry tied to the specific project.
* **CI_COMMIT_REF_NAME** - The branch or tag name for which project is built.
* **CI_COMMIT_REF_SLUG** -  `$CI_COMMIT_REF_NAME` lowercased, shortened to 63 bytes, and with everything except 0-9 and a-z replaced with -. No leading / trailing -. Use in URLs, host names and domain names.
* **CI_COMMIT_SHA** - The commit revision for which project is built.
* **CI_ENVIRONMENT_URL** - The URL of the environment for this job.


Passing Your Own Variables
-----
All environment variables with prefix `OBK_{SECRET_}ODOO_` are set in the deployed pod's environment (with the prefix removed). Therefore, if you need more environment variable than the one mentioned above to be passed to the pod, simply add the `OBK_ODOO_` prefix (`OBK_SECRET_ODOO_` for sensitive data) to the variable of your deployment environment. Take care, secret variables must be base64 encoded !

Use Case Example
============
Image Build
-----
You are free to build your image the way you want, with the tools of your choice, as soon as it ends up in your GitLab Container Registry as `$CI_REGISTRY_IMAGE/$CI_COMMIT_REF_SLUG:$CI_COMMIT_SHA`. Nonetheless, here is an example of how we are used to build and upload our images, with the [Buildah](https://github.com/containers/buildah) and [Podman](https://github.com/containers/libpod) tools.

```
before_script:
  - |
    function registry_login() {
        if [[ -n "$CI_REGISTRY_USER" ]]; then
          echo "Logging in to GitLab Container Registry with CI credentials..."
          podman login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
          echo ""
        fi
    }
    function build_and_push_image() {
        registry_login

        echo "Building Dockerfile-based application..."
        buildah bud --pull-always --volume ${PWD}/release:/tmp/release -t "$CI_REGISTRY_IMAGE/$CI_COMMIT_REF_SLUG:$CI_COMMIT_SHA" .

        echo "Pushing to GitLab Container Registry..."
        buildah push "$CI_REGISTRY_IMAGE/$CI_COMMIT_REF_SLUG:$CI_COMMIT_SHA"
        
        echo "Removing image from Runner..."
        buildah rmi "$CI_REGISTRY_IMAGE/$CI_COMMIT_REF_SLUG:$CI_COMMIT_SHA"
        echo ""
    }

build:
  stage: build
  script:
    - build_and_push_image
```

Deployment
-----
Here is a sample of basic `gitlab-ci.yml` file configuration.

```
before_script:
  - git clone --depth=1 https://github.com/acsone/odoo-bedrock-k8s-deployer.git

deploy-review:
  stage: deploy
  script:
    - ./odoo-bedrock-k8s-deployer/obk_deploy
  environment:
    name: review/$CI_COMMIT_REF_NAME
    url: http://$CI_PROJECT_PATH_SLUG-$CI_ENVIRONMENT_SLUG.$APP_DEPLOY_DOMAIN
    on_stop: stop_review
  when: manual
  only:
    kubernetes: active
    
stop_review:
  stage: deploy
  script:
    - ./odoo-bedrock-k8s-deployer/obk_undeploy
  environment:
    name: review/$CI_COMMIT_REF_NAME
    action: stop
  when: manual
  allow_failure: true
  only:
    kubernetes: active
```

Credits
============
Inspiration has been drawn from [GitLab Auto DevOps](https://docs.gitlab.com/ee/topics/autodevops/).

Contributors
-----
* Quentin Groulard \<<quentin.groulard@acsone.eu>\>
* Stéphane Bidoul \<<stephane.bidoul@acsone.eu>\>
