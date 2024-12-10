# Deliverable ⓶ Automated deployment: JWT Pizza

🔑 **Key points**

- Deploy JWT Pizza using a continuous integration pipeline

[🎥 Video overview](https://youtu.be/MID8dL4gppE)

---

![course overview](../sharedImages/courseOverview.png)

In your last deliverable you manually built the JWT Pizza frontend, copied it into a branch, and pushed it to GitHub pages. While this process works, it has several problems.

1. **Slow**: It requires a human to execute a series of steps. That slows down your ability to move quickly.
1. **Error-prone**: If a step is skipped or a parameter is misplaced it could cause the deployment to fail, or worse, take down the website entirely.
1. **Non-reproducible**: You have no way of knowing which steps were followed, or who executed them.

Basically, manual processes violate the prime directive of DevOps, that everything should be automated.

So now it is time to take the next step. You are going to use `GitHub Actions` to deploy your static deployment each time a commit is pushed to your fork of the jwt-pizza repository.

## Create the GitHub Action workflow

Here are the steps to make the automation magic happen.

1. Open your fork of `jwt-pizza` on GitHub.com.
1. Change the GitHub Pages option for the _Build and deployment/Source_ to `GitHub Actions`. This causes GitHub to add an environment called `github-pages`. This environment is used to specify deployment rules and secrets that are specific to GitHub action deployments. You will reference this environment in your CI pipeline.

   > ![GitHub Pages Source Actions](gitHubPagesSourceAction.png)

   Changing the option turns off the monitoring of the `gh-pages` branch so that it will no longer trigger a deployment.

1. Modify the `.github/workflows/ci.yml` you created in the previous assignment. This file is the automation script that GitHub actions will execute whenever you make a commit to the `main` branch.
1. Insert the following into the newly created file

   ```yml
   name: CI Pipeline

   on:
     push:
       branches:
         - main
     workflow_dispatch:
   jobs:
     build:
       name: Build
       runs-on: ubuntu-latest
       outputs:
         version: ${{ steps.set_version.outputs.version }}
       steps:
         - name: Checkout repo
           uses: actions/checkout@v4

         - name: Setup Node
           uses: actions/setup-node@v4
           with:
             node-version: '20.x'

         - name: set version
           id: set_version
           run: |
             version=$(date +'%Y%m%d.%H%M%S')
             echo "version=$version" >> "$GITHUB_OUTPUT"
             printf '{"version": "%s" }' "$version" > public/version.json

         - name: Build
           run: |
             npm ci && npm run build
             cp dist/index.html dist/404.html

         - name: Update pages artifact
           uses: actions/upload-pages-artifact@v3
           with:
             path: dist/
     deploy:
       needs: build
       permissions:
         pages: write
         id-token: write
       environment:
         name: github-pages
         url: ${{ steps.deployment.outputs.page_url }}
       runs-on: ubuntu-latest
       steps:
         - name: Deploy to GitHub Pages
           id: deployment
           uses: actions/deploy-pages@v4
   ```

1. Commit and push. This should trigger Vite to bundle the source code and then deploy the `dist` directory to the repository's GitHub Pages URL. 🎉 Wow! Automation is awesome.

## Review of the workflow steps

Let's look at each of the steps in the GitHub Actions workflow.

First off we tell GitHub actions that we want to trigger this workflow whenever a commit is pushed into the main branch. We also add a trigger for when a **workflow dispatch** event is created, which will allow the AutoGrader to trigger your workflow manually or via an API call.

```yml
on:
  push:
    branches:
      - main
  workflow_dispatch:
jobs:
```

Next we define the jobs we want to run.

### Build job

Our first job is named **build**. We specify that it will use the Linux operating system and that the job build is expected to output the version of the build.

```yml
build:
  name: Build
  runs-on: ubuntu-latest
  outputs:
    version: ${{ steps.set_version.outputs.version }}
```

The first step in the **build** job does a checkout of the repository.

```yml
steps:
  - name: Checkout repo
    uses: actions/checkout@v4
```

Then it installs Node.js version 20.

```yml
- name: Setup Node
  uses: actions/setup-node@v4
  with:
    node-version: '20.x'
```

We then create our version number, set it in a GitHub Action output variable that will make it available to other jobs, and persistently store it in a file named `version.json`.

```yml
- name: set version
  id: set_version
  run: |
    version=$(date +'%Y%m%d.%H%M%S')
    echo "version=$version" >> "$GITHUB_OUTPUT"
    printf '{"version": "%s" }' "$version" > public/version.json
```

Next it builds the distribution bundle by installing all the dependencies, running Vite to do the bundling, and then copying over our 404 handler file fix that we introduced in the last deliverable.

```yml
- name: Build
  run: |
    npm ci && npm run build
    cp dist/index.html dist/404.html
```

Finally, it packages up the `dist` directory and stores it as an artifact for the next job to deploy to GitHub Pages.

```yml
- name: Update pages artifact
  uses: actions/upload-pages-artifact@v3
  with:
    path: dist/
```

### Deploy job

The `deploy` job specifies that it needs `build` to complete before it runs. It also provides the permission to act on your behalf in order to modify GitHub Pages for this repository.

```yml
needs: build
permissions:
  pages: write
  id-token: write
```

It then references the `github-pages` environment that was automatically created when you chose to use GitHub Actions to control the source of your GitHub Pages static deployment. This environment isolates what GitHub Pages needs from any other type of deployment environment that you might use.

```yml
environment:
  name: github-pages
  url: ${{ steps.deployment.outputs.page_url }}
```

The final step deploys the contents of the artifact that was created in the previous job. This looks like magic because it used a bunch of default parameters that are tightly coupled with the `upload-pages-artifact`. This includes the name of the artifact and where to deploy the files to.

```yml
- name: Deploy to GitHub Pages
  id: deployment
  uses: actions/deploy-pages@v4
```

## Adding a status badge

GitHub Actions will automatically create an image that shows the state of an Action workflow. You can reference the badge in the `README.md` file found in the root of your fork of `jwt-pizza`.

All you need to do copy the following Markdown, paste it in the README file, replace the account name with your GitHub account name, and commit.

```md
[![CI Pipeline](https://github.com/YOURACCOUNTNAMEHERE/jwt-pizza/actions/workflows/ci.yml/badge.svg)](https://github.com/YOURACCOUNTNAMEHERE/jwt-pizza/actions/workflows/ci.yml)
```

Now whenever you pipeline runs you will be able to see the status in your repos home page on GitHub. Don't worry about the coverage reporting 0%. You will correct that in a future assignment.

![Pipeline status](pipelineStatus.png)

## Wrap up

In about the same amount of code that we used to do our manual deployment we are able to do an automated deployment that is triggered by a git push.

We only showed a simple basic example of what you can do with GitHub Actions, but even that is pretty impressive. You should experiment with what the workflow is doing. It is all checked into git and so it is easy to revert whatever you mess up.

## ☑ Assignment

Complete the steps outlined above by creating a GitHub Actions workflow to deploy the JWT Pizza frontend whenever a change is pushed to the main branch.

Once JWT Pizza is live on your domain, go to the [AutoGrader](https://cs329.cs.byu.edu) and submit your work for the deliverable.

### Rubric

| Percent | Item                                                                                        |
| ------- | ------------------------------------------------------------------------------------------- |
| 60%     | Successful execution of GitHub Actions to deploy JWT Pizza to GitHub Pages on push          |
| 10%     | Version updated with each build and visible in app footer.                                  |
| 35%     | Completely functional JWT Pizza deployed with GitHub Pages accessible on custom domain name |
| 5%      | Pipeline status badge display on your JWT Pizza **README.md** home page                     |
