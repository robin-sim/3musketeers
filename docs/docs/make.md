# Make

Having a clean `Makefile` is key. It helps to understand it quickly and is easier to maintain. Therefore, having some conventions like [target vs _target][linkTargetVSUnderscoreTarget], and [Pipeline targets][linkPipelineTargets] really aim to make the developer's life easier. The conventions are for the context of the 3 Musketeers.

::: tip SNIPPETS
The snippets are there to help understanding the documentation but may be incomplete or missing context. If you wish to see complete code, go over the [examples][linkExamples] section.
:::

## target vs _target

Using `target` and `_target` is a naming convention to distinguish targets that can be called on any platform (Windows, Linux, MacOS) versus those that need specific environment/dependencies.

```makefile
# buid target uses Compose which is available on Windows, Unix, and MacOS
deploy:
	docker-compose run --rm serverless make _deploy

# This _deploy target depends on a NodeJS environment and Serverless Framework which may not be available on the hosts.
# It is executed in a Docker container that provides the right environment for its execution.
# If the host has NodeJS and Serverless Framework installed, `$ make _build` can be called.
_deploy:
	serverless deploy
```

## .PHONY

> A phony target is one that is not really the name of a file; rather it is just a name for a recipe to be executed when you make an explicit request. There are two reasons to use a phony target: to avoid a conflict with a file of the same name, and to improve performance. - [GNU][linkPhony]

By default `.PHONY` can be left out unless the target collides with the name of file/folder. One case is a target `test` which often conflicts with a folder named `test`.

```makefile
# can be written into a single line with other targets that need .PHONY
.PHONY: test targetA targetB

test:
	docker-compose run --rm node make _test
.PHONY: test # can be put here for the target test

_test:
	npx jest
```

Sometimes you want the target to match the name of a file in which case `.PHONY` would not be used. See [Environment variables & .env file][linkEnvironmentVariables] for an example.

## Docker and Compose commands as variables

Docker and Compose commands can be assigned to variables.

```makefile
COMPOSE_RUN_SERVERLESS = docker-compose run --rm serverless

deploy:
	$(COMPOSE_RUN_SERVERLESS) make _deploy
```

## Target dependencies

To make the Makefile easier to read, avoid having many target dependencies: `target: a b c`. Restrict the dependencies only to `target` and not `_target`. Even more, restrict `target` to file dependencies only. This allows one to call a specific target without worrying that other targets will be executed too.

::: tip
Use [Pipeline targets][linkPipelineTargets] as a way to describe the list of dependencies.
:::

```makefile
deploy: package.zip
	$(COMPOSE_RUN_NODE) make _deploy
```

## Pipeline targets

Section [Target dependencies][linkTargetDependencies] suggests to limit target dependencies as mush as possible but there is one exception: pipeline targets.

We call Pipeline targets the targets that have a list of dependencies, usually other targets. They are often used in CI to reduce the number of Make calls and keep the CI pipelines clean.

It is best having them at the top of the Makefile as they give an understanding of the application pipelines when reading the Makefile.

```makefile
# pipeline targets first
stageTest: build test clean

# other targets below
```

## make and target all

Running only `$ make` will trigger the first target from the Makefile. A convention among developer is to have a target `all` as the first target. In the 3 Musketeers context, `all` is a perfect [pipeline target][linkPipelineTargets] to document and test locally the sequence of targets to test, build, run, etc.

```makefile
# first target
all: deps test build clean

# other targets below
```

```sh
$ make # will run the target all
```

## Target and Single Responsibility

It is a good idea to make the target as focus as possible on a specific task. This leaves the flexibility to anyone to manually test/call each target individually for a single purpose.

Targets can be composed as a [pipeline target][linkPipelineTargets] which ensures the right order of targets to call for executing a specific task.

## Targets .env and envfile

The target `envfile` creates the file `.env` which is very useful for a project that follows the 3 Musketeers pattern. See [Environment Variables & envfile][linkEnvironmentVariables] for more details.

## Project dependencies

It is a good thing to have a target `deps` for installing all the dependencies required to test, build, and deploy an application.

A tar file of the dependencies can be created as an artifact to be passed along through the CI/CD stages. This step is useful as it acts as a cache and means subsequent CI/CD agents don’t need to re-install the dependencies again when testing and building. Moreover, it is faster to pass along a tar file than a folder with many files.

```makefile
COMPOSE_RUN_NODE=docker-compose run --rm node
NODE_MODULES_DIR=node_modules
NODE_MODULES_ARTIFACT=$(NODE_MODULES_DIR).tar.gz

# deps will create the folder node_modules and the file node_modules.tar.gz
deps:
	$(COMPOSE_RUN_NODE) make _depsNode _packNodeModules

# test requires the folder node_modules
test: $(NODE_MODULES_DIR)
	$(COMPOSE_RUN_NODE) make _test
.PHONY: test

# if folder node_modules exist, do nothing
# if folder node_modules does not exist, unpack file node_modules.tar.gz
$(NODE_MODULES_DIR):
	$(COMPOSE_RUN_NODE) make _unpackNodeModules

_depsNode:
	npm install

_packNodeModules:
	rm -f $(NODE_MODULES_ARTIFACT)
	tar czf $(NODE_MODULES_ARTIFACT) $(NODE_MODULES_DIR)

_unpackNodeModules: $(NODE_MODULES_ARTIFACT)
	rm -fr $(NODE_MODULES_DIR)
	tar -xzf $(NODE_MODULES_ARTIFACT)
```

