## GitLab GitOps Secrets 

### Manage secrets in Git with a GitOps approach

To manage secrets in Git, we will need some tooling to take care of the encryption/decryption of the secrets. In this lab we will set up and use [Bitnami's Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets). 

Bitnami's Sealed Secrets comprises an in-cluster controller and a CLI tool. The cluster component defines a `SealedSecret` custom resource that stores the encrypted secret and related metadata. Once a `SealedSecret` is deployed into the cluster, the controller decrypts it and creates a native Kubernetes `Secret` resource. To create a `SealedSecret` resource, the `kubeseal` utility can be used. `kubeseal` can take a public key and transform and encrypt a native Kubernetes `Secret` into a `SealedSecret`, and `kubeseal` can help retrieve the public key from the cluster-side controller.

## Setting up Bitnami's Sealed Secrets

As the GitLab Agent supports pure Kubernetes manifests to do GitOps, we will need the manifests for Sealed Secrets. Open the [Sealed Secrets releases page](https://github.com/bitnami-labs/sealed-secrets/releases/) and download the most recent release of `controller.yaml`.

- In GitLab, enter the `kubernetes` folder and upload the `controller.yaml` file. 

  Update the agent config to watch for files in the `kubernetes` directory by uncommenting.

  ```yaml
   - glob: 'kubernetes/**/*.yaml'
  ```

  You also need to comment out the line: 

  ```yaml
  - glob: 'kubernetes/test_config.yaml'
  ```

- Commit your changes.

- Rename `controller.yaml` to `sealed-secrets.yaml`



Check that the controller was deployed successfully using: `kubectl get pods -n kube-system -l name=sealed-secrets-controller`

## Retrieving the public key

Install `kubeseal` in the cluster

```bash
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.20.5/kubeseal-0.20.5-linux-amd64.tar.gz
tar -xvzf kubeseal-0.20.5-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```



While you can encrypt a secret directly with `kubeseal`, this approach requires access to the Kube API. Instead of providing access, we can fetch the public key from the Sealed Secrets controller and store it in the Git repo. The public key can be used to encrypt secrets but is useless for decrypting them.

```
kubeseal --fetch-cert > sealed-secrets.pub.pem
```



### Clone the GitLab project

Follow the steps from previous labs to create an SSH key, add it to GitLab, and clone the project to the Ubuntu VM.



### How to avoid storing unencrypted secrets

I prefer to have an `ignored` directory within my Git repo. The content of this directory is never committed to Git, and I put sensitive data under this directory.

Enter the repository directory and run:

```
mkdir ignored
cat <<EOF > ignored/.gitignore
*
!.gitignore
EOF
```

Now, you can create sealed secrets with the following two commands:

```
echo "Very secret" | kubectl create secret generic my-secret -n gitlab-agent-gitlab-k8s-agent --dry-run=client --type=Opaque --from-file=token=/dev/stdin -o yaml > ignored/my-secret.yaml
kubeseal --format=yaml --cert=~/sealed-secrets.pub.pem < ignored/my-secret.yaml > kubernetes/my-secret.yaml
```

The first command creates a regular Kubernetes `Secret` resource in the `gitlab-agent` namespace. Setting the namespace is important if you use Sealed Secrets, and every SealedSecret is scoped for a specific namespace. 

The second command takes a `Secret` resource object and turns it into an encrypted `SealedSecret`resource. In my case, the secret file:

```yaml
apiVersion: v1
data:
  token: VmVyeSBzZWNyZXQK
kind: Secret
metadata:
  creationTimestamp: null
  name: my-secret
  namespace: gitlab-agent-gitlab-k8s-agent
type: Opaque
```



Becomes: 

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: my-secret
  namespace: gitlab-agent-gitlab-k8s-agent
spec:
  encryptedData:
    token: AgB0CvGF5GKhl6Ac+07+dyKiDwvP9dQ81SAVgD8zNDr95JmWycQe4wlqHBWNcxywQEee4MYWPetbxS+rqy0KjA5Kko/iFSsOD30VDLKgIxJodEEKlGh8YGEi3rMc0u7LiiwOKGdNVmk4PjoyTFg9prrnQPTflYKufGR9J7PHKBGdPwn3J0KKrpybYwn3i6IwyX3bi8+yV+PJ3u1PyU5vROHApOvh0ZtSFhMzDTPB/le3DOoFxx3gfPehLpQRyKw6n3glKCDRPX+ZBuXZ7juCCdfX0xXKIPbwmHrqHbpwS79hU9RMsE5MYKrnOQmpaF10t0McHzeLE7CEJVqD/uezt/jxkja2Ry6d3m1t5H8jG3YLkhPbhdeeNUqGmS44aSXHYx9iVax01fVP5lqmo3qxcndFUviiKS3RGwIziuQ8uhggwUdtZARpXw+HEPtpFOzKRuINXd8bBuJRrtsgHMHJi+Ec7GDRAEEQQ9d9dO/Qh2PBjuUoGNjFomoS4FANXse06QqMRagwPzkD4gLTZu8gk+lmWm9PNC4TBAa7vw3aUmVCcwxRgh5m7CVZr57SBkKz7QCXX/cme1m3Otna5CmAHW6AW0tLKoA7fFLidd96cvp2O4vVfdhMIfOdiiQcO6ANXxYvR5p4m80tFdA67STF0y0h9KwzZTBXPb+IeaazfbP0ZbW6lOpS6/BmsDcyJ2M/7gzIBdu+HrRhZfeTPFU=
  template:
    metadata:
      creationTimestamp: null
      name: my-secret
      namespace: gitlab-agent-gitlab-k8s-agent
    type: Opaque
```

Just commit the `SealedSecret` and quickly start to watch for the event stream using `kubectl get events --all-namespaces --watch` to see when the sealed secret is unsealed and applied as a regular `Secret`.

You should see something like this: 

```
gitlab-agent-gitlab-k8s-agent   1s          Normal    Unsealed            sealedsecret/my-secret                            SealedSecret unsealed successfully
```

