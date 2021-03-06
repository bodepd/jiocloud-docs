# Introduction

This document is intended to capture the architecture, implementation, and
usage of JioCloud's automated integration, testing, and deployment frameworks.

## What is CI/CD?

CI/CD ( [continuous integration](http://en.wikipedia.org/wiki/Continuous_integration) and [continuous delivery](http://en.wikipedia.org/wiki/Continuous_delivery) ) are
practices through which changes to code repositories are automatically integrated,
verified, and deployed (also called continuous deployment).

# Architecture Overview

The CI/CD system consumes upstream patches, runs their unit tests, and packages
them as a single package repository snapshot that can be expressed by a single
version.

Once a new snapshot version exists, it is run through several independent steps
of a build pipeline before being deployed into the production environment.

A package repository snapshot version is applied against each step in a pipeline.
Each step is only performed on a version once it has already passed all proceeding
steps.

NOTE: some steps may be run in parallel in order to reduce the total amount of time
required to promote a change set through the pipeline, but for our current purposes,
we will consider them to be in entirely in serial.

The final pipeline will have the following stages:

* Acceptance - quick tests that will provision out a full openstack environment using VMs
  for faster turn-around.
* Non-functional tests - Provisions openstack on VMs to validate non-functional requirements
  (such as performance)
* Performance tests - will run benchmarks against the previous and current version of the
  cloud software.
* Upgrade tests - Verifies that the code can upgrade from the previous version, resulting in
  a functional environment.
* Staging tests - Tests that are run in a staging environment that is intended to behave exactly
  the same as the production environment.
* Production deployments - once a snapshot version has been promoted through the previous stages,
  it is ready to be deployed in production.

## Guiding Design principals

It's easier to understand many of the design decisions made for this project
within the context of our guiding design principals.

1. Automate everything - Humans make too many mistakes to have them perform
   any manual repetitive tasks. Automating tasks also results in fewer mistakes
   and allows tasks to scale.

2. No one should ever make any ad hoc modification to our cloud systems. This
   leads to risk that certain systems can wind up in an inconsistent state.
   Machines with inconsistent states are harder to debug at scale and invalidate
   our upgrade testing.

3. Stay as close to master as possible - The closer you are to master, the
   less code you need to deploy to stay up-to-date. This should limit the
   likelihood that any change set will lead to a failure.

4. Tests are the gateway for code getting deployed. We have to fully trust
   our automated tests in order to validate that our services are running
   correctly.

5. Build everything expecting it to have to scale massively. Design decisions
   are made assuming that we eventually have to reach 10s or 100s of thousands
   of managed systems. Because of this, we intend to limit the number of
   centralized services as much as possible and expect each of the individual
   servers of our cloud to take on as much of the processing related to their
   own management as possible.

6. Make the deployments as similar as possible for each environment.

## End to End process:

1. Package build server contains a configuration file that tells it
   what external package and code repositories it should be monitoring
   for changes. Jobs exist that pull in the current version of those
   repositories and run their unit tests. Once the unit tests are validated,
   the latest version of the set of external repositories is packaged
   as a single versioned repository.

2. Package versioning system periodically (every 15 minutes by default)
   monitors the package build server for updates. When updates are detected,
   it builds a new version (snapshot) that represents the latest version of
   all packages (NOTE: snapshots are time based (not patch based) and may
   contain multiple unrelated changes across multiple projects.

3. Jenkins monitors the package versioning system for updates. When updates are
   detected, it initializes a new jenkins job that pushes the latest package
   snapshot version through the build pipeline.

4. The initial job runs what we are calling acceptance tests. These tests
   create a set of virtual machines from scratch and configure them to install
   the desired set of packages. Once the build is verified, tests are run to
   ensure that it results in a functional environment.

5. If this build is successful, the next steps of the pipeline are executed on
   the same snapshot version.

## Deployment process

The above section focused on the high level overview of the build pipeline, but omitted the
details about what the actual environment deployment looks like.

### Deployment script

The same script that is used to provision and validated instances of openstack from jenkins
is available in the [puppet-rjil repo](https://github.com/JioCloud/puppet-rjil/blob/master/build_scripts/deploy.sh).
Specific details about how to use the script by hand for testing can be found in the project's
[README](https://github.com/JioCloud/puppet-rjil/blob/master/README.md).

### Deployment Process

1. The resource file is processed by jiocloud.apply\_resources and results in
all machine required machine being provisioned. This resource file contains
a specification of how many instances of what roles should exist.

2. Each machine that does not already exist will be provisioned along with a
specified userdata script responsible for:

- installing Puppet
- setting up repositories to the correct version
- running a special Puppet class that is responsible for installing a
  pre-defined [cron job](https://github.com/JioCloud/puppet-rjil/blob/master/files/maybe-upgrade.sh).

````
puppet apply --debug -e "include rjil::jiocloud"
````

Currently, this userdata script is included inside of the [deploy.sh](https://github.com/JioCloud/puppet-rjil/blob/master/build_scripts/deploy.sh) file.

3. The desired version for all machines is published into etcd using the command:

In order to achieve this, the deployment script does the following:

* gets the IP address of the etc instance:

````
python -m jiocloud.utils get\_ip\_of\_node etcd1
````

* triggers an update is required by specifying a version in etcd:

````
python -m jiocloud.orchestrate trigger\_update ${BUILD\_NUMBER}
````

* blocks until all hosts are ready

* verifies that no hosts are in a failed state (and fails if this happens)

NOTE: the code for this is not completed yet.

* blocks until all hosts are reported as having a single version installed

3. Each machine now has a running cron job that is responsible for managing the state of that machine.
   To do this, the cron job does the following:
* verifies that it can contact etc and derives is its desired snapshot version is greater than it's current
  version
* If it is greater, it does the following:
    * update the repo sources to use the correct version
    * run apt-get update (to update package sources
    * run apt-get upgrade
    * run puppet (to update other non-package update related config)
    * update version in etcd along with pass/fail information

# ToolChain

## Packages

Package are a central component to the design of this CI/CD system. All code changes
that will be applied to the pipeline are stored and configured as packages.

Packages allow keeping track of all changes, and which specific components
they effect. They also allow the update of only the specific bits of a system
that need to be effected.

## Puppet

Puppet is responsible for configuring of each individual machine into
it's desired roles.

For example, if a machine needed to be configured as a database, Puppet is
responsible for the logic involved in converting a blank machine into this role.

This class assignment is performed by encoding the desired role into a machines
hostname and use Puppet's node expression to match those hostnames into roles.

Puppet will also be used for configuring the jenkins instance using this (repo)[https://github.com/bodepd/puppet-openstack-gater]

## Openstack API

The openstack API will be used for provisioning all virtual machines and
bare-metal servers required for both test environment as well as production.

### Jenkins

Jenkins is responsible for launching all jobs used as a part of the build
pipeline.

It is also responsible for querying upstream repositories for changes, firing off
builds in response to those changes and setting up build steps that need to
be executed serially as a pipeline.

## JioCloud python tools

### python.jiocloud.apply\_resources

The (python-jiocloud)[https://github.com/jiocloud/python-jiocloud]
project contains python helpers tools that are used as a part
of the build process.

#### Apply Resources

The apply resources code is used to manage stacks of VMs
that need to exist in order for a deployment to match the
desired provisioning specification.

#### Apply Command

The apply command is used to evaluate a specification for which
servers/VMs should be provisioned in order to fulfill a specification.

The command is as follows:

````
python -m jiocloud.python apply <resource\_file> <userdata> [--project\tag tag]
````

Arguments:

* resource\_file

A file that list the specifications of the stack to be created.

#### Delete Command

The delete command is used to delete specific VMs/servers related to a specific
stack.

### list

Given a project tag and a resource file, lists the specified servers.

## Installation and Updates

The core infrastructure for running CI/CD is installed and
updated using Puppet and shares as much code and process as possible
with the existing openstack-infra project.

## Developing on CI/CD framework

It is recommended that you test all changes for the CI/CD framework
locally to avoid the likelihood of pushing code that may disrupt the
function of the CI/CD system.

## CI/CD components


## Build workflow
