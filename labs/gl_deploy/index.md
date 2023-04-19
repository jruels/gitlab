# GitLab Continous Deployment Pipeline

GitLab provides powerful features beyond hosting a code repository. You can track issues, host packages and registries, maintain Wikis, set up continuous integration (CI) and continuous deployment (CD) pipelines, and more.

In this tutorial you’ll build a continuous deployment pipeline with GitLab. You will configure the pipeline to build a Docker image, push it to the GitLab container registry, and deploy it to your server using SSH. The pipeline will run for each commit pushed to the repository.

You will deploy a small, static web page, but the focus of this tutorial is configuring the CD pipeline. The static web page is only for demonstration purposes; you can apply the same pipeline configuration using other Docker images for the deployment as well.

When you have finished this tutorial, you can visit `http://your_server_IP` in a browser for the results of the automatic deployment.

## Creating the GitLab Repository

Let’s start by creating a GitLab project and adding an HTML file to it. You will later copy the HTML file into an Nginx Docker image, which in turn you’ll deploy to the server.

* Create a new private project without a `README.md` file

* Create an `index.html` with the following 

  ```html
  <html>
    <body>
      <h1>My Personal Website</h1>
    </body>
  </html>
  ```

* Click **Commit changes** to create the file.

This HTML will produce a blank page with one headline showing **My Personal Website** when opened in a browser.

`Dockerfiles` are recipes used by Docker to build Docker images. Let’s create a `Dockerfile` to copy the HTML file into an Nginx image.

* Create a file named `Dockerfile` with the following instructions

  ```dockerfile
  FROM nginx:1.24.0-alpine
  
  COPY index.html /usr/share/nginx/html
  
  ```

The `FROM` instruction specifies the image to inherit from—in this case the `nginx:1.24.0-alpine` image. `1.24.0-alpine` is the image tag representing the Nginx version. The `nginx:latest` tag references the latest Nginx release, but that could break your application in the future, which is why fixed versions are recommended.

The `COPY` instruction copies the `index.html` file to `/usr/share/nginx/html` in the Docker image. This is the directory where Nginx stores static HTML content.

* Click **Commit changes** to create the file.

In the next step, you’ll configure a GitLab runner to keep control of who gets to execute the deployment job.

## Registering a GitLab Runner

In order to keep track of the environments that will have contact with the SSH private key, you’ll register your server as a GitLab runner.

In your deployment pipeline you want to log in to your server using SSH. To achieve this, you’ll store the SSH private key in a GitLab CI/CD variable. 

The SSH private key is a very sensitive piece of data, because it is the entry ticket to your server. Usually, the private key never leaves the system it was generated on. In the usual case, you would generate an SSH key on your host machine, then authorize it on the server (that is, copy the public key to the server) in order to log in manually and perform the deployment routine.

Here the situation changes slightly: You want to grant an autonomous authority (GitLab CI/CD) access to your server to automate the deployment routine. Therefore the private key needs to leave the system it was generated on and be given in trust to GitLab and other involved parties. You never want your private key to enter an environment that is not either controlled or trusted by you.

Besides GitLab, the GitLab runner is yet another system that your private key will enter. For each pipeline, GitLab uses runners to perform the heavy work, that is, execute the jobs you have specified in the CI/CD configuration. That means the deployment job will ultimately be executed on a GitLab runner, hence the private key will be copied to the runner such that it can log in to the server using SSH.



Start by logging in to the server provided by your instructor and installing the `gitlab-runner`

Add the official GitLab repository, using the install script:

```bash
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh > script.deb.sh
sudo bash script.deb.sh
```



Next install the `gitlab-runner` service:

```bash 
sudo apt install gitlab-runner
```

Verify the installation by checking the service status:

```bash
systemctl status gitlab-runner
sudo gitlab-runner status
```

1. In your project, in the left navigation pane, click **Settings > CI/CD**.
2. Scroll down to the **Runners** section. Click the **Expand** button next to that section.
3. Within the **Specific runners** section, navigate to **Set up a specific runner manually**.
4. Copy the URL in step 2, labeled **Register the runner with this URL**.
5. Also copy the **registration token**, and save it for later in this lab.
6. Disable **shared runners**

