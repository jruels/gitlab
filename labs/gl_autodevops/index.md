## AUTO DEVOPS WITH A PREDEFINED PROJECT TEMPLATE

We will use a pre-defined template for **NodeJS Express** to show how Auto DevOps works.

### Create a new Node JS Express project with Auto DevOps

Create a new private project using the **NodeJS Express** template

For the project name use `Auto DevOps-test`

Auto DevOps is an alternative to writing and using your own `.gitlab-ci.yml` file. Note the banner at the top of the window alerting you to enable Auto DevOps. Click it and enable **Default to Auto DevOps pipeline**.  Click **Save changes**.

Create a new branch named `new-feature`

Back in **CI/CD > Pipelines**. You’ll see a pipeline with the **Auto DevOps** label, which is running on the branch you just created.

Select the pipeline’s **running** status icon and note the stages (represented by the columns in the pipeline graph) that Auto DevOps has created.

### Commit a change to trigger a pipeline run

The most common way to run a pipeline is to commit to a branch of your project’s repository. You'll do that next.

In the **new-feature** branch, edit the `views/index.pug` file so the last line says:

```
  p GitLab welcomes you to #{title}
```

**NOTE: include the leading spaces**

Commit the change with a message of `Update welcome message in index.pug`

1. For **Commit message**, type `Update welcome message in index.pug`

2. Leave **Target branch** set to `new-feature`

3. Select **Commit changes**.

4. Select **Create merge request**

5. Add `Draft:` to the beginning of the text in the **Title** field to show that the merge request isn’t ready to be merged yet.

6. Assign the merge request to yourself.

7. Leave all other fields at their default values and select **Create merge request** at the bottom of the page.

8. To mark the merge request ready to merge, select **Mark as ready**. This removes `Draft:` from your MR’s title.

   You now have an active merge request for merging the `new-feature` branch into the `master` branch. The page you are on shows the details of that merge request, including the status of the last pipeline that was run on the `new-feature` branch (you might have to refresh the page to see the pipeline status). GitLab will run a new pipeline every time you commit to the `new-feature` branch.

9. The **Review** stage of the Auto DevOps pipeline deploys your NodeJS Express application into a review environment dedicated to this branch. You can see the status of each pipeline stage by hovering over the circular icons in the pipeline overview tab. Once the pipeline has completed the **Review** stage, view the deployed application by selecting **View app** in the middle of the merge request. You should see the text that you modified. Note: You may receive a page with the error "Your connection is not private." This is normal behavior as the SSL certificationss in the GitLab demo cloud may expire. Select **Advanced** and choose **Proceed**.

10. Return to the GitLab browser tab. In the left-hand navigation pane, select **Packages & Registries > Container Registry**. You should see the Docker container that the Auto DevOps pipeline created when it deployed your application to the review environment.