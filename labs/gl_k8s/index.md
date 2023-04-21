# GitOps with GitLab

This lab shows how to connect a Kubernetes cluster with GitLab for pull and push-based deployments and easy security integrations. In order to do so, the following elements are required:

* A Kubernetes cluster that you can access and can create new resources
* You will need `kubectl` and your local environment configured to access the cluster.
* A GitLab Kubernetes agent



## Set up our GitOps project 

First, create a project for our GitOps configuration using the **GitLab Cluster Management** template.

### Create an agent configuration file

The Agent has a component that needs to be installed into your cluster named `agentk`. Once `agentk` is installed it reaches out to GitLab, and authenticates itself with a token. So, the first step is to get a token from GitLab by registering it. 

Once the connection is established, the Agent retrieves its own configuration from GitLab. This configuration is a `config.yaml` file under a repository, and you actually register the location of this configuration file when you register a new Agent. The configuration describes the various capabilities of the agent.

To create the agent configuration

1. Create `.gitlab/agents/gitlab-k8s-agent/config.yaml`
2. Leave it blank for now.



### Register the agent with GitLab

You must register an agent before you can install the agent in your cluster. To register an agent:

1. From the left sidebar, select **Infrastructure > Kubernetes clusters**.
2. Select **Connect a cluster**
   1. Select `gitlab-k8s-agent` from the drop-down menu
3. Select **Register**.
4. GitLab generates an access token for the agent. You need this token to install the agent in your cluster.
5. Save the token somewhere safe!

### Install the agent in the cluster

To connect the cluster to GitLab, install the registered agent in your cluster. 

The recommended approach for this is to use `helm`

Install `helm` on your Ubuntu VM: 

```
export VERIFY_CHECKSUM=false
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

Now execute the following code **(replace <your token>)** to install the agent

```
helm repo add gitlab https://charts.gitlab.io
helm repo update
helm upgrade --install gitlab-k8s-agent gitlab/gitlab-agent \
    --namespace gitlab-agent-gitlab-k8s-agent \
    --create-namespace \
    --set image.tag=v15.11.0 \
    --set config.token=<your token>\
    --set config.kasAddress=wss://kas.gitlab.com
```



### Test the Agent

We have installed the Agent; now what? How can we start using it? In the next section, we will see in detail how to deploy a more serious application into the cluster. 

We need to confirm the agent is working first. 

To check cluster synchronization works, let's deploy a `ConfigMap`.

* Create `kubernetes/test_config.yaml`

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: gitlab-gitops
    namespace: default
  data:
    key: It works!
  ```

* Modify your Agent configuration file under `.gitlab/agents/gitlab-k8s-agent/config.yaml`, and add the following to it:

  ```yaml
  gitops:
    # Manifest projects are watched by the agent. Whenever a project changes,
    # GitLab deploys the changes using the agent.
    manifest_projects:
    - id: path/to/your/project
      default_namespace: default
      # Paths inside of the repository to scan for manifest files.
      # Directories with names starting with a dot are ignored.
      paths:
      - glob: 'kubernetes/test_config.yaml'
      #- glob: 'kubernetes/**/*.yaml'
  ```

  **Change the `- id: path/to/your/project` line above to point to your project's path**

  for example: `jruels/gitlab-gitops`

The above configuration tells the Agent to keep the `kubernetes/test_config.yaml` file in sync with the cluster. I've left a comment line at the end showing how to use wildcards. This will come in handy in the future. The`default_namespace` is used if no namespace is provided in the Kubernetes manifests.

Once you commit the above changes, GitLab notifies `agentk` about the changed files. First, `agentk` updates its configuration; second, it retrieves the `ConfigMap`.

Wait a few seconds, and run `kubectl describe configmap gitlab-gitops` to check that the changes were applied to your cluster. 

### Congrats! 

We have successfully set up a GitOps flow and deployed a configuration file to the cluster! 

In a future lab, we will configure secrets and deploy a full application. 