Back in your terminal, register the runner (remember to replace the project_token):

```bash
sudo gitlab-runner register -n --url https://gitlab.com --registration-token <project_token> --executor docker --description "Deployment Runner" --docker-image "docker:stable" --tag-list deployment --docker-privileged
```

Confirm the runner registration was succesful. 

```
sudo gitlab-runner list 
```



Also check in the GitLab UI > Settings > CICD > Runners

You should see something simliar to:

<img src="images/runner.png" alt="image-20230418233306326" style="zoom:50%;" />

## Creating a Deployment User

You are going to create a user that is dedicated for the deployment task. You will later configure the CI/CD pipeline to log in to the server with that user.

On your server, create a new user:

```bash
sudo adduser deployer
```

You’ll be guided through the user creation process. Enter a strong password and optionally any further user information you want to specify. Finally confirm the user creation with `Y`.

Add the user to the Docker group:

```bash
sudo usermod -aG docker deployer
```

This permits **deployer** to execute the `docker` command, which is required to perform the deployment.

## Setting Up an SSH Key

You are going to create an SSH key for the deployment user. GitLab CI/CD will later use the key to log in to the server and perform the deployment routine.

Let’s start by switching to the newly created **deployer** user for whom you’ll generate the SSH key:

```bash
sudo su - deployer
```

Next, generate a 4096-bit SSH key. It is important to answer the questions of the `ssh-keygen` command correctly:

1. First question: answer it with `ENTER`, which stores the key in the default location (the rest of this tutorial assumes the key is stored in the default location).
2. Second question: do not provide a passphrase.

To summarize, run the following command and confirm both questions with `ENTER` to create a 4096-bit SSH key and store it in the default location with an empty passphrase:

```bash
ssh-keygen -b 4096
```

To authorize the SSH key for the **deployer** user, you need to append the public key to the `authorized_keys` file:

```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

In this step you have created an SSH key pair for the CI/CD pipeline to log in and deploy the application. Next you’ll store the private key in GitLab to make it accessible during the pipeline process.



## Storing the Private Key in a GitLab CI/CD Variable

You are going to store the SSH private key in a GitLab CI/CD file variable, so that the pipeline can make use of the key to log in to the server.

When GitLab creates a CI/CD pipeline, it will send all variables to the corresponding runner and the variables will be set as environment variables for the duration of the job. In particular, the values of **file** variables are stored in a file and the environment variable will contain the path to this file.

While you’re in the variables section, you’ll also add a variable for the server IP and the server user, which will inform the pipeline about the destination server and user to log in.

Start by showing the SSH private key:

```bash
cat ~/.ssh/id_rsa
```

Copy the output to your clipboard. Make sure to add a linebreak after `-----END RSA PRIVATE KEY-----`:

Now navigate to **Settings** > **CI / CD** > **Variables** in your GitLab project and click **Add Variable**. Fill out the form as follows:

- Key: `ID_RSA`
- Value: Paste your SSH private key from your clipboard (including a line break at the end).
- Type: **File**
- Environment Scope: **All (default)**
- Protect variable: **Checked**
- Mask variable: **Unchecked**
- Expand variable reference: **Checked**

**Note:** The variable can’t be masked because it does not meet the regular expression requirements (see [GitLab’s documentation about masked variables](https://gitlab.com/help/ci/variables/README#masked-variables)). However, the private key will never appear in the console log, which makes masking it obsolete.

A file containing the private key will be created on the runner for each CI/CD job and its path will be stored in the `$ID_RSA` environment variable.

Create another variable with your server IP from the spreadsheet. Click **Add Variable** and fill out the form as follows:

- Key: `SERVER_IP`
- Value: `<Server IP from spreadsheet>`
- Type: **Variable**
- Environment scope: **All (default)**
- Protect variable: **Checked**
- Mask variable: **Checked**
- Expand variable reference: **Checked**

Finally, create a variable with the login user. Click **Add Variable** and fill out the form as follows:

- Key: `SERVER_USER`
- Value: `deployer`
- Type: **Variable**
- Environment scope: **All (default)**
- Protect variable: **Checked**
- Mask variable: **Checked**
- Expand variable reference: **Checked**

You have now stored the private key in a GitLab CI/CD variable, which makes the key available during pipeline execution. In the next step, you’re moving on to configuring the CI/CD pipeline.



## Configuring the `.gitlab-ci.yml` File

You are going to configure the GitLab CI/CD pipeline. The pipeline will build a Docker image and push it to the container registry. GitLab provides a container registry for each project. You can explore the container registry by going to **Packages & Registries** > **Container Registry** in your GitLab project  The final step in your pipeline is to log in to your server, pull the latest Docker image, remove the old container, and start a new container.

Now you’re going to create the `.gitlab-ci.yml` file that contains the pipeline configuration. In GitLab, go to the **Project overview** page, click the **+** button and select **New file**. Then set the **File name** to `.gitlab-ci.yml`.

(Alternatively you can clone the repository and make all following changes to `.gitlab-ci.yml` on your local machine, then commit and push to the remote repository.)

To begin add the following:

```yaml
stages:
  - publish
  - deploy
