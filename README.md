# Jenkins demo setup for Merkely

## Components

### Jenkins

https://jenkins.merkely.com/ is a Jenkins instance used to run the pipelines for this demo

### Merkely

There is *demo* organization at https://app.merkely.com/ - please reach out to us if you can't see that organization when logged in.

### Infrastructure

Jenkins is hosted on a Virtual Machine in Merkely's Google Cloud. It runs as a docker container. Same VM serves as the only build agent for that Jenkins.  

In the same Google Cloud we set up k8s cluster and configured Jenkins VM's to be able to connect to that cluster ( `gcloud container clusters get-credentials (...)` ).

### Demo Project

The project we've created for this demo consists of [Dockerfile](Dockerfile) we use to build the image and [k8s/deployment.yaml](k8s/deployment.yaml) we use to deploy it to our cluster.

We're building a simple web server that display date and time of the last build of the project. Once the new version is deployed to the cluster it is available at [http://35.195.53.178/](http://35.195.53.178/)

## How to use Merkely with Jenkins

In order to be able to communicate with Jenkins from your pipeline you need to download [Merkely CLI](https://github.com/merkely-development/cli). You can find the latest version of the CLI under [Releases](https://github.com/merkely-development/cli/releases) - choose the one that fits your architecture.  
If your prefer to use doocker image with CLI preinstalled you can find these under [Packages/merkely-cli](https://github.com/orgs/merkely-development/packages/container/package/merkely-cli)

Documentation for latest released version of the CLI is always available at [docs.merkely.com](https://docs.merkely.com/)

### Flags vs environment variables 

Most of Merkely CLI commands require certain amount of flags, the list can be quite long in some cases.  

Let's have a look at reporting artifact creation as an example:

#### Synopsis


   Report an artifact creation to a pipeline in Merkely. 
   The artifact SHA256 fingerprint is calculated and reported 
   or,alternatively, can be provided directly. 
   The following flags are defaulted as follows in the CI list below:

```
merkely pipeline artifact report creation ARTIFACT-NAME-OR-PATH [flags]
```

#### Options

```
  -t, --artifact-type string   The type of the artifact. Options are [dir, file, docker]. Only required if you don't specify --sha256
  -b, --build-url string       The url of CI pipeline that built the artifact. (default "https://github.com/merkely-development/cli/actions/runs/1633529383")
  -u, --commit-url string      The url for the git commit that created the artifact. (default "https://github.com/merkely-development/cli/commit/640b7926a585aee68a6ccec17ffa96c860c130ee")
  -C, --compliant              Whether the artifact is compliant or not. (default true)
  -d, --description string     [optional] The artifact description.
  -g, --git-commit string      The git commit from which the artifact was created. (default "640b7926a585aee68a6ccec17ffa96c860c130ee")
  -h, --help                   help for creation
  -p, --pipeline string        The Merkely pipeline name.
  -s, --sha256 string          The SHA256 fingerprint for the artifact. Only required if you don't specify --artifact-type.
```

### #Options inherited from parent commands

```
  -a, --api-token string      The merkely API token.
  -c, --config-file string    [optional] The merkely config file path. (default "merkely")
  -D, --dry-run               Whether to send the request to the endpoint or just log it in stdout.
  -H, --host string           The merkely endpoint. (default "https://app.merkely.com")
  -r, --max-api-retries int   How many times should API calls be retried when the API host is not reachable. (default 3)
  -o, --owner string          The merkely user or organization.
  -v, --verbose              
```

Some of the flags are optional (`description`), some are only required if another flag is not set (either `artifact-type` or `sha256`) but most you need to provide.  
Once you start using Merkely you will quickly notice some of the flags ALWAYS have the same value (likely `api-token` and `owner`), some stay the same within one pipeline (likely `pipeline` or `artifact-type`).

In order to help you manage such cases, merkely CLI supports environment variables. Every flag for `merkely` command can be represented as one. 

If you know you'll use the same option value for many commands - like owner, or pipeline name, you can use an environment variable to define that option.
To represent an option as an environment variable that will be recognised by Merkely client, you need to create a variable that has the same name as the option (skipping starting double hyphens '--'), capital letters, with "MERKELY_" added at the beginning, all remaining hyphens replaced by underscores.

E.g.  
`--owner` would become `MERKELY_OWNER`  
`--artifact-type` would become `MERKELY_ARTIFACT_TYPE`  
etc

It makes sense to define some environment variables globally - like owner, or host if you're using separately hosted Merkely instance.  
In Jenkins you do it at **Manage Jenkins** > **Configure System** > **Global properties** > **Environment variables** 

Some are pipeline specific so they are defined as part of the pipeline, as we do in our [Jenkinsfile](Jenkinsfile):

```json
pipeline {
    agent any
    environment {
        MERKELY_API_TOKEN = credentials('merkely-api')
        GITHUB = credentials('github')
        MERKELY_CLI_VERSION = "1.1.0"
        
        MERKELY_PIPELINE = "docker-test"
        MERKELY_ENVIRONMENT = "staging-k8s"

        DOCKERHUB_PAT = 'dockerhub-pat'
        DOCKER_ORG = "ewelinawilkosz"
        IMAGE_NAME = "time-server"
        DOCKER_IMAGE_NAME = "$DOCKER_ORG/$IMAGE_NAME"
        DOCKER_IMAGE = ''
    }

    (...)
}
```

There is more environment variables we use in the pipeline, but from the example above, the ones that will be interpreted by Merkely CLI are:  
`MERKELY_API_TOKEN` instead if using the --api-token flag in every `merkely` command
`MERKELY_PIPELINE` instead of using `--pipeline` in pipeline creation and artifact and evidence reporting
`MERKELY_ENVIRONMENT` instead of using `--environment` in reporting deployment command, etc.

Some parameters will be differen for each run, like commit sha and build url - Jenkins provides most of the information Merkely may need as environment variables (`BUILD_URL` or `GIT_COMMIT`)

Look around both Jenkinsfiles in this repository to see how we used environment variables in out setup.

### Basic workflow

There are two workflows: 

* Build
* Test
* Approve
* Deploy
* [pull request]

There is an optional approval control ...
