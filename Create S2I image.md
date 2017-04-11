# Create your own S2I image

In this lab you will learn to created S2I (Source to Image) builder images using RHEL base image. This is a simple example that creates a Lighttpd S2I image to run on OpenShift. You can follow a similar approach for other technologies.


### Prerequisites
* This lab requires a RHEL box as we will be using RHEL base image in this use case.
* This rhel box should be subscribed to RedHat Network.
* Make sure that docker is installed and is running on this RHEL box. (yum install docker -y)

### Step 1 Download the S2I tools 

Download latest linux version from [https://github.com/openshift/source-to-image/releases/]()

```
wget https://github.com/openshift/source-to-image/releases/download/v1.1.2/source-to-image-v1.1.2-5732fdd-linux-386.tar.gz

tar -xvzf source-to-image-v1.1.2-5732fdd-linux-386.tar.gz
```

It will have two executables s2i and sti

```
mv s2i /usr/local/bin     
mv sti /usr/local/bin 
```
or add to PATH the folder where the execs are untarred

```
PATH=$PATH:/home/<myuser>/source-to-image-binaries 
```

Verify that the `s2i` tool works by running

```
$ s2i
Source-to-image (S2I) is a tool for building repeatable docker images.

A command line interface that injects and assembles source code into a docker image.
Complete documentation is available at http://github.com/openshift/source-to-image

Usage:
  s2i [flags]
  s2i [command]

Available Commands:
  version           Display version
  build             Build a new image
  rebuild           Rebuild an existing image
  usage             Print usage of the assemble script associated with the image
  create            Bootstrap a new S2I image repository
  genbashcompletion Generate Bash completion for the s2i command

Flags:
      --ca="": Set the path of the docker TLS ca file
      --cert="": Set the path of the docker TLS certificate file
  -h, --help[=false]: help for s2i
      --key="": Set the path of the docker TLS key file
      --loglevel=0: Set the level of log output (0-5)
  -U, --url="unix:///var/run/docker.sock": Set the url of the docker socket to use

Use "s2i [command] --help" for more information about a command.

```

### Step 2 Create the S2I Builder Working Structure

Run the following command to create a directory named `s2i-lighttpd` where the artifacts for the S2I builder will be added. The target S2I builder image name will be `lighttpd-rhel`

```
s2i create lighttpd-rhel s2i-lighttpd
```

Run `ls s2i-lighttpd` and observe the following folder structure

* `Dockerfile` – This is a standard Dockerfile where we’ll define the builder image
* `Makefile` – a helper script for building and testing the builder image
* `test/`
	* `run` – test script to test if the builder image works correctly
	* `test-app/` – directory for your test application
* `.s2i/bin/`
	* `assemble` – script responsible for building the application
	* `run` – script responsible for running the application
	* `save-artifacts` – script responsible for incremental builds, covered in a future article
	* `usage` – script responsible for printing the usage of the builder image


### Step 3 Update Assemble and Run Scripts

Lighttpd is a httpd server. The only thing we need to do during assembly is to copy the source files into a directory from which Lighttpd will serve. 

The S2I Builder will copy the source code by default into `/tmp/src`. This location can be changed by setting the `io.openshift.s2i.destination` label or passing `--destination` flag, in which case the sources will be placed in the `src` subdirectory of the directory you specified. In the `assemble` code below we will copy the source from this `/tmp/src` to the working directory.

Working directory is currently defined by the `s2i-base-rhel7` base image here [https://github.com/openshift/s2i-base/blob/master/Dockerfile.rhel7#L25)]() as `/opt/app-root/src`.

Edit `.s2i/bin/assemble` script to be as below:

```
#!/bin/bash -e
#
# S2I assemble script for the 'lighttpd-rhel' image.
# The 'assemble' script builds your application source so that it is ready to run.
#
# For more information refer to the documentation:
#   https://github.com/openshift/source-to-image/blob/master/docs/builder_image.md
#

if [[ "$1" == "-h" ]]; then
    # If the 'lighttpd-rhel' assemble script is executed with '-h' flag,
    # print the usage.
    exec /usr/libexec/s2i/usage
fi

# Restore artifacts from the previous build (if they exist).
#
if [ "$(ls /tmp/artifacts/ 2>/dev/null)" ]; then
  echo "---> Restoring build artifacts..."
  mv /tmp/artifacts/. ./
fi

echo "---> Installing application source..."
cp -Rf /tmp/src/. ./

echo "---> Building application from source..."
# TODO: Add build steps for your application, eg npm install, bundle install
```

Now, change the `.s2i/bin/run` script to start Lighttpd server as shown below:

```
#!/bin/bash -e
#
# S2I run script for the 'lighttpd-rhel' image.
# The run script executes the server that runs your application.
#
# For more information see the documentation:
#   https://github.com/openshift/source-to-image/blob/master/docs/builder_image.md
#

#exec <start your server here>
exec lighttpd -D -f /opt/app-root/etc/lighttpd.conf
```
**Note** The above `exec` command refers to a `lighttpd.conf` configuration file. We will create that in the next step. 


We won't be using `.s2i/bin/save-artifacts`. So let us delete the same

```
rm .s2i/bin/save-artifacts
```

Change permissions 

```
chmod 755 .s2i/bin/*
```

Now we have the S2I scripts ready.

### Step 4 Add Lighttpd Configuration File

Lighttpd requires a configuration file to run this server. Let us add this file `s2i-lighttpd/etc/lighttpd.conf` with minimal content for Lighttpd to run. 

**Note** While we are placing this configuration file in the `etc` folder, we will copy this file to appropriate location when we create the container image in the next step. 


```
# directory where the documents will be served from
server.document-root = "/opt/app-root/src"

# port the server listens on
server.port = 8080

# default file if none is provided in the URL
index-file.names = ( "index.html" )

# configure specific mimetypes, otherwise application/octet-stream will be used for every file
mimetype.assign = (
  ".html" => "text/html",
  ".txt" => "text/plain",
  ".jpg" => "image/jpeg",
  ".png" => "image/png"
)
```

**Note** In this config file, we have set the `server.document-root` to the place where we copied the sources in the assemble script :)



### Step 5 Edit the Container Image Specification

Modify the `Dockerfile` for the following changes: 
 
* To use RHEL based builder image from Redhat's registry i.e., `registry.access.redhat.com/rhscl/s2i-base-rhel7`
* Added an environment variable to include LIGHTTPD version
* Adding Docker Labels
* In order to install Lighttpd, you will need to install epel as well. Hence both are included in the `yum install`
* This label defines the location for S2I scripts
`io.openshift.s2i.scripts-url=image:///usr/libexec/s2i` and copy the s2i scripts to `/usr/libexec/sti`  
* Copy the Lighttpd configuration file from `/etc` into `/opt/app-root/etc` 
* Change the ownership of `/opt/app-root' to user 1001 and set the Docker USER to 1001
* This container would expose port 8080

The resultant code is also shown below. Edit slowly and **double verify** :)) 

```

# lighttpd-rhel
FROM registry.access.redhat.com/rhscl/s2i-base-rhel7

# TODO: Put the maintainer name in the image metadata
# MAINTAINER Veer Muchandi <veer@redhat.com>

# TODO: Rename the builder environment variable to inform users about application you provide them
ENV LIGHTTPD_VERSION=1.4.35

# TODO: Set labels used in OpenShift to describe the builder image
LABEL io.k8s.description="Platform for serving static html pages" \
      io.k8s.display-name="Lighttpd 1.4.35" \
      io.openshift.expose-services="8080:http" \
      io.openshift.tags="builder,html,lighttpd"

# TODO: Install required packages here:
# RUN yum install -y ... && yum clean all -y
RUN rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm && \
    yum install -y lighttpd && \
# clean yum cache files, as they are not needed and will only make the image bigger in the end
    yum clean all -y




# Defines the location of the S2I
LABEL io.openshift.s2i.scripts-url=image:///usr/libexec/s2i

# TODO: Copy the S2I scripts to /usr/libexec/s2i
COPY ./.s2i/bin/ /usr/libexec/s2i

# TODO (optional): Copy the builder files into /opt/app-root
# COPY ./<builder_folder>/ /opt/app-root/
# Copy the lighttpd configuration file
COPY ./etc/ /opt/app-root/etc

# TODO: Drop the root user and make the content of /opt/app-root owned by user 1001
RUN chown -R 1001:1001 /opt/app-root

# This default user is created in the openshift/base-centos7 image
USER 1001

# TODO: Set the default port for applications built using this image
EXPOSE 8080

# TODO: Set the default CMD for the image
CMD ["/usr/libexec/s2i/usage"]
```



### Step 6 Build the S2I Builder Image

Change to `s2i-lighttpd` directory and run

```
make build
```

This step will invoke "docker build" using the Dockerfile above to create a docker image with name `lighttpd-rhel`.

Upon successful execution of `make build` run 

```
docker images
```
to verify the existence of the `lighttpd-rhel` image on your rhel box.

### Step 7 Testing the S2I Image

Now it’s time to test our builder image. Let's create an index.html file in the s2i-lighttpd/test/test-app directory with the following contents:

```
<!doctype html>
<html>
  <head>
    <title>test-app</title>
  </head>
<body>
  <h1>Hello from lighttpd served index.html!</h1>
</body>
```

**Note** from the lighttpd.conf file above that index.html is the default

Now let us do the first build. Run the following command from the s2i-lighttpd directory:

```
s2i build test/test-app/ lighttpd-rhel sample-app
```

This invokes the S2I build process using the `assemble` script in the lighttpd-rhel image. The build creates an applicatiom image for the test application with name `sample-app`.

Once the image is created you can check the existence by running `docker images` again to find `sample-app`.

Now test the `sample-app` image by running

```
docker run -p 8080:8080 sample-app 
```

### Step 8 Login to OpenShift registry

Now we are ready to test this on OpenShift Cluster. 

---------
**Notes for the Cluster Administrator** others can read as well

Pushing to OpenShift Cluster requires the registry on the cluster to have been exposed (i.e registry service should have been exposed by a route) following the process listed here [https://docs.openshift.com/container-platform/3.3/install_config/registry/securing_and_exposing_registry.html]()

Also you need a user that is given `system:registry` role to be able to login to this registry as described here [https://docs.openshift.com/container-platform/3.3/install_config/registry/accessing_registry.html#access-user-prerequisites]()

As an example the cluster administrator should have run the following command to allow me to talk to registry.

```
oadm policy add-role-to-user system:registry veer
```

Also the user pushing the images should be given `image-builder` role by the administrator to the project into which the image is being pushed. This is not available by default even if it is user's own project. As an example, the cluster administrator should run the following command.

```
oadm policy add-role-to-user system:image-builder veer -n s2itest
```
------------


Login as the user that has access to push to the docker registry (`system:image-builder` role as explained in the above link) using the `oc login` command.

Create a new project with name `s2itest-UserName` where you want to push the S2I image to. Substitute with your user name. It makes your project name unique.

```
oc new-project s2itest-UserName
```

If the user is given the `image-builder` access on a per project basis, ask your cluster administrator to provide `image-builder` access to your user account for the `s2itest-UserName` project.

Find the oauth token assigned at login by running the following command. We will use this same token to log into docker registry.

```
oc whoami -t
```
**Note** the token output by the command above.

In order to connect to the trusted registry, you will need certificate that is trusted by the registry. Place the certificate in `/etc/docker/certs.d/<registry-url>` folder. For this lab, **ask the instructor** for the certificate to use and copy to the above location as shown below:

**Make sure the registry-url is as given by the instructor** when you create the directory below.

```
mkdir -p /etc/docker/certs.d/registry.apps.devday.ocpcloud.com
cp ca.crt /etc/docker/certs.d/registry.apps.devday.ocpcloud.com/
```


Login to the docker registry using oauth-token from the last step, your username, email, and the registry url for your exposed registry. **Substitute appropriate values below**

```
docker login -u <username> -e <email> -p <oauth-token> <registry url>
```
as an example 

```
docker login -u veer -e veer@example.com -p I6PMthJN-BjK7jsPNpeQ9vIZlXxOoQfjaMOYqFOaexU registry.apps.devday.ocpcloud.com
```

**As an alternative** you can also set the docker registry as `insecure`, in which case the `ca.crt` wouldn't be required. For this edit the file `/etc/sysconfig/docker` and add the line  `INSECURE_REGISTRY='--insecure-registry <registry name>'`. Then restart docker service by running `systemctl restart docker`.

Now tag your `lighttpd-rhel` docker image to be able to push into the registry. **Substitute appropriate values below**

```
docker tag lighttpd-rhel <registry url>/s2itest-<username>/lighttpd-rhel
```
Run `docker images` to verify the newly tagged image. Make a note of the complete image name (As an example this image name would be like `registry.apps.devday.ocpcloud.com/s2itest-veer/lighttpd-rhel:latest`)


### Step 9 Create ImageStream 

Create a json ImageStream file with the following content. Edit to make sure your image name matches with what you noted in the previous step. Let us name it as `lighttpd-rhel-is.json`. Also make sure that the `namespace` is pointing to your project name (`s2itest-UserName`)

```
{
    "kind": "ImageStream",
    "apiVersion": "v1",
    "metadata": {
        "name": "lighttpd-rhel",
        "namespace": "s2itest-veer"
    },
    "spec": {
        "tags": [
            {
                "name": "latest",
                "annotations": {
                    "description": "Run HTML",
                    "iconClass": "icon-tomcat",
                    "tags": "builder,lighttpd"
                },
            "from": {
              "kind": "DockerImage",
              "name": "registry.apps.devday.ocpcloud.com/s2itest-veer/lighttpd-rhel:latest"
            }
            }
        ]
    }
}
```
Note the `tags` section that uses the `builder` tag. This is **required** for OpenShift to recognize this imagestream as a builder image and **to display on the catalog** when you access from the web console.

Add this image stream to your project **substitute the UserName with yours**

```
oc create -f lighttpd-rhel-is.json -n s2itest-UserName
```

Verify the imagestreams in your project by running the following command. 

```
oc get is -n s2itest-UserName
```
and you should see an image with name `lighttpd-rhel`. However, the tags should be empty since we did not push the docker image into the registry yet.


### Step 10 Pushing image to Registry

ow push the docker image into the docker registry on your openshift cluster. **Substitute the username and registry-url with appropriate values**

```
docker push <registry-url>/s2itest-<username>/lighttpd-rhel
```

Check the image stream in your project and you should see the `lighthttpd-rhel` image with appropriate tags as shown below -

```
$ oc get is lighttpd-rhel -o yaml
apiVersion: v1
kind: ImageStream
metadata:
  annotations:
    openshift.io/image.dockerRepositoryCheck: 2016-08-10T13:40:48Z
  creationTimestamp: 2016-08-10T13:40:47Z
  generation: 3
  name: lighttpd-rhel
  namespace: s2itest-veer
  resourceVersion: "17110911"
  selfLink: /oapi/v1/namespaces/s2itest-veer/imagestreams/lighttpd-rhel
  uid: 0b43be6a-5f00-11e6-9126-fa163e38132c
spec:
  tags:
  - annotations:
      description: Run HTML
      iconClass: icon-tomcat
      tags: builder,lighttpd
    from:
      kind: DockerImage
      name: registry.apps.example.com/s2itest-veer/lighttpd-rhel:latest
    generation: 2
    importPolicy: {}
    name: latest
status:
  dockerImageRepository: 172.30.12.99:5000/s2itest-veer/lighttpd-rhel
  tags:
  - items:
    - created: 2016-08-10T13:41:19Z
      dockerImageReference: 172.30.12.99:5000/s2itest-veer/lighttpd-rhel@sha256:ea56d389358ead149ad8e51c0760900d72b763899ec03ad81730621c9759fcff
      generation: 2
      image: sha256:ea56d389358ead149ad8e51c0760900d72b763899ec03ad81730621c9759fcff
    tag: latest
```

Note that tags at the end that refer to the docker image id. If you don't see that the image stream won't work.

### Step 11 Test deploying an application

Now it is time to test your `lighttpd-rhel` S2I image from the Web Console.

* Login to the Web Console and select the `s2itest-UserName` project
* Select `Add to Project` Button
* You should see `lighttpd-rhel:latest` image in the catalog. If you don't see it readily, you can always filter the list by typing `lighttpd`
* Select this image, give a name and the following git-url to try out a simple webpage to deploy [https://github.com/VeerMuchandi/simple]()

This should build and deploy the application.



Once tested, if you want to make this image available across the cluster, handover your imagestream file and the image to the cluster administrator. The administrator would have to repeat steps 7 thru 9 for the `openshift` namespace.

### Summary

In this lab you have learnt how to create, test and use an S2I Builder image of your own.




















 