```

Each job is assigned to a stage. Jobs assigned to the same stage run in parallel (if there are enough runners available). Stages will be executed in the order they were specified. Here, the `publish` stage will go first and the `deploy` stage second. Successive stages only start when the previous stage finished successfully (that is, all jobs have passed). Stage names can be chosen arbitrarily.

When you want to combine this CD configuration with your existing CI pipeline, which tests and builds the app, you may want to add the `publish` and `deploy` stages after your existing stages, such that the deployment only takes place if the tests passed.

Following this, add this to your `.gitlab-ci.yml` file:

```yaml
variables:
  TAG_LATEST: $CI_REGISTRY_IMAGE/$CI_COMMIT_REF_NAME:latest
  TAG_COMMIT: $CI_REGISTRY_IMAGE/$CI_COMMIT_REF_NAME:$CI_COMMIT_SHORT_SHA
  DOCKER_TLS_CERTDIR: ""
```

[The variables section](https://docs.gitlab.com/ee/ci/yaml/#variables) defines environment variables that will be available in the context of a job’s `script` section. These variables will be available as usual Linux environment variables; that is, you can reference them in the script by prefixing with a dollar sign such as `$TAG_LATEST`. GitLab creates some predefined variables for each job that provide context specific information, such as the branch name or the commit hash the job is working on (read more about [predefined variable](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html)). Here you compose two environment variables out of predefined variables. They represent:

- `CI_REGISTRY_IMAGE`: Represents the URL of the container registry tied to the specific project. This URL depends on the GitLab instance. For example, registry URLs for [gitlab.com](https://gitlab.com/) projects follow the pattern: `registry.gitlab.com/your_user/your_project`. But since GitLab will provide this variable, you do not need to know the exact URL.
- `CI_COMMIT_REF_NAME`: The branch or tag name for which project is built.
- `CI_COMMIT_SHORT_SHA`: The first eight characters of the commit revision for which the project is built.

Both of the variables are composed of predefined variables and will be used to tag the Docker image.

`TAG_LATEST` will add the `latest` tag to the image. This is a common strategy to provide a tag that always represents the latest release. For each deployment, the `latest` image will be overridden in the container registry with the newly built Docker image.

`TAG_COMMIT`, on the other hand, uses the first eight characters of the commit SHA being deployed as the image tag, thereby creating a unique Docker image for each commit. You will be able to trace the history of Docker images down to the granularity of Git commits. This is a common technique when doing continuous deployments, because it allows you to quickly deploy an older version of the code in case of a defective deployment.

As you’ll explore in the coming steps, the process of rolling back a deployment to an older Git revision can be done directly in GitLab.

`$CI_REGISTRY_IMAGE/$CI_COMMIT_REF_NAME` specifies the Docker image base name. 

`$CI_REGISTRY_IMAGE` represents the `<registry URL>/<namespace>/<project>` part and is mandatory because it is the project’s registry root. `$CI_COMMIT_REF_NAME` is optional but useful to host Docker images for different branches. In this tutorial you will only work with one branch, but it is good to build an extendable structure. In general, there are three levels of image repository names supported by GitLab:

```
repository name levels
registry.example.com/group/project:some-tag
registry.example.com/group/project/image:latest
registry.example.com/group/project/my/image:rc1
```

For your `TAG_COMMIT` variable you used the second option, where `image` will be replaced with the branch name.

Next, add the following to your `.gitlab-ci.yml` file:

```yaml
publish:
  image: docker:latest
  stage: publish
  services:
    - docker:dind
  script:
    - docker build -t $TAG_COMMIT -t $TAG_LATEST .
    - docker login -u gitlab-ci-token -p $CI_BUILD_TOKEN $CI_REGISTRY
    - docker push $TAG_COMMIT
    - docker push $TAG_LATEST
