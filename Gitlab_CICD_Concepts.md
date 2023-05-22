# Terminology
## Continuos Integration
Developers continuoly integrates their code with the main branch as often as possible.

## Continuous Delivery
Pipeline is automated only until the non-production environment.
Update to production happens only with a manual approval.

## Continuous Deployment
Pipeline is completelu automated, code is deployed to the production environment on successfull testing.

![](Pasted%20image%2020230519205430.png)

# Gitlab CI/CD Pipeline
## Configure a pipeline
- `.gitlab-ci.yml` contains the whole CI/CD pipeline configuration in YAML format.
- pipelines are configured for a specific project.
- Once you create the `.gitlab-ci.yml` file in repo root and commit, gitlab will automatically detect and run the pipeline.
- on every commit, gitlab triggers the pipeline automatically.
- 
# Pipeline building blocks
## Jobs
Jobs are the most fundamental building block of pipelines.
Jobs define what to do.
you can define arbitrary names for your jobs.
Must contain at least the script clause
script specify the commands to execute.

Pipeline example:
![](Pasted%20image%2020230520135338.png)
```yml
run_tests:
	before_script:
		- echo "preparing the data.."
	script:
		- echo "running tests.."
build_image:
	script:
		- echo "Building the docker image"
		- echo "tagging the docker image"
push_image:
	script:
		- echo "Logging into docker image"
		- echo "Pushing docker images to docker registry."
```

### script
Used to run any command in the pipeline.
**Inline commands**
- OS commands, like unix commands
```sh
script:
	- ls
	- pwd
	- rm test.txt
	- mkdir testdir
```
- Execute a script file
```sh
script:
	- chmod +x ./prepare-test.sh
	- ./prepare-test.sh
```
**Note on pipeline execution:** Every time pipeline runs in a new fresh environment, so all the data with in pipeline is ephemeral.

### before_script
Define commands that should run before the script commands.

### after_script
Define commands that should run after the jobs, including failed jobs.

### Dependency between jobs
- In general jobs with in a stage run in parallel.
- `needs` keyword define dependency between the jobs

```yml
stages:
	- test
	- build
	- deploy
run_unit_tests:
	stage: test
	before_script:
		- echo "preparing the data.."
	script:
		- echo "running tests.."
run_lint_tests:
	stage: test
	before_script:
		- echo "preparing the data.."
	script:
		- echo "running tests.."

build_image:
	stage: build
	script:
		- echo "Building the docker image"
		- echo "tagging the docker image"
push_image:
	stage: build
	needs: 
		- build_image
	script:
		- echo "Logging into docker image"
		- echo "Pushing docker images to docker registry."
deploy_app:
	stage: deploy
	script:
		- echo "Deploy app to server."
```
### Define when to run a job
Pipeline configuration applies to all the branches, so pipeline runs on every code commit on all the branches.
**`only/except`** Job keywords are used to define when to run a job.
**only** define when to run a job
**except** define when not to run a job
```yml
stages:
	- test
	- build
	- deploy
run_unit_tests:
	only:
		- main
	stage: test
	before_script:
		- echo "preparing the data.."
	script:
		- echo "running tests.."
run_lint_tests:
	only:
		- main
	stage: test
	before_script:
		- echo "preparing the data.."
	script:
		- echo "running tests.."
```
## Stages
- Multiple jobs can be grouped in to stages, that run in a defined order.
- Multiple jobs in the same stage run in parallel.
- Only if all jobs in a stage succeed, the pipeline moves on to next stage.
- If any job in a stage fails, the next stage is not executed and the pipeline ends.
```yml
stages:
	- test
	- build
	- deploy
run_unit_tests:
	stage: test
	before_script:
		- echo "preparing the data.."
	script:
		- echo "running tests.."
run_lint_tests:
	stage: test
	before_script:
		- echo "preparing the data.."
	script:
		- echo "running tests.."

build_image:
	stage: build
	script:
		- echo "Building the docker image"
		- echo "tagging the docker image"
push_image:
	stage: build
	script:
		- echo "Logging into docker image"
		- echo "Pushing docker images to docker registry."
deploy_app:
	stage: deploy
	script:
		- echo "Deploy app to server."
```
Note: when no stages are defined, gitlab assigns all the jobs to the default stage called test.

