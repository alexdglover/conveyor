# conveyor
Pipelines are for shit. Conveyors move product. This is a CI/CD platform written in Python


# Topology

Conveyor is meant to be a distributed build system. It is comprised of two
services (Orchestrators and Runners), a relational database, and queue(s).

## Orchestrator

The Orchestrator is the core of Conveyor, and serves multiple purposes:

* web UI for human interaction
* API backend for progammatic interaction
* CRUD operations for jobs

A minimum of one Orchestrator is required, but multiple Orchestrators can be
deployed for high availability if using an external database and queue.

## Runners

The Runners is what actually does the work. Runners are dumb, can be hosted
anywhere (as long as they can reach the database and queue), and are stateless.

## Relational database

The database stores all stateful information for Conveyor, including build
information, credentials, and even the Conveyor configuration. The database is
abstracted by SQLAlchemy - SQLite is used in development, but in 'real'
deployments any SQLAlchemy-supported database _should work_.

### GUIDs

Dealing with IDs and collisions is boring. Conveyor leverages GUIDs where
possible to give referential keys without complicating logic elsewhere.

## Queue(s)

Conveyor uses queues to make jobs available to one or more Runners, while
ensuring each job is executed exactly one time.

By default, Conveyor uses one queue for all Runners. If you would like to run
different builds on different Runners (e.g. Runners with GPUs, Runners with
Docker installed, Runners on a different network) you will need a separate
queue for each "fleet" of Runners.

Queues are abstracted by the use of Celery - Redis is used for local
development and testing, but Redis and SQS will be supported for deployments.

# Logical Objects

## Jobs and Belts

Conveyor's primary focus is to provide inter-operable, modular Jobs that can be
chained together to provide more valuable automation flows. Jobs can execute
any set of arbitrary commands within the job, but they have resolvable inputs
and outputs. This allows for artifacts or state to be passed from one Job to
the next, decision-based workflows, sane retries, and more.

A collection of related Jobs is called a Belt. While each Job is atomically
defined, they can consume inputs/outputs from other jobs. Conveyor evaluates
an entire Belt at once, immediately scheduling any jobs with no dependencies
and registering the remaining jobs.

This concept of chaining jobs is the main differentiator for Conveyor.

### Job Outputs

In order to separate Orchestrators and Runners, Job Outputs must be generic
strings. This allows outputs to be used for serialized data (like JSON/YAML),
file contents (like configuration files), references to external artifacts
(like S3 objects or Docker images). It is each Job's responsibility to be inter-
operable with its upstream and downstream Jobs.

For example, the outputs of a BASH Job are `STDOUT` and
any environment variables set when the session is terminated. In a Docker
build Job, you could build a new Docker image and set the environment variable
`DOCKER_IMAGE` to `my_image:my_tag`. The next Job can then consume string as an
input to create a new Kubernetes pod/ECS service with that image.


## Products

If you're coming from Jenkins, a Product is similar to a 'Project' in Jenkins.
For many users, a Product will be comprised of:

* a single source, like GitHub a repo
* a single trigger, like GitHub pull request or merge
* a single Belts (set of Jobs), for unit testing and producing an artifact

Conveyor supports these types of Products just fine. The entire Product
configuration and Belt definition can be captured in a `.conveyor.yml` file
alongside the source code (much like Travis or other hosted CI tools). But this
is just one layout, and a myopic one at that.

The next evolutionary step would be to simply chain Products together the same
way Jobs are chained together.

For example, let's pretend your organization has a Rails apps that is used for
an API backend. You write Ruby, test it, publish it as a Gem, build a new
AMI, and deploy that AMI to one or more environments with something like
CloudFormation or Terraform. This could easily span 3 or 4 CI platforms,
repositories, and teams.

With Conveyor, each of these layers can be defined as a Product, feeding the
output from one Product into one or more downstream Products. For example, the
repo for the actualy Ruby on Rails code may have a `.conveyor.yml` like this:

```yaml
product: rails-backend
triggers:
  - scm-event:
    repo: git://github.com/rails-backend
    type: pull-request
    belts:
      - pull-request-tests
  - scm-event:
    repo: git://github.com/rails-backend
    type: merge
    belts:
      - build-gem
belts:
  - name: pull-request-tests
    jobs:
      - name: rubocop
        inputs:
          - git:<source_url>
        pre-run: |
          bundle install
        commands: |
          bundle exec rubocop
      - name: rspec
        inputs: source
        pre-run: |
          bundle install
        commands: |
          bundle exec spec
  - name: build-gem
    jobs:
      - name: compile-gem
        inputs:
          - git:<source_url>
        pre-run: |
          bundle install
        commands: |
          bundle exec rake gem:build
        outputs:
          - GEM_VERSION
      - name: publish-gem
        pre-run: |
          bundle install
        commands: |
          GEM_VERSION={{compile-gem.GEM_VERSION}} bundle exec rake gem:publish
        outputs:
          - GEM_VERSION
        downstreams:
            belts:
              - downstream-product: ami
                belt: build-rails-backend-ami

```

In a separate repo, you probably have a series of Packer files, maybe some
variables or code to wrap Packer. The repo has Packer files for multiple
images, not just images for the API backend. Regardless of the specifics, the
process of building the Gem is disjointed from the process of producing a new
AMI. With Conveyor, we can pass relevant information across Jobs/Belts/Products
to unify automation across silos.

Let's look at how we would define our `.conveyor.yml` for the AMI repo:

```yaml
product: ami
triggers: # No triggers required since upstream job will trigger
belts:
  - name: build-ami
    jobs:
      - name: execute-packer
        inputs:
          - git:<source_url>
          - rails-backend # Makes the job object available for reference
        commands: |
          # Explicit Jinja syntax for consuming job outputs is <product>.<belt>.<job>.<output>
          # Conveyor will also 'flatten' the outputs to the parent; a given output is also
          # available via <product>.<output>, but be mindful of namespacing if you using
          # this implied format
          GEM_VERSION={{rails-backend.build-gem.GEM_VERSION}} packer build rails_backend_packer_file.json
        outputs:
          - AMI_ID
```

Next, your Terraform repo would define it's own Product and Belts in a separate
`.conveyor.yml` file. In this example, we're defining an upstream Job as a
trigger, as opposed to declaring the `terraform` project as a downstream. In
other words, bidirectionality is supported. Here's the example:

```yaml
product: terraform
triggers:
  - upstream-build:
    upstream-product: ami
    upstream-belt: build-ami
    build: last # By default, a job will look at the last successful job.
    # Other `build` options include last_successful, last_failure, and explicit
    # build numbers
belts:
  - name: update-autoscaling-group
    jobs:
      - name: build-gem
        inputs:
          - git:<source_url>
          # - ami # Declaring the ami product as an input is unnecessary since
          # this job was triggered by an `ami` Belt
        commands: |
          TF_VAR_ami_id={{ami.build_ami.ami_id}} terraform apply
```