```

The `publish` section is the first [job](https://docs.gitlab.com/ee/ci/yaml/#introduction) in your CI/CD configuration. Let’s break it down:

- `image` is the Docker image to use for this job. The GitLab runner will create a Docker container for each job and execute the script within this container. `docker:latest` image ensures that the `docker` command will be available.
- `stage` assigns the job to the `publish` stage.
- `services` specifies Docker-in-Docker—the `dind` service. This is the reason why you registered the GitLab runner in privileged mode.

[The `script` section](https://docs.gitlab.com/ee/ci/yaml/#script) of the `publish` job specifies the shell commands to execute for this job. The working directory will be set to the repository root when these commands will be executed.

- `docker build ...`: Builds the Docker image based on the `Dockerfile` and tags it with the latest commit tag defined in the variables section.
- `docker login ...`: Logs Docker in to the project’s container registry. You use the predefined variable `$CI_BUILD_TOKEN` as an authentication token. GitLab will generate the token and stay valid for the job’s lifetime.
- `docker push ...`: Pushes both image tags to the container registry.

Following this, add the `deploy` job to your `.gitlab-ci.yml`:

```yaml
deploy:
  image: alpine:latest
  stage: deploy
  tags:
    - deployment
  script:
    - chmod og= $ID_RSA
    - apk update && apk add openssh-client
    - ssh -i $ID_RSA -o StrictHostKeyChecking=no $SERVER_USER@$SERVER_IP "docker login -u gitlab-ci-token -p $CI_BUILD_TOKEN $CI_REGISTRY"
    - ssh -i $ID_RSA -o StrictHostKeyChecking=no $SERVER_USER@$SERVER_IP "docker pull $TAG_COMMIT"
    - ssh -i $ID_RSA -o StrictHostKeyChecking=no $SERVER_USER@$SERVER_IP "docker container rm -f my-app || true"
    - ssh -i $ID_RSA -o StrictHostKeyChecking=no $SERVER_USER@$SERVER_IP "docker run -d -p 80:80 --name my-app $TAG_COMMIT"
