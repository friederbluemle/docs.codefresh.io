---
title: "Working with Docker Registries"
description: "How to push, pull and tag Docker images in Codefresh pipelines"
group: docker-registries
redirect_from:
  - /docs/build-specific-revision-image/
  - /docs/image-management/build-specific-revision-image/
toc: true
---

Codefresh contains first class Docker registry support. This means that you don't need to manually write `docker login` and `docker pull/push` commands inside pipelines. You use instead declarative YAML and all credential configuration is configured centrally once.

## Viewing Docker images

To see all images currently from [all your connected Registries]({{site.baseurl}}/docs/docker-registries/external-docker-registries/), select *Artifacts -> Images* from the left sidebar and you will see a sorted list of all images.

{% 
  include image.html 
  lightbox="true" 
  file="/images/artifacts/cfcr/codefresh-registry-list.png" 
  url="/images/artifacts/cfcr/codefresh-registry-list.png" 
  alt="Codefresh Registry Image List" 
  caption="Codefresh Registry Image List" 
  max-width="70%" 
%}

For each image you get some basic details such as the git branch, commit message and hash that created it, date of creation as well as all tags. You can click on any image and look at [its individual metadata]({{site.baseurl}}/docs/docker-registries/metadata-annotations/).


On the top left of the screen you can find several filters that allow you to search for a specific subset of Docker images:

{% 
  include image.html 
  lightbox="true" 
  file="/images/artifacts/cfcr/codefresh-registry-filters.png" 
  url="/images/artifacts/cfcr/codefresh-registry-filters.png" 
  alt="Codefresh Registry Image filters" 
  caption="Codefresh Registry Image filters" 
  max-width="40%" 
%}

Filters include:

* Tagged/untagged images
* Base image name
* Git branch
* Tag
* Pipeline volumes

You can add multiple filters and they will work in an `AND` manner.

On the right side of the screen you also have a list of buttons for actions on each Docker image.
These are:

* Launching a Docker image as a [test environment]({{site.baseurl}}/docs/getting-started/on-demand-environments/)
* Promoting a Docker image (explained in the following sections)
* Looking at the docker commands that allow you to pull the image locally on your workstation
* Re-running the pipeline that created this image


## Pulling Docker images

Pulling Docker images in Codefresh is completely automatic. You only need to mention a Docker image by name and Codefresh will automatically pull it for you and use it in a pipeline. 

### Pulling public images

To pull a public image from Dockerhub or other public registry you simply mention the name of the image and tag that you want to use. For example:

```yaml
CollectAllMyDeps:
  title: Install dependencies
  image: python:3.6.4-alpine3.6
  commands:
    - pip install .
```

This [freestyle step]({{site.baseurl}}/docs/codefresh-yaml/steps/freestyle/) will pull from Dockerhub the image `python:3.6.4-alpine3.6` and then run the command `pip install .` inside it. You can see the image get pulled in the [Codefresh pipeline log]({{site.baseurl}}/docs/configure-ci-cd-pipeline/monitoring-pipelines/).

{% 
	include image.html 
	lightbox="true" 
	file="/images/artifacts/working-with-images/pull-public-image.png" 
	url="/images/artifacts/working-with-images/pull-public-image.png" 
	alt="Pulling a public image" 
	caption="Pulling a public image" 
	max-width="70%" 
%}

