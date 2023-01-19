release time :2022-02-25 11:54

Create a React projectgithub-action-demo

Will (replace backendcloud with your github account name)

    "homepage": "https://backendcloud.github.io/github-actions-demo",


Add to package.json

push to the github repository.

Add the github aciton CI configuration file: the process is generally that once the master branch is monitored to have a push, then checkout the project in the virtual environment, build the React project, and deploy the static file to the gh-pagesbranch of the code warehouse.

    hanwei@hanweideMacBook-Air github-actions-demo]$ cat .github/workflows/ci.yml 
    name: GitHub Actions Build and Deploy Demo
    on:
    push:
        branches:
        - master
    jobs:
    build-and-deploy:
        concurrency: ci-${{ github.ref }} # Recommended if you intend to make multiple deployments in quick succession.
        runs-on: ubuntu-latest
        steps:
        - name: Checkout 🛎️
            uses: actions/checkout@v2

        - name: Install and Build 🔧 # This example project is built using npm and outputs the result to the 'build' folder. Replace with the commands required to build your project, or remove this step entirely if your site is pre-built.
            run: |
            npm ci
            npm run build

        - name: Deploy 🚀
            uses: JamesIves/github-pages-deploy-action@v4.2.5
            with:
            branch: gh-pages # The branch the action should deploy to.
            folder: build # The folder the action should deploy.
    hanwei@hanweideMacBook-Air github-actions-demo]$ 



Push the code to the github code warehouse again, trigger the github action process, and check the github action log (the github website is saved for 30 days):

    deploy
    succeeded 17 minutes ago in 4s
    Search logs
    1s
    Current runner version: '2.287.1'
    Operating System
    Virtual Environment
    Virtual Environment Provisioner
    GITHUB_TOKEN Permissions
    Secret source: Actions
    Prepare workflow directory
    Prepare all required actions
    Getting action download info
    Download action repository 'actions/deploy-pages@v1' (SHA:479e82243a95585ebee8ab037c4bfd8e6356a47b)
    0s
    Run actions/deploy-pages@v1
    Sending telemetry for run id 1896726019
    3s
    Run actions/deploy-pages@v1
    Actor: github-pages[bot]
    Action ID: 1896726019
    Artifact URL: https://pipelines.actions.githubusercontent.com/7IW4xoAefje3iJJjmBQg228AYglZEwNoROQmK16mg2iViZ9muv/_apis/pipelines/workflows/1896726019/artifacts?api-version=6.0-preview
    {"count":1,"value":[{"containerId":3420907,"size":614400,"signedContent":null,"fileContainerResourceUrl":"https://pipelines.actions.githubusercontent.com/7IW4xoAefje3iJJjmBQg228AYglZEwNoROQmK16mg2iViZ9muv/_apis/resources/Containers/3420907","type":"actions_storage","name":"github-pages","url":"https://pipelines.actions.githubusercontent.com/7IW4xoAefje3iJJjmBQg228AYglZEwNoROQmK16mg2iViZ9muv/_apis/pipelines/1/runs/2/artifacts?artifactName=github-pages","expiresOn":"2022-05-26T03:47:37.6013822Z","items":null}]}
    Creating deployment with payload:
    {
        "artifact_url": "https://pipelines.actions.githubusercontent.com/7IW4xoAefje3iJJjmBQg228AYglZEwNoROQmK16mg2iViZ9muv/_apis/pipelines/1/runs/2/artifacts?artifactName=github-pages&%24expand=SignedContent",
        "pages_build_version": "f018db912dfb491e86183fbc140fe14e3a256905",
        "oidc_token": "***"
    }
    Created deployment for f018db912dfb491e86183fbc140fe14e3a256905
    {"page_url":"https://backendcloud.github.io/github-actions-demo/","status_url":"https://api.github.com/repos/backendcloud/github-actions-demo/pages/deployment/status/f018db912dfb491e86183fbc140fe14e3a256905"}

    Current status: updating_pages
    Reported success!

    Sending telemetry for run id 1896726019
    0s
    Evaluate and set environment url
    Evaluated environment url: https://backendcloud.github.io/github-actions-demo/
    Cleaning up orphan processes



After the CI/CD process is completed, open it with a browser https://backendcloud.github.io/github-actions-demo/and find that the React project is successfully deployed, as shown below:

![](2023-01-19-13-15-38.png)

GitHub has two recent changes:

1 is used for github.io is no longer the default gh-pagesbranch, which branch to use for static web pages and content needs to be configured in advance. If not configured, an error https://backendcloud.github.io/github-actions-demo/will be reported when opening .404

![](2023-01-19-13-15-52.png)

2 is that the user name and password authentication has been canceled Githubfor developers to call Github API, and only more secure token authentication can be used.


![](2023-01-19-13-16-06.png)

