In our [previous](https://dev.to/raigaurav/getting-ready-for-production-jio) article we have created a Dockerfile for our [Mojolicious](https://mojolicious.org/) application. In there I mentioned that we have to run several docker command and when we map it across different environment we have to take care of different permutations and combinations.
What if we can automate it more. What if we can abstract the complexity and as a user I just have to run minimum command to get it working.
Here comes the Makefile. We will be using this to make our process more easier. For windows you can use `gmake.exe` utility instead of `make`. It comes with Strawberry Perl.
Lets create a Make file in `mojo_react_app/Makefile`
```Makefile
DOCKER_TAG ?= mojo_react_app:development
NO_CACHE ?= false
PULL ?= false
BUILD_TYPE ?= development
CONTAINER_NAME ?= mojo_react_app
HTTPS_OUT_PORT ?= 443
HTTPS_IN_PORT ?= 443
HTTP_OUT_PORT ?= 80
HTTP_IN_PORT ?= 80
DOCKER_REPO ?= <docker_repo_url>
ADDRESS ?= 0.0.0.0
DOCKER_CFG = <path to docker config on machine>

build:
	# Build the dockerfile
	docker build --pull=${PULL} --no-cache=$(NO_CACHE) -t $(DOCKER_TAG) .

create:
	# Create the container and copy the certificates
	echo 'Creating container for ${ADDRESS}'
	docker create --name ${CONTAINER_NAME}_${BUILD_TYPE} -t \
		-p ${ADDRESS}:$(HTTPS_OUT_PORT):$(HTTPS_IN_PORT) -p ${ADDRESS}:$(HTTP_OUT_PORT):$(HTTP_IN_PORT) --env mode=${BUILD_TYPE} \
		${DOCKER_TAG}
	docker cp ${DOCKER_CFG}/mojo_react_app/apache-certificate/apache_certificate_${BUILD_TYPE}.crt ${CONTAINER_NAME}_${BUILD_TYPE}:/etc/ssl/certs/server.crt
	docker cp ${DOCKER_CFG}/mojo_react_app/apache-certificate/apache_certificate_${BUILD_TYPE}.key ${CONTAINER_NAME}_${BUILD_TYPE}:/etc/ssl/private/server.key

start:
	# Start a container
	docker start ${CONTAINER_NAME}_${BUILD_TYPE}

run:
	# Create and start the container
	make create -e ADDRESS=${ADDRESS} HTTPS_OUT_PORT=${HTTPS_OUT_PORT} HTTPS_IN_PORT=${HTTPS_IN_PORT} BUILD_TYPE=${BUILD_TYPE} DOCKER_TAG=${DOCKER_TAG}
	make start -e BUILD_TYPE=${BUILD_TYPE}
	make show

stop:
	# Stop a running container
	docker stop ${CONTAINER_NAME}_${BUILD_TYPE};

clean_container:
	# remove previous container
	docker rm -f ${CONTAINER_NAME}_${BUILD_TYPE} 2>/dev/null \
	&& echo 'Container for "${CONTAINER_NAME}_${BUILD_TYPE}" removed.' \
	|| echo 'Container for "${CONTAINER_NAME}_${BUILD_TYPE}" already removed or not found.'

clean_image:
	# remove created image
	docker rmi ${DOCKER_TAG} 2>/dev/null \
	&& echo 'Image(s) for "${DOCKER_TAG}" removed.' \
	|| echo 'Image(s) for "${DOCKER_TAG}" already removed or not found.'

dev: 
	make build -e PULL=${PULL} NO_CACHE=${NO_CACHE}
	make clean_container
	make create
	make start
	make show

stag:
	$(eval override BUILD_TYPE=staging)
	$(eval override DOCKER_TAG=${CONTAINER_NAME}:${BUILD_TYPE})
	# This will return only the IP address associated with the domain name ans assign it to ADDRESS
	$(eval override ADDRESS=$(shell dig +short <your staging URL e.g. mojo-react-app-staging.com>))

	make build -e PULL=true NO_CACHE=true DOCKER_TAG=${DOCKER_TAG}
	make clean_container -e BUILD_TYPE=${BUILD_TYPE}
	make create -e ADDRESS=${ADDRESS} BUILD_TYPE=${BUILD_TYPE} DOCKER_TAG=${DOCKER_TAG}
	make start -e BUILD_TYPE=${BUILD_TYPE}
	make show

prod:
	$(eval override BUILD_TYPE=production)
	$(eval override DOCKER_TAG = ${CONTAINER_NAME}:${BUILD_TYPE})
	# This will return only the IP address associated with the domain name ans assign it to ADDRESS
	$(eval override ADDRESS=$(shell dig +short <your production URL e.g. mojo-react-app.com))

	make build -e PULL=true NO_CACHE=true DOCKER_TAG=${DOCKER_TAG}
	make clean_container -e BUILD_TYPE=${BUILD_TYPE}
	make create -e ADDRESS=${ADDRESS} BUILD_TYPE=${BUILD_TYPE} DOCKER_TAG=${DOCKER_TAG}
	make start -e BUILD_TYPE=${BUILD_TYPE}
	make show

show:
	# show running containers
	docker ps | grep ${CONTAINER_NAME}

rebuild:
	# rebuilt the dockerfile
	make clean_container -e BUILD_TYPE=${BUILD_TYPE}
	make build -e PULL=true NO_CACHE=true DOCKER_TAG=${DOCKER_TAG}

up:
	# Run container on port
	make build -e PULL=true NO_CACHE=true DOCKER_TAG=${DOCKER_TAG}
	make run -e HTTPS_OUT_PORT=${HTTPS_OUT_PORT} HTTPS_IN_PORT=${HTTPS_IN_PORT} BUILD_TYPE=${BUILD_TYPE} DOCKER_TAG=${DOCKER_TAG}

login:
	# run as a service and attach to it
	docker exec -it ${CONTAINER_NAME}_${BUILD_TYPE} bash

release:
	make build -e PULL=true NO_CACHE=true DOCKER_TAG=${DOCKER_TAG}
	make push -e VERSION=${VERSION}

push:
	docker push $(DOCKER_REPO)/$(CONTAINER_NAME):$(VERSION)

pull:
	docker pull $(DOCKER_REPO)/$(CONTAINER_NAME):$(VERSION)

# Docker tagging
tag:
	## Generate container tags for the `{version}`
	@echo 'create tag $(VERSION)'
	docker tag $(DOCKER_TAG) $(DOCKER_REPO)/$(CONTAINER_NAME):$(VERSION)

help:
	@echo ''
	@echo 'Usage: make [TARGET] [EXTRA_ARGUMENTS]'
	@echo 'Targets:'
	@echo '  build    	build docker --image--'
	@echo '  rebuild  	rebuild docker --image--'
	@echo '  dev     	run docker --container-- in development mode => $(DOCKER_TAG)'
	@echo '  stag     	run docker --container-- in staging mode'
	@echo '  prod     	run docker --container-- in production mode'
	@echo '  login   	run as service and login --container--'
	@echo '  clean_image    	remove docker --image-- '
	@echo ''
	@echo 'Extra arguments:'
	@echo 'CONTAINER_NAME=: 	make clean_container -e CONTAINER_NAME=my_app (no need to provide this param, it will be set by default)'
	@echo 'BUILD_TYPE=:		make clean_container -e CONTAINER_NAME=my_app BUILD_TYPE=staging (whether the build type is 'development', 'staging' or 'production')'
	@echo 'DOCKER_TAG=:		make build -e DOCKER_TAG=my_app:staging'
	@echo 'HTTPS_OUT_PORT=:		make create -e HTTPS_IN_PORT=8080 (port from which the request will come- outside world)'
	@echo 'HTTPS_IN_PORT=:		make create -e HTTPS_IN_PORT=443 (port to which the request will be forwarded)'
```

I know its overwhelming. Lets go through each one of them.
* Initial 12 lines are the default parameters. In case you will not provide any parameter it will take those arguments. By default everything is in `development` mode. You will have to overwrite it for different modes.
```
DOCKER_TAG     =>  Name of your app with the env name (e.g. mojo_react_app:development)
NO_CACHE       =>  Whether to start the build from scratch (true|false). Default - false
PULL           =>  Whether to pull a newer version of the image (true|false). Default - false
BUILD_TYPE     =>  Build environment (development|staging|production). Default - development
CONTAINER_NAME =>  Name of the container (e.g. mojo_react_app)
HTTPS_OUT_PORT =>  HTTPS host port visible to outside world while publishing the container(443)
HTTPS_IN_PORT  =>  HTTPS container port on which traffic will be coming inside the container(443)
HTTP_OUT_PORT  =>  HTTP host port visible to outside world while publishing the container(80)
HTTP_IN_PORT   =>  HTTP container port on which traffic will be coming inside the container(80)
DOCKER_REPO    =>  URL of the docker repo where you want to push or pull image.
ADDRESS        =>  IP address of the container from where the outside word can access it. Default - 0.0.0.0
DOCKER_CFG     =?  Path to docker config files on host machine. This will contain the apache certificate and will be private to you.
```
* `build` will build your dockerfile
* `create` will create the container and copy the apache certificates
* `start` will start the container
* `run` will create and start the container. It will internally call the `create` and `start` command.
* `stop` will stop the running container.
* `clean_container` will remove/delete the remove previous container if exist.
* `clean_image` will remove/delete created image if exist.
* `show` will show the info about running container.
* `rebuild` will rebuilt the dockerfile. It will just call `clean_container` and `build` the image again.
* `up` will run the container on given port. It will always pull  the new image.
* `login` will login to the container. Using this you can see the  traffic logs and what is going on inside the container.
* `release` `push` `pull` `tag` - These commands you will be using when you want to create tag and push it to docker repo url(somewhere where you can access it and you don't have to create the image again). This is handful in Jenkins deployments, otherwise in normal dev work its not that important and will not be used much.
* `help` will give you the info about different param and what they do.
* Now 3 important commands which we will be using more often is -
```shell
    make dev
    make stag
    make prod
```

# make dev
`dev` is written in such a way that you don't have to worry about your container name and param. Just update the default value with your project specific value one time and your dev environment will be ready in no time.
It will build the container, create it and run it with single command.
```shell
make dev
```
Run this command.
Here are the screenshot at different point of time for this command. For the first time this command will take few minutes but subsequent run will be completed within few seconds.
![make_dev](https://raw.githubusercontent.com/rai-gaurav/mojo_react_app/main/with_jsx/server/output_screenshot/make_dev.png)
![mojo_installation](https://raw.githubusercontent.com/rai-gaurav/mojo_react_app/main/with_jsx/server/output_screenshot/Mojo_installation.png)
![container_up](https://raw.githubusercontent.com/rai-gaurav/mojo_react_app/main/with_jsx/server/output_screenshot/container_up.png)
I have highlighted different section for better understanding.
You can access your app on https://localhost/
![HomePage](https://raw.githubusercontent.com/rai-gaurav/mojo_react_app/main/with_jsx/server/output_screenshot/HomePage.png)

# make stag
This command you will be running when you want to deploy your application in staging environment.
If you look closely we are overriding the default config for staging in initial few lines.
```Makefile
	 $(eval override BUILD_TYPE=staging)
	$(eval override DOCKER_TAG=${CONTAINER_NAME}:${BUILD_TYPE})
	# This will return only the IP address associated with the domain name ans assign it to ADDRESS
	$(eval override ADDRESS=$(shell dig +short <your staging URL e.g. mojo-react-app-staging.com>))
```
The 4th lines is where you have to make your changes as per your requirement. We are overriding the default value of address(0.0.0.0) with your domain name of staging environment. We are using the `dig` utility to generate the IP address from that domain name and assigning it to ADDRESS.
Just run 
```shell
make stag
```
and you can access your staging environment.
E.g. 
      https://mojo-react-app-staging.com/ 
**or**
      https://staging-ip-add/

# make prod
This command you will be running when you want to deploy your application in production environment.
Again, you can see we are overriding the default config for production in initial few lines.
```Makefile
	 $(eval override BUILD_TYPE=production)
	$(eval override DOCKER_TAG = ${CONTAINER_NAME}:${BUILD_TYPE})
	# This will return only the IP address associated with the domain name ans assign it to ADDRESS
	$(eval override ADDRESS=$(shell dig +short <your production URL e.g. mojo-react-app.com))
```
Here again in 4th lines is where you have to make your changes as per your requirement. Update it to your prod environment hostname. We are using the `dig` utility to generate the IP address from that domain name and assigning it to ADDRESS.
Just run 
```shell
make prod
```
and you can access your staging environment.
E.g. 
      https://mojo-react-app.com/ 
**or**
      https://production-ip-add/

# Stop the container 
```shell
make stop
```
By default it will stop the dev container. Pass different param for stag or prod env.

# Login to container
```shell
make login
```
Again it will login to dev one by default. You can pass param to override that behavior.
![LoginContainer](https://raw.githubusercontent.com/rai-gaurav/mojo_react_app/main/with_jsx/server/output_screenshot/make_login.png)

# Conclusion
This Makefile is just a wrapper around different docker commands.
Especially during development you can enable live reload and your changes in host will be available in docker container also . More info [here](https://www.freecodecamp.org/news/how-to-enable-live-reload-on-docker-based-applications/).
You can use it for any Dockerfile. Here I am using a Mojolicious application. But the same Makefile can be used for a Django application or any other.
 
I hope this is useful for you and make your life more smoother.
The Makefile is also available on [github](https://github.com/rai-gaurav/mojo_react_app/tree/main/with_jsx/server/mojo_react_app)

Perl onion logo taken from [here](https://github.com/dnmfarrell/Perl-Icons/blob/master/Icons/Perl_Onion_Color.svg)
Mojolicious logo taken from [here](https://github.com/mojolicious/mojo/blob/master/lib/Mojolicious/resources/public/mojo/logo.png)
Docker logo taken from [here](https://www.docker.com/company/newsroom/media-resources)
GNU logo taken from [here](https://www.gnu.org/software/make/)