```

Alpine is a lightweight Linux distribution and is sufficient as a Docker image here. You assign the job to the `deploy` stage. The deployment tag ensures that the job will be executed on runners that are tagged `deployment`, such as the runner you configured in Step 2.

The `script` section of the `deploy` job starts with two configurative commands:

- `chmod og= $ID_RSA`: Revokes all permissions for **group** and **others** from the private key, such that only the owner can use it. This is a requirement, otherwise SSH refuses to work with the private key.
- `apk update && apk add openssh-client`: Updates Alpine’s package manager (apk) and installs the `openssh-client`, which provides the `ssh` command.

Four consecutive `ssh` commands follow. The pattern for each is:

```
ssh connect pattern for all deployment commands
ssh -i $ID_RSA -o StrictHostKeyChecking=no $SERVER_USER@$SERVER_IP "command"
```

In each `ssh` statement you are executing `command` on the remote server. To do so, you authenticate with your private key.

The options are as follows:

- `-i` stands for **identity file** and `$ID_RSA` is the GitLab variable containing the path to the private key file.
- `-o StrictHostKeyChecking=no` makes sure to bypass the question, whether or not you trust the remote host. This question can not be answered in a non-interactive context such as the pipeline.
- `$SERVER_USER` and `$SERVER_IP` are the GitLab variables you created in Step 5. They specify the remote host and login user for the SSH connection.
- `command` will be executed on the remote host.

The deployment ultimately takes place by executing these four commands on your server:

1. `docker login ...`: Logs Docker in to the container registry.
2. `docker pull ...`: Pulls the latest image from the container registry.
3. `docker container rm ...`: Deletes the existing container if it exists. `|| true` makes sure that the exit code is always successful, even if there was no container running by the name `my-app`. This guarantees a *delete if exists* routine without breaking the pipeline when the container does not exist (for example, for the first deployment).
4. `docker run ...`: Starts a new container using the latest image from the registry. The container will be named `my-app`. Port `80` on the host will be bound to port `80` of the container (the order is `-p host:container`). `-d` starts the container in detached mode, otherwise the pipeline would be stuck waiting for the command to terminate.

> **Note:** It may seem odd to use SSH to run these commands on your server, considering the GitLab runner that executes the commands is the exact same server. Yet it is required, because the runner executes the commands in a Docker container, thus you would deploy inside the container instead of the server if you’d execute the commands without the use of SSH. One could argue that instead of using [Docker as a runner executor](https://docs.gitlab.com/runner/executors/docker.html), you could use the [shell executor](https://docs.gitlab.com/runner/executors/shell.html) to run the commands on the host itself. But, that would create a constraint to your pipeline, namely that the runner has to be the same server as the one you want to deploy to. This is not a sustainable and extensible solution because one day you may want to migrate the application to a different server or use a different runner server. In any case it makes sense to use SSH to execute the deployment commands, may it be for technical or migration-related reasons.

Let’s move on by adding this to the deployment job in your `.gitlab-ci.yml`:

```yaml
deploy:
  environment:
    name: production
    url: http://your_server_IP
  only:
    - main
```

GitLab environments allow you to control the deployments within GitLab. You can examine the environments in your GitLab project by going to **Operations > Environments**. If the pipeline did not finish yet, there will be no environment available, as no deployment took place so far.

When a pipeline job defines an `environment` section, GitLab will create a deployment for the given environment (here `production`) each time the job successfully finishes. This allows you to trace all the deployments created by GitLab CI/CD. For each deployment you can see the related commit and the branch it was created for.

There is also a button available for re-deployment that allows you to rollback to an older version of the software. The URL that was specified in the `environment` section will be opened when clicking the **View deployment** button.

The `only` section defines the names of branches and tags for which the job will run. By default, GitLab will start a pipeline for each push to the repository and run all jobs (provided that the `.gitlab-ci.yml` file exists). The `only` section is one option of restricting job execution to certain branches/tags. Here you want to execute the deployment job for the `main` branch only.

Your complete `.gitlab-ci.yml` file will look like the following:

```yaml
stages:
  - publish
  - deploy

variables:
  TAG_LATEST: $CI_REGISTRY_IMAGE/$CI_COMMIT_REF_NAME:latest
  TAG_COMMIT: $CI_REGISTRY_IMAGE/$CI_COMMIT_REF_NAME:$CI_COMMIT_SHORT_SHA

publish:
  image: docker:latest
  stage: publish
  services:
    - docker:dind
  script:
    - docker build -t $TAG_COMMIT -t $TAG_LATEST .
    - docker login -u gitlab-ci-token -p $CI_BUILD_TOKEN $CI_REGISTRY
    - docker push $TAG_COMMIT
    - docker push $TAG_LATEST

deploy:
  image: alpine:latest
  stage: deploy
  tags:
    - deployment
  script:
    - chmod og= $ID_RSA
    - apk update && apk add openssh-client
    - ssh -i $ID_RSA -o StrictHostKeyChecking=no $SERVER_USER@$SERVER_IP "docker login -u gitlab-ci-token -p $CI_BUILD_TOKEN $CI_REGISTRY"
    - ssh -i $ID_RSA -o StrictHostKeyChecking=no $SERVER_USER@$SERVER_IP "docker pull $TAG_COMMIT"
    - ssh -i $ID_RSA -o StrictHostKeyChecking=no $SERVER_USER@$SERVER_IP "docker container rm -f my-app || true"
    - ssh -i $ID_RSA -o StrictHostKeyChecking=no $SERVER_USER@$SERVER_IP "docker run -d -p 80:80 --name my-app $TAG_COMMIT"
  environment:
    name: production
    url: http://your_server_IP
  only:
    - main