## Workflow rules
- Control whether or not the whole pipleine is executed.
- "rules" can also be used at the job level as a replacement for only/except.

```yml
#Pipeline runs only for the main branch.
workflow:
	rules:
		- if: $CI_COMMIT_BRANCH != 'main'
		  when: never
		- when: always  
stages:
...
...
```

## Trigger pipeline on merge request
- Run the test on merge requests.
- Once branch merged into main, execute the complete pipeline.
```yml
#Pipeline runs only on main branch or merge request
workflow:
	rules:
		- if: $CI_COMMIT_BRANCH != 'main' && $CI_PIPELINE_SOURCE != "merge_request_event"
		  when: never
		- when: always  
stages:
#configure only -main for other stages except test. 
...
...

```
# Variables
## Predefined
Gitlab provides predefined variables across the environment.

Examples:
$CI_COMMIT_BRANCH  Current branch

## Custom variables
Gitlab -> Project -> Settings -> CICD -> variables (Type - variable)

## File variables
Gitlab -> Project -> Settings -> CICD -> variables (Type - file)
![](Pasted%20image%2020230521172616.png)
The Value (content) can be a simple key valur pair, or a YAML or a JSON content.
The content is saved to a temporary file.
Value of the variable `$PROPERTIES_FILE` is the path to the temporary file.

## Local variables (variables defined within the pipeline - .gitlab-ci.yml)
- Variable defined at the top level of the file: gloablly available and all jobs can use it.
- Variable defined in a job: only that job can use it.
```yml
workflow:
	rules:
		- if: $CI_COMMIT_BRANCH != 'main' && $CI_PIPELINE_SOURCE != "merge_request_event"
		  when: never
		- when: always  
stages:
#configure only -main for other stages except test. 
...
...
variables:
	image_repository: docker.io/my-docker-id
	image_tag: v1.0

job1:
	script:
		- echo "Docekr image to push $image_repository:$image_tag"
...
...
```

# Gitlab Architecture and Execution Flow
![](Pasted%20image%2020230522125231.png)
**Execution flow**
1. Runner requests new jobs from GitLab instance (e.g. GitLab.com)
2. Runner compiles and sends the job's payload to Executor
3. Executor clones sources or downloads artifacts from GitLab instance and executes the job
4. Executor returns job output and status to the Runner
5. Runner updates job output and status to GitLab instance

Gitlab has two options,
- SaaS - Managed by Gitlab, i.e gitlab.com
- Self-managed
- Runners are provided by gitlab that are shared or self-managed.

## Gitlab (Gitlab server)
- Main component
- Pipeline configuration
- Manages the pipeline execution
- Stores the pipeline results

## Gitlab runners
- Program that you should install on a machine, that separate from one that hosts the gitlab instance.
- Gitlab runners are the agents that run CI/CD pipeline jobs.
- Gitlab server assigns these jobs to the available runners.

### Executors
![](Pasted%20image%2020230521175414.png)
**Shell executor**
- The simplet executor
**Disadvantages** 
	- All the required programs for executing the jobs,need to be installed manually.
	- ex: if you want to run docker command, docker need to be installed first.
	- No clean build environment for every job execution, needs manual cleanup.

**Alternative executors**
There are several executors available, and you can choose your executor, while registering a runner.

#### Docker Executor
- Commands are executed inside a container.
- Jobs run on user provided docker images
- All the needed tools can be put in the container image.
#### K8s executor
- Allows you to use an existing k8s cluster for your builds.
- K8s executor will create a new  pod for each gitlab job.
#### Docker machine executor
- Special version of the docker executor.
- supports auto-scaling.
- Managing layer over docker
- Let's you create Docker hosts on your computer,  on cloud providers on demand, dynamically.   
- Creates servers, installs Docker on them and configures the client to talk to them   
- Docker Machine executor works like the normal Docker executor.
- Gitlab's shared runners are using docker machine executor.
- Docker machine has been deprecated by docker.
- Gitlab may replace it in the future.
#### SSH Executor
- least supported among all executors.
- It makes gitlab connect to an external server and run jobs there.
#### Parallel executor
- Parllel is one of the two virtual machine executors, other is virtual box.
### Selecting the executor