## Calling multiple targets in a single command

Make allows you to call multiple targets in a single command like this `$ make targetA targetB targetC`. This is useful if you want to use a different `.env` file and call another target

```bash
# create .env with the default
$ make envfile anotherTarget
# create .env from another file
$ make envfile anotherTarget ENVFILE=your.envfile
```

## Prevent echoing the command

The symbol `@` prevents the command to be printed out prior its execution. Useful when there are secrets at stake.

```makefile
# If '@ 'is omitted, `DOCKERHUB_TRIGGER_URL`, which has a token in the URL,
# would be printed out in the logs
_triggerDockerHubBuild:
	@curl -H "Content-Type: application/json" --data '{"docker_tag": "latest"}' -X POST $(DOCKERHUB_TRIGGER_URL)
```

## Continue on error

The symbol `-` allows the execution to continue even if the command failed.

```makefile
TAG=v1.0.0

# _tag creates a new tag and fails if the tag already exists
_tag:
	git tag $(TAG)
	git push origin $(TAG)

# _overwriteTag creates a new tag or re-tags the existing one
_overwriteTag:
	-git tag -d $(TAG)
	-git push origin :refs/tags/$(TAG)
	git tag $(TAG)
	git push origin $(TAG)
```

## Clean Docker and files

Using Compose creates a network that you may want to remove after your stage or pipeline is completed. You may also want to remove existing stopped and running containers. Moreover, files and folders that have been created can also be cleaned up after. A pipeline would maybe contain a stage clean or call clean after `test` for instance: `$ make test clean`.

`clean` could also have the command to clean Docker. However having the target `cleanDocker` may be very useful for targets that want to only clean the containers. See section [Managing containers in target][linkManagingContainersInTarget].

It may happen that you face a permission issue like the following

```
rm -fr bin vendor
rm: cannot remove ‘vendor/gopkg.in/yaml.v2/README.md’: Permission denied
```

This happens because the creation of those files was done with a different user (in a container as root) and the current user does not have permission to delete them. One way to mitigate this is to call the command in the docker container.

```makefile
cleanDocker:
	docker-compose down --remove-orphans

clean:
	$(COMPOSE_RUN_GOLANG) make _clean
	$(MAKE) cleanDocker

_clean:
	rm -fr files folders
```

## Managing containers in target

Sometimes a target needs to run a container in order to execute its task.

For instance, a target `test` may need a database to run prior executing the tests.

```makefile
# target test calls cleanDocker before starting a postgres container
test: cleanDocker startPostgres
	$(DOCKER_RUN_NODE) make _test
	# cleanDocker stops the postgrest container and removes it
	$(MAKE) cleanDocker
.PHONY: test

postgresStart:
	docker-compose up -d postgres
	sleep 10

cleanDocker:
	docker-compose down --remove-orphans
```

## Multiple Makefiles

The Makefile can be split into smaller files.

```makefile
# makefiles/deploy.mk
deploy:
	docker-compose run --rm serverless make _deploy

_deploy:
	serverless deploy
```

```makefile
# Makefile
include makefiles/*.mk
```

## Complex targets

In some situations, targets become very complex due to the syntax and limitations of Make or you may simply prefer to write the task in Bash or other languages. Refer to the [patterns][linkPatterns] section for other Make alternatives.

## Self-Documented Makefile

[This][linkSelfDocumentedMakefileGist] is pretty neat for self-documenting the Makefile.

```makefile
# Add the following 'help' target to your Makefile
# And add help text after each target name starting with '\#\#'
DOCKER_RUN_MUSKETEERS = docker run --rm -v $(PWD):/opt/app -w /opt/app flemay/musketeers

help:           ## Show this help.
	$(DOCKER_RUN_MUSKETEERS) make _help

_help:
	@fgrep -h "##" $(MAKEFILE_LIST) | fgrep -v fgrep | sed -e 's/\\$$//' | sed -e 's/##//'

# Everything below is an example

target00:       ## This message will show up when typing 'make help'
	@echo does nothing

target01:       ## This message will also show up when typing 'make help'
	@echo does something

# Remember that targets can have multiple entries (if your target specifications are very long, etc.)
target02:       ## This message will show up too!!!
target02: target00 target01
	@echo does even more
```

```bash
# Output
help: Show this help.
target00: This message will show up when typing 'make help'
target01: This message will also show up when typing 'make help'
target02: This message will show up too!!!
```


[linkPipelineTargets]: #pipeline-targets
[linkTargetVSUnderscoreTarget]: #target-vs-_target
[linkTargetDependencies]: #target-dependencies
[linkManagingContainersInTarget]: #managing-containers-in-target

[linkPatterns]: patterns
[linkEnvironmentVariables]: environment-variables
[linkExamples]: ../examples/

[linkPhony]: https://www.gnu.org/software/make/manual/html_node/Phony-Targets.html
[linkSelfDocumentedMakefileGist]: https://gist.github.com/prwhite/8168133