```

## Validating the Deployment

Now you’ll validate the deployment in various places of GitLab as well as on your server and in a browser.

When a `.gitlab-ci.yml` file is pushed to the repository, GitLab will automatically detect it and start a CI/CD pipeline. At the time you created the `.gitlab-ci.yml` file, GitLab started the first pipeline.

Go to **CI/CD** > **Pipelines** in your GitLab project to see the pipeline’s status. If the jobs are still running/pending, wait until they are complete. You will see a **Passed** pipeline with two green checkmarks, denoting that the publish and deploy job ran successfully.

![The pipeline overview page showing a passed pipeline](https://assets.digitalocean.com/articles/gitlab_pipeline1804/step7a.png)

Let’s examine the pipeline. Click the **passed** button in the **Status** column to open the pipeline’s overview page. You will get an overview of general information such as:

- Execution duration of the whole pipeline.
- For which commit and branch the pipeline was executed.
- Related merge requests. If there is an open merge request for the branch in charge, it would show up here.
- All jobs executed in this pipeline as well as their status.

Next click the **deploy** button to open the result page of the deploy job.

![The result page of the deploy job](https://assets.digitalocean.com/articles/gitlab_pipeline1804/step7b.png)

On the job result page you can see the shell output of the job’s script. This is the place to look for when debugging a failed pipeline. In the right sidebar you’ll find the deployment tag you added to this job, and that it was executed on your **Deployment Runner**.

If you scroll to the top of the page, you will find the **This job is deployed to production** message. GitLab recognizes that a deployment took place because of the job’s environment section. 

Confirm the website was deployed successfully by visting the `http://<IP FROM SPREADSHEET>` in a browser

Finally we want to check the deployed container on your server. Head over to your terminal, and list the running containers:

```bash
docker ps
```

You should the `my-app` container running

You have now validated the deployment. In the next step, you will go through the process of rolling back a deployment.

## Rolling Back a Deployment

Next you’ll update the web page, which will create a new deployment and then re-deploy the previous deployment using GitLab environments. This covers the use case of a deployment rollback in case of a defective deployment.

Start by making a little change in the `index.html` file:

1. In GitLab, go to the **Project overview** and open the `index.html` file.
2. Click the **Edit** button to open the online editor.
3. Change the file content to the following:

```html
<html>
  <body>
    <h1>My Enhanced Personal Website</h1>
  </body>
</html>
```

Click **Commit changes** to create the file.



A new pipeline will be created to deploy the changes. In GitLab, go to **CI/CD > Pipelines**. When the pipeline has completed, you can open `http://<IP FROM SPREADSHEET>` in a browser for the updated web page now showing **My Enhanced Personal Website** instead of **My Personal Website**.

When you move over to **Deployments > Environments > production** you will see the newly created deployment. Now click the **re-deploy** button of the initial, older deployment:

![A list of the deployments of the production environment in GitLab with emphasize on the re-deploy button of the first deployment](https://assets.digitalocean.com/articles/gitlab_pipeline1804/step8a.png)

Confirm the popup by clicking the **Rollback** button.

The deploy job of that older pipeline will be restarted and you will be redirected to the job’s overview page. Wait for the job to finish, then open `http://<IP FROM SPREADSHEET>` in a browser, where you’ll see the initial headline **My Personal Website** showing up again.

Let’s summarize what you have achieved throughout this tutorial.



## Conclusion

In this tutorial, you have configured a continuous deployment pipeline with GitLab CI/CD. You created a small web project consisting of an HTML file and a Dockerfile. Then you configured the `.gitlab-ci.yml` pipeline configuration to:

1. Build the Docker image.
2. Push the Docker image to the container registry.
3. Log in to the server, pull the latest image, stop the current container, and start a new one.

GitLab will now deploy the web page to your server for each push to the repository.

Furthermore you have verified a deployment in GitLab and on your server. You have also created a second deployment and rolled back to the first deployment using GitLab environments, which demonstrates how you deal with defective deployments.

At this point you have automated the whole deployment chain. You can now share code changes more frequently with the world and/or customer. As a result, development cycles are likely to become shorter, as less time is required to gather feedback and publish the code changes.