# Continuous Intgration with Docker interactions

This article explains how to configure gitlabci runners to run your code and provision target to validate your code.

> Note: [Docker](https://docs.docker.com/install/) is a requirement on your host device to both run gitlab runner and your project.

## Configure a gitlab ci runner

First step is to configure a gitlab runner and connect it to your projet. Gitlab provides a docker image to deploy their runner. It is available on this [page](https://docs.gitlab.com/ee/ci/README.html).

A default command to start a runner is the following:

```shell
$ docker run -d 
   --name gitlab-runner \
   --restart always \
   -v /var/run/docker.sock:/var/run/docker.sock \
   gitlab/gitlab-runner:latest
```

This command starts a docker gitlabci runner and share following folder with host filesystem:

- `/var/run/docker.sock` is bind to the same path within container

In this scenario, configuration is not permanent and will be deleted after we stop the container. And especially configuration related to gitlabci service. To make it permanent, add following line to previous command:

```shell
$ mkdir -p /home/docker/gitlab-runner

$ docker run -d 
   --name gitlab-runner \
   --restart always \
   -v /var/run/docker.sock:/var/run/docker.sock \
   -v /home/docker/gitlab-runner:/etc/gitlab-runner \
   gitlab/gitlab-runner:latest
```

Then, we have to register runner to your gitlab project. Go to `https://gitlab.your.server.com/your-namespace/your-project/settings/ci_cd` and note both your __`token`__ and __`URL`__

Once you have these information, we can register the runner with following command:

```shell
docker exec -it gitlab-runner gitlab-runner register \
	  --non-interactive \
	  --url $gitlab.your.server.com$ \
	  --registration-token $TOKEN$ \
	  --executor "docker" \
	  --docker-image alpine:3 \
	  --description "docker-runner" \
	  --tag-list $YOUR_TAG \
	  --docker-volumes /var/run/docker.sock:/var/run/docker.sock \
	  --run-untagged \
	  --locked="false"
```