The image will also be cached in the [image cache]({{site.baseurl}}/docs/configure-ci-cd-pipeline/pipeline-caching/#distributed-docker-image-caching) without any other configuration.

Codefresh will also automatically pull for you any images mentioned in Dockerfiles (i.e. the `FROM` directive) as well as [service containers]({{site.baseurl}}/docs/codefresh-yaml/service-containers/).


### Pulling private images

To pull a private image from one of your connected registries, again you mention the image by name and tag. In order for Codefresh to understand that you are talking about a private image you need to prepend the appropriate prefix of the registry domain.

For example in the case of [ACR]({{site.baseurl}}/docs/docker-registries/external-docker-registries/azure-docker-registry/):

```
registry-name.azurecr.io/my-docker-repo/my-image-name:tag
```

You can find the full name of any docker image by visiting the image dashboard and looking at the URL field of any tag:

{% 
	include image.html 
	lightbox="true" 
	file="/images/artifacts/working-with-images/image-dashboard-tag.png" 
	url="/images/artifacts/working-with-images/image-dashboard-tag.png" 
	alt="Looking at tag of a private image" 
	caption="Looking at tag of a private image"
	max-width="65%" 
%}

The exact format of the full image name will depend on the type of registry you use. Codefresh will use the domain prefix of each image to understand which integration it will use. It will then take care of all `docker login` and `docker pull` commands on its own behind the scenes.

For example if you have connected [Azure]({{site.baseurl}}/docs/docker-registries/external-docker-registries/azure-docker-registry/), [AWS]({{site.baseurl}}/docs/docker-registries/external-docker-registries/amazon-ec2-container-registry/) and [Google]({{site.baseurl}}/docs/docker-registries/external-docker-registries/google-container-registry/) registries, you can pull 3 images for each in a pipeline like this:


 `codefresh.yml`
{% highlight yaml %}
{% raw %}
version: '1.0'
steps:
  my_go_unit_tests:
    title: Running Go Unit tests
    image: 'us.gcr.io/project-k8s-sample-123454/my-golang-app:prod'
    commands:
      - go test -v
  my_mvn_unit_tests:
    title: Running Maven Unit tests
    image: '123456789012.dkr.ecr.us-west-2.amazonaws.com/my-java-app:latest'
    commands:
      - mvn test
  my_python_unit_tests:
    title: Running Python Unit tests
    image: 'my-azure-registry.azurecr.io/kostis-codefresh/my-python-app:master'
    commands:
      - python setup.py test        
{% endraw %}
{% endhighlight %}

Codefresh will automatically login to each registry using the credentials you have defined centrally and pull all the images. The same thing will happen with Dockerfiles that mention any valid docker image in their `FROM` directive.


### Pulling images that were just built in the same pipeline

Codefresh allows you to create a docker image on demand and use it in the same pipeline that created it. In several scenarios (such as [unit tests]({{site.baseurl}}/docs/testing/unit-tests/)) it is very common to use a Docker image right after it was built.

In that case, as a shortcut Codefresh allows you to simply [mention the name]({{site.baseurl}}/docs/codefresh-yaml/variables/#context-related-variables) of the respective [build step]({{site.baseurl}}/docs/codefresh-yaml/steps/build/).

 `codefresh.yml`
{% highlight yaml %}
{% raw %}
version: '1.0'
steps:
  main_clone:
    title: Cloning main repository...
    type: git-clone
    repo: 'codefresh-contrib/python-flask-sample-app'
    revision: 'master'
    git: github
  MyAppDockerImage:
    title: Building Docker Image
    type: build
    image_name: my-app-image
    working_directory: ./
    tag: 'master'
    dockerfile: Dockerfile
  MyUnitTests:
    title: Running Unit tests
    image: '${{MyAppDockerImage}}'
    commands:
      - python setup.py test
  
{% endraw %}
{% endhighlight %}

In this pipeline Codefresh:

1. Checks out source code with the [git-clone step]({{site.baseurl}}/docs/codefresh-yaml/steps/git-clone/)
1. Builds a docker image that gets named `my-app-image:master`. Notice the lack of `docker push` commands
1. In the next step automatically uses that image and runs `python setup.py test` inside it. Again notice the lack of `docker pull` commands.

The important line here is the following:

{% highlight yaml %}
{% raw %}
    image: ${{MyAppDockerImage}}
{% endraw %}    
{% endhighlight %}

This says to Codefresh "in this step please use the Docker image that was built in the step named `MyAppDockerImage`".

You can see the automatic pull inside the Codefresh logs.

{% 
	include image.html 
	lightbox="true" 
	file="/images/artifacts/working-with-images/pull-private-image.png" 
	url="/images/artifacts/working-with-images/pull-private-image.png" 
	alt="Auto-Pulling a private image" 
	caption="Auto-Pulling a private image" 
	max-width="70%" 
%}

The image will still be pushed in your default Docker registry. If you don't want this behavior you can simply add the `disable_push` property in the build step.


## Pushing Docker images

Pushing to your default Docker registry is completely automatic. All successful [build steps]({{site.baseurl}}/docs/codefresh-yaml/steps/build/) automatically push to the default Docker registry of your Codefresh account without any extra configuration.

To push to another registry you only need to know how this registry is [linked into Codefresh]({{site.baseurl}}/docs/docker-registries/external-docker-registries/) and more specifically what is unique name of the integration. You can see that name by visiting your [integrations screen](https://g.codefresh.io/account-admin/account-conf/integration/registry) or asking your Codefresh administrator.


{% 
	include image.html 
	lightbox="true" 
	file="/images/artifacts/working-with-images/linked-docker-registries.png" 
	url="/images/artifacts/working-with-images/linked-docker-registries.png" 
	alt="Name of linked Docker Registries" 
	caption="Name of linked Docker Registries" 
	max-width="50%" 
%}

Once you know the registry identifier you can use it an [push step]({{site.baseurl}}/docs/codefresh-yaml/steps/push/) or [build step]({{site.baseurl}}/docs/codefresh-yaml/steps/build/) by mentioning the registry with that name:

 `codefresh.yml`
{% highlight yaml %}
{% raw %}
  build_image:
    title: Building my app image
    type: build
    image_name: my-app-image
    dockerfile: Dockerfile
    tag: 'master'
  push_to_registry:
    title: Pushing to Docker Registry 
    type: push
    #Name of the build step that is building the image
    candidate: '${{build_image}}'
    tag: '1.2.3'
    # Unique registry name
    registry: azure-demo       
{% endraw %}
{% endhighlight %}

Notice that
 * the `candidate` field of the push step mentions the name of the build step (`build_image`) that will be used for the image to be pushed
 * The registry is only identified by name (i.e. `azure-demo`). The domain and credentials are not part of the pipeline (they are already known to Codefresh by the Docker registry integration)

 You can also override the name of the image with any custom name. This way the push step can choose any image name regardless of what was used in the build step.

  `codefresh.yml`
{% highlight yaml %}
{% raw %}
  build_image:
    title: Building my app image
    type: build
    image_name: my-app-image
    dockerfile: Dockerfile
    tag: 'master'
  push_to_registry:
    title: Pushing to Docker Registry 
    type: push
    #Name of the build step that is building the image
    candidate: '${{build_image}}'
    tag: '1.2.3'
    # Unique registry name
    registry: azure-demo
    image_name: my-company/web-app       
{% endraw %}
{% endhighlight %}

Here the build step is creating an image named `my-app-image:master` but the push step will actually push it as `my-company/web-app:1.2.3`

For more examples such as using multiple tags, or pushing in parallel see the [push examples]({{site.baseurl}}/docs/codefresh-yaml/steps/push/#examples)

## Promoting Docker images

Apart from building and pushing a brand new docker image, you can also "promote" a docker image by copying it from one registry to another. 
You can perform this action either from the Codefresh UI or automatically from pipelines.


### Promoting images via the Codefresh UI

You have the capability to "promote" any image of your choosing and push it to an external Registry that you have integrated into Codefresh (such as Azure, Google, Bintray etc)


Visit the Docker image dashboard and click the *Promote* button for the image you wish to promote:

{% 
  include image.html 
  lightbox="true" 
  file="/images/artifacts/cfcr/docker-image-promotion.png" 
  url="/images/artifacts/cfcr/docker-image-promotion.png" 
  alt="Promoting a Docker image" 
  caption="Promoting a Docker image" 
  max-width="50%" 
%}

You will get a list of your connected registries. Choose the target Registry and define the tag that you want to push. Then click the *Promote* button to "copy" this image from its existing registry to the target Registry.

### Promoting images in pipelines

You can also copy images from one registry to the other inside a pipeline.
This is accomplished by specifying an existing image in the `candidate` field of the push step.

For example:

  `codefresh.yml`
{% highlight yaml %}
{% raw %}
  promote_to_production_registry:
    title: Promoting to Azure registry 
    type: push
    candidate: us.gcr.io/project-k8s-sample-123454/my-golang-app
    tag: '1.2.3'
    # Unique registry name
    registry: azure-demo      
{% endraw %}
{% endhighlight %}

In the example above we promote an image from [GCR]({{site.baseurl}}/docs/docker-registries/external-docker-registries/google-container-registry/) to [ACR]({{site.baseurl}}/docs/docker-registries/external-docker-registries/azure-docker-registry/) (which is already setup as `azure-demo`).

You can even "promote" docker images within the same registry by simply creating new tags. 
For example:

  `codefresh.yml`
{% highlight yaml %}
{% raw %}
  promote_to_production:
    title: Marking image with prod tag
    type: push
    candidate: my-azure-registry.azurecr.io/kostis-codefresh/my-python-app:1.2.3
    tag: 'production'
    # Unique registry name
    registry: azure-demo      
{% endraw %}
{% endhighlight %}

In the example above the image `my-azure-registry.azurecr.io/kostis-codefresh/my-python-app:1.2.3` is re-tagged as `my-azure-registry.azurecr.io/kostis-codefresh/my-python-app:prod`. A very common pattern is to mark images with a special tag such as `prod` **after** the image has been deployed successfully.


## What to read next

* [Push pipeline step]({{site.baseurl}}/docs/codefresh-yaml/steps/push/)
* [External Docker Registries]({{site.baseurl}}/docs/docker-registries/external-docker-registries/)
* [Accessing a Docker registry from your Kubernetes cluster]({{site.baseurl}}/docs/deploy-to-kubernetes/access-docker-registry-from-kubernetes/)
* [Build and Push an image example]({{site.baseurl}}/docs/yaml-examples/examples/build-and-push-an-image/)

