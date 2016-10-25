# Creating and Using Templates

In this lab we will learn to create a template and use the same. We will use a two tiered application example to create a template out of it.

####Step 1: Add a new project

Let us add a new project with name `templatize-UserName`. **Remember** to substitute the username.

```
oc new-project templatize-UserName
```

####Step 2: Create a two tiered application

We will use the same example as in the [Using Templates](https://github.com/RedHatWorkshops/openshiftv3-workshop/blob/master/5.%20Using%20templates.md) lab before. However here we will do it quickly from the commmand line.

* Add the MySQL database

```
$ oc new-app -e MYSQL_USER='user',MYSQL_PASSWORD='password',MYSQL_DATABASE='sampledb' mysql --name='mysql' -l tier=database
--> Found image 1a7506a (5 weeks old) in image stream "mysql" in project "openshift" under tag "5.6" for "mysql"

    MySQL 5.6 
    --------- 
    MySQL is a multi-user, multi-threaded SQL database server

    Tags: database, mysql, mysql56, rh-mysql56

    * This image will be deployed in deployment config "mysql"
    * Port 3306/tcp will be load balanced by service "mysql"
      * Other containers can access this service through the hostname "mysql"
    * This image declares volumes and will default to use non-persistent, host-local storage.
      You can add persistent volumes later by running 'volume dc/mysql --add ...'

--> Creating resources with label tier=database ...
    deploymentconfig "mysql" created
    service "mysql" created
--> Success
    Run 'oc status' to view your app.
```

Note that the above command created two components, a deployment configuration with the name "mysql" and a service with the same name.


* Add the front-end PHP application

```
$ oc new-app -e MYSQL_USER='user',MYSQL_PASSWORD='password',MYSQL_DATABASE='sampledb' https://github.com/VeerMuchandi/dbtest -l tier=frontend
--> Found image 2240241 (5 weeks old) in image stream "php" in project "openshift" under tag "5.6" for "php"

    Apache 2.4 with PHP 5.6 
    ----------------------- 
    Platform for building and running PHP 5.6 applications

    Tags: builder, php, php56, rh-php56

    * The source repository appears to match: php
    * A source build using source code from https://github.com/VeerMuchandi/dbtest will be created
      * The resulting image will be pushed to image stream "dbtest:latest"
      * Use 'start-build' to trigger a new build
    * This image will be deployed in deployment config "dbtest"
    * Port 8080/tcp will be load balanced by service "dbtest"
      * Other containers can access this service through the hostname "dbtest"

--> Creating resources with label tier=frontend ...
    imagestream "dbtest" created
    buildconfig "dbtest" created
    deploymentconfig "dbtest" created
    service "dbtest" created
--> Success
    Build scheduled, use 'oc logs -f bc/dbtest' to track its progress.
    Run 'oc status' to view your app.
```

We added PHP application and since it is based on source code, it created a build configuration with the name "dbtest, an image stream, a deployment configuration and a service with the same name.

Now let us expose the service to create the route object.

```
$ oc expose service dbtest
route "dbtest" exposed
```

**Bonus Points:** Based on what you have learnt already, run the commands to verify the existence of these objects and view at these objects as json files. 


#### Create a template 

We will now create a template by combining all the above objects. 

Run the following command to export all the objects above as a template and redirect the result into a json file.

```
$ oc export is,bc,dc,svc,route --as-template=php-mysql-template -o json > php-mysql-template.json
```

And yes, it is as simple as that to create a template!!

The resultant json file will be very similar to the snippet shown below. You will notice that this is nothing but all the objects we created earlier put together.

```
$ cat php-mysql-template.json 
{
    "kind": "Template",
    "apiVersion": "v1",
    "metadata": {
        "name": "php-mysql-template",
        "creationTimestamp": null
    },
    "objects": [
        {
            "kind": "ImageStream",
            "apiVersion": "v1",
            "metadata": {
                "name": "dbtest",
                "generation": 1,
                "creationTimestamp": null,
                "labels": {
                    "tier": "frontend"
                },
                "annotations": {
                    "openshift.io/generated-by": "OpenShiftNewApp"
                }
            },
            "spec": {},
            "status": {
                "dockerImageRepository": ""
            }
        },
        {
            "kind": "BuildConfig",
            "apiVersion": "v1",
            "metadata": {
                "name": "dbtest",
                "creationTimestamp": null,
                "labels": {
                    "tier": "frontend"
                },
                "annotations": {
                    "openshift.io/generated-by": "OpenShiftNewApp"
                }
            },
            "spec": {
                "triggers": [
                    {
                        "type": "GitHub",
                        "github": {
                            "secret": "c8-QkV1x1LtenM70kfyr"
                        }
                    },
                    {
                        "type": "Generic",
                        "generic": {
                            "secret": "WeWKHN9NftTwKYt7GQ7_"
                        }
                    },
                    {
                        "type": "ConfigChange"
                    },
                    {
                        "type": "ImageChange",
                        "imageChange": {}
                    }
                ],
                "runPolicy": "Serial",
                "source": {
                    "type": "Git",
                    "git": {
                        "uri": "https://github.com/VeerMuchandi/dbtest"
                    }
                },
                "strategy": {
                    "type": "Source",
                    "sourceStrategy": {
                        "from": {
                            "kind": "ImageStreamTag",
                            "namespace": "openshift",
                            "name": "php:5.6"
                        }
                    }
                },
                "output": {
                    "to": {
                        "kind": "ImageStreamTag",
                        "name": "dbtest:latest"
                    }
                },
                "resources": {},
                "postCommit": {}
            },
            "status": {
                "lastVersion": 0
            }
        },
        {
            "kind": "DeploymentConfig",
            "apiVersion": "v1",
            "metadata": {
                "name": "dbtest",
                "generation": 1,
                "creationTimestamp": null,
                "labels": {
                    "tier": "frontend"
                },
                "annotations": {
                    "openshift.io/generated-by": "OpenShiftNewApp"
                }
            },
            "spec": {
                "strategy": {
                    "type": "Rolling",
                    "rollingParams": {
                        "updatePeriodSeconds": 1,
                        "intervalSeconds": 1,
                        "timeoutSeconds": 600,
                        "maxUnavailable": "25%",
                        "maxSurge": "25%"
                    },
                    "resources": {}
                },
                "triggers": [
                    {
                        "type": "ConfigChange"
                    },
                    {
                        "type": "ImageChange",
                        "imageChangeParams": {
                            "automatic": true,
                            "containerNames": [
                                "dbtest"
                            ],
                            "from": {
                                "kind": "ImageStreamTag",
                                "namespace": "templatize",
                                "name": "dbtest:latest"
                            }
                        }
                    }
                ],
                "replicas": 1,
                "test": false,
                "selector": {
                    "deploymentconfig": "dbtest",
                    "tier": "frontend"
                },
                "template": {
                    "metadata": {
                        "creationTimestamp": null,
                        "labels": {
                            "deploymentconfig": "dbtest",
                            "tier": "frontend"
                        },
                        "annotations": {
                            "openshift.io/container.dbtest.image.entrypoint": "[\"container-entrypoint\",\"/bin/sh\",\"-c\",\"$STI_SCRIPTS_PATH/usage\"]",
                            "openshift.io/generated-by": "OpenShiftNewApp"
                        }
                    },
                    "spec": {
                        "containers": [
                            {
                                "name": "dbtest",
                                "image": "dbtest:latest",
                                "ports": [
                                    {
                                        "containerPort": 8080,
                                        "protocol": "TCP"
                                    }
                                ],
                                "env": [
                                    {
                                        "name": "MYSQL_DATABASE",
                                        "value": "sampledb"
                                    },
                                    {
                                        "name": "MYSQL_PASSWORD",
                                        "value": "password"
                                    },
                                    {
                                        "name": "MYSQL_USER",
                                        "value": "user"
                                    }
                                ],
                                "resources": {},
                                "terminationMessagePath": "/dev/termination-log",
                                "imagePullPolicy": "Always"
                            }
                        ],
                        "restartPolicy": "Always",
                        "terminationGracePeriodSeconds": 30,
                        "dnsPolicy": "ClusterFirst",
                        "securityContext": {}
                    }
                }
            },
            "status": {
                "observedGeneration": 1
            }
        },
        {
            "kind": "DeploymentConfig",
            "apiVersion": "v1",
            "metadata": {
                "name": "mysql",
                "generation": 2,
                "creationTimestamp": null,
                "labels": {
                    "tier": "database"
                },
                "annotations": {
                    "openshift.io/generated-by": "OpenShiftNewApp"
                }
            },
            "spec": {
                "strategy": {
                    "type": "Rolling",
                    "rollingParams": {
                        "updatePeriodSeconds": 1,
                        "intervalSeconds": 1,
                        "timeoutSeconds": 600,
                        "maxUnavailable": "25%",
                        "maxSurge": "25%"
                    },
                    "resources": {}
                },
                "triggers": [
                    {
                        "type": "ConfigChange"
                    },
                    {
                        "type": "ImageChange",
                        "imageChangeParams": {
                            "automatic": true,
                            "containerNames": [
                                "mysql"
                            ],
                            "from": {
                                "kind": "ImageStreamTag",
                                "namespace": "openshift",
                                "name": "mysql:5.6"
                            }
                        }
                    }
                ],
                "replicas": 1,
                "test": false,
                "selector": {
                    "deploymentconfig": "mysql",
                    "tier": "database"
                },
                "template": {
                    "metadata": {
                        "creationTimestamp": null,
                        "labels": {
                            "deploymentconfig": "mysql",
                            "tier": "database"
                        },
                        "annotations": {
                            "openshift.io/container.mysql.image.entrypoint": "[\"container-entrypoint\",\"run-mysqld\"]",
                            "openshift.io/generated-by": "OpenShiftNewApp"
                        }
                    },
                    "spec": {
                        "volumes": [
                            {
                                "name": "mysql-volume-1",
                                "emptyDir": {}
                            }
                        ],
                        "containers": [
                            {
                                "name": "mysql",
                                "image": "registry.access.redhat.com/rhscl/mysql-56-rhel7@sha256:0a710d858de8752b70c4fbebb7c157050549a0cfc1b2e52301a20990648d02a7",
                                "ports": [
                                    {
                                        "containerPort": 3306,
                                        "protocol": "TCP"
                                    }
                                ],
                                "env": [
                                    {
                                        "name": "MYSQL_DATABASE",
                                        "value": "sampledb"
                                    },
                                    {
                                        "name": "MYSQL_PASSWORD",
                                        "value": "password"
                                    },
                                    {
                                        "name": "MYSQL_USER",
                                        "value": "user"
                                    }
                                ],
                                "resources": {},
                                "volumeMounts": [
                                    {
                                        "name": "mysql-volume-1",
                                        "mountPath": "/var/lib/mysql/data"
                                    }
                                ],
                                "terminationMessagePath": "/dev/termination-log",
                                "imagePullPolicy": "IfNotPresent"
                            }
                        ],
                        "restartPolicy": "Always",
                        "terminationGracePeriodSeconds": 30,
                        "dnsPolicy": "ClusterFirst",
                        "securityContext": {}
                    }
                }
            },
            "status": {
                "observedGeneration": 2,
                "replicas": 1,
                "updatedReplicas": 1,
                "availableReplicas": 1
            }
        },
        {
            "kind": "Service",
            "apiVersion": "v1",
            "metadata": {
                "name": "dbtest",
                "creationTimestamp": null,
                "labels": {
                    "tier": "frontend"
                },
                "annotations": {
                    "openshift.io/generated-by": "OpenShiftNewApp"
                }
            },
            "spec": {
                "ports": [
                    {
                        "name": "8080-tcp",
                        "protocol": "TCP",
                        "port": 8080,
                        "targetPort": 8080
                    }
                ],
                "selector": {
                    "deploymentconfig": "dbtest",
                    "tier": "frontend"
                },
                "type": "ClusterIP",
                "sessionAffinity": "None"
            },
            "status": {
                "loadBalancer": {}
            }
        },
        {
            "kind": "Service",
            "apiVersion": "v1",
            "metadata": {
                "name": "mysql",
                "creationTimestamp": null,
                "labels": {
                    "tier": "database"
                },
                "annotations": {
                    "openshift.io/generated-by": "OpenShiftNewApp"
                }
            },
            "spec": {
                "ports": [
                    {
                        "name": "3306-tcp",
                        "protocol": "TCP",
                        "port": 3306,
                        "targetPort": 3306
                    }
                ],
                "selector": {
                    "deploymentconfig": "mysql",
                    "tier": "database"
                },
                "type": "ClusterIP",
                "sessionAffinity": "None"
            },
            "status": {
                "loadBalancer": {}
            }
        },
        {
            "kind": "Route",
            "apiVersion": "v1",
            "metadata": {
                "name": "dbtest",
                "creationTimestamp": null,
                "labels": {
                    "tier": "frontend"
                },
                "annotations": {
                    "openshift.io/host.generated": "true"
                }
            },
            "spec": {
                "host": "dbtest-templatize.apps.testv3.osecloud.com",
                "to": {
                    "kind": "Service",
                    "name": "dbtest",
                    "weight": 100
                },
                "port": {
                    "targetPort": "8080-tcp"
                }
            },
            "status": {
                "ingress": [
                    {
                        "host": "dbtest-templatize.apps.testv3.osecloud.com",
                        "routerName": "router",
                        "conditions": [
                            {
                                "type": "Admitted",
                                "status": "True",
                                "lastTransitionTime": "2016-10-17T15:32:48Z"
                            }
                        ]
                    }
                ]
            }
        }
    ]
}

```

#### Parameterize the template for re-use

While we created the template for a two tier application that involves a mysql database and a php application, the contents of this template are very specific to the application we created before. Since template is supposed to be generic, let us make it generic by editing it.

**Note** Make a copy of the template file as a backup, just in case.

We will add the following parameters to the template:

* `APPLICATION_NAME`	
The name assigned to all of the frontend objects defined in this template

* `GIT_URI`	
The URL of the repository with your application source code

* `GIT_REF`	
Branch name, tag or other ref of your repository

* `APPLICATION_DOMAIN`	
The exposed hostname/URL that will be given to the route

* `GITHUB_WEBHOOK_SECRET`		
A secret string used to configure the GitHub webhook

* `GENERIC_WEBHOOK_SECRET`	
A secret string used to configure the GitHub webhook

* `DATABASE_SERVICE_NAME`		
Database Service Name given to the MySQL service

* `MYSQL_DATABASE`	
Name of the database that will be created

* `MYSQL_USER`	
Database Username

* `MYSQL_PASSWORD`	
Database Password

The template will use these parameters by referring them using a the `$` symbol just like the environment variables. 
For example, `APPLICATION_NAME` will be referred as `${APPLICATION_NAME}` in the template and while processing OpenShift will substitute the value assigned to this variable.

To declare these parameters in a template, we will add the parameters section as below

```
    "parameters": [
        {
            "name": "APPLICATION_NAME",
            "displayName": "Application Name",
            "description": "The name assigned to all of the frontend objects defined in this template.",
            "value": "php-mysql-example",
            "required": true
        },
        {
            "name": "GIT_URI",
            "displayName": "Git Repository URL",
            "description": "The URL of the repository with your application source code.",
            "value": "https://github.com/VeerMuchandi/dbtest.git",
            "required": true
        },
        {
            "name": "GIT_REF",
            "displayName": "Git Reference",
            "description": "Set this to a branch name, tag or other ref of your repository if you are not using the default branch."
        },
        {
            "name": "APPLICATION_DOMAIN",
            "displayName": "Application HostName",
            "description": "The exposed hostname that will route to the application service, if left blank a value will be defaulted."
        },
        {
            "name": "GITHUB_WEBHOOK_SECRET",
            "displayName": "GitHub Webhook Secret",
            "description": "A secret string used to configure the GitHub webhook.",
            "generate": "expression",
            "from": "[a-zA-Z0-9]{40}"
        },
        {
            "name": "GENERIC_WEBHOOK_SECRET",
            "displayName": "GitHub Webhook Secret",
            "description": "A secret string used to configure the GitHub webhook.",
            "generate": "expression",
            "from": "[a-zA-Z0-9]{40}"
        },
        {
            "name": "DATABASE_SERVICE_NAME",
            "displayName": "Database Service Name",
            "value": "mysql"
        },
        {
            "name": "MYSQL_DATABASE",
            "displayName": "Database Name",
            "value": "sampledb"
        },
        {
            "name": "MYSQL_USER",
            "displayName": "Database User",
            "generate": "expression",
            "from": "[a-zA-Z0-9]{16}"
        },
        {
            "name": "MYSQL_PASSWORD",
            "displayName": "Database Password",
            "generate": "expression",
            "from": "[a-zA-Z0-9]{40}"
        }
    ]

```

`name` represents the name of the parameter
`displayName` is how it is displayed when the template is seen on Webconsole
`value` is the default value assigned if no value is keyed in by the user
`generate` allows you to auto-generate the values if no value is keyed in by the user
`from` defines the range of values allowed

Remove "namespace" parameter from the DeploymentConfig here as you want your template to be usable in any project

```
"triggers": [
 ..
                        "type": "ImageChange",
                        "imageChangeParams": {
..
                            "from": {
                                "kind": "ImageStreamTag",
                                "namespace": "templatize",
                                "name": "${APPLICATION_NAME}:latest"
```

Also remove the `status` sections as they represent the result of processing each of the objects. We don't need them in a template. As an example remove the sections that start with `status`; usually this is the last section in each object. So when you remove make sure that the **comma before this object is also removed**. 

```
            ,
            "status": {
                "observedGeneration": 2,
                "replicas": 1,
                "updatedReplicas": 1,
                "availableReplicas": 1
            }
```



The resultant template after the edits would be as under. 

**Take your time** and do it slow as you don't want to mix up this json. **Also note** that your json files may be slightly different from the ones you see below; don't worry, just focus on the changes we are making to parameterize and you should be ok.

```
$ cat php-mysql-template.json 
{
    "kind": "Template",
    "apiVersion": "v1",
    "metadata": {
        "name": "php-mysql-template",
        "creationTimestamp": null
    },
    "labels": {
      "createdBy": "php-mysql-template"
    },
    "objects": [
        {
            "kind": "ImageStream",
            "apiVersion": "v1",
            "metadata": {
                "name": "${APPLICATION_NAME}",
                "generation": 1,
                "creationTimestamp": null,
                "labels": {
                    "tier": "frontend"
                },
                "annotations": {
                    "openshift.io/generated-by": "OpenShiftNewApp"
                }
            },
            "spec": {},
            "status": {
                "dockerImageRepository": ""
            }
        },
        {
            "kind": "BuildConfig",
            "apiVersion": "v1",
            "metadata": {
                "name": "${APPLICATION_NAME}",
                "creationTimestamp": null,
                "labels": {
                    "tier": "frontend"
                },
                "annotations": {
                    "openshift.io/generated-by": "OpenShiftNewApp"
                }
            },
            "spec": {
                "triggers": [
                    {
                        "type": "GitHub",
                        "github": {
                            "secret": "${GITHUB_TRIGGER_SECRET}"
                        }
                    },
                    {
                        "type": "Generic",
                        "generic": {
                            "secret": "${GENERIC_TRIGGER_SECRET}"
                        }
                    },
                    {
                        "type": "ConfigChange"
                    },
                    {
                        "type": "ImageChange",
                        "imageChange": {}
                    }
                ],
                "runPolicy": "Serial",
                "source": {
                    "type": "Git",
                    "git": {
                         "uri": "${GIT_URI}",
                         "ref": "${GIT_REF}"
                    }
                },
                "strategy": {
                    "type": "Source",
                    "sourceStrategy": {
                        "from": {
                            "kind": "ImageStreamTag",
                            "namespace": "openshift",
                            "name": "php:5.6"
                        }
                    }
                },
                "output": {
                    "to": {
                        "kind": "ImageStreamTag",
                        "name": "${APPLICATION_NAME}:latest"
                    }
                },
                "resources": {},
                "postCommit": {}
            }
        },
        {
            "kind": "DeploymentConfig",
            "apiVersion": "v1",
            "metadata": {
                "name": "${APPLICATION_NAME}",
                "generation": 1,
                "creationTimestamp": null,
                "labels": {
                    "tier": "frontend"
                },
                "annotations": {
                    "openshift.io/generated-by": "OpenShiftNewApp"
                }
            },
            "spec": {
                "strategy": {
                    "type": "Rolling",
                    "rollingParams": {
                        "updatePeriodSeconds": 1,
                        "intervalSeconds": 1,
                        "timeoutSeconds": 600,
                        "maxUnavailable": "25%",
                        "maxSurge": "25%"
                    },
                    "resources": {}
                },
                "triggers": [
                    {
                        "type": "ConfigChange"
                    },
                    {
                        "type": "ImageChange",
                        "imageChangeParams": {
                            "automatic": true,
                            "containerNames": [
                                "${APPLICATION_NAME}"
                            ],
                            "from": {
                                "kind": "ImageStreamTag",
                                "name": "${APPLICATION_NAME}:latest"
                            }
                        }
                    }
                ],
                "replicas": 1,
                "test": false,
                "selector": {
                    "deploymentconfig": "${APPLICATION_NAME}",
                    "tier": "frontend"
                },
                "template": {
                    "metadata": {
                        "creationTimestamp": null,
                        "labels": {
                            "deploymentconfig": "${APPLICATION_NAME}",
                            "tier": "frontend"
                        },
                        "annotations": {
                            "openshift.io/container.${APPLICATION_NAME}.image.entrypoint": "[\"container-entrypoint\",\"/bin/sh\",\"-c\",\"$STI_SCRIPTS_PATH/usage\"]",
                            "openshift.io/generated-by": "OpenShiftNewApp"
                        }
                    },
                    "spec": {
                        "containers": [
                            {
                                "name": "${APPLICATION_NAME}",
                                "image": "${APPLICATION_NAME}:latest",
                                "ports": [
                                    {
                                        "containerPort": 8080,
                                        "protocol": "TCP"
                                    }
                                ],
                                "env": [
                                    {
                                        "name": "MYSQL_DATABASE",
                                        "value": "${MYSQL_DATABASE}"
                                    },
                                    {
                                        "name": "MYSQL_PASSWORD",
                                        "value": "${MYSQL_PASSWORD}"
                                    },
                                    {
                                        "name": "MYSQL_USER",
                                        "value": "${MYSQL_USER}"
                                    }
                                ],
                                "resources": {},
                                "terminationMessagePath": "/dev/termination-log",
                                "imagePullPolicy": "Always"
                            }
                        ],
                        "restartPolicy": "Always",
                        "terminationGracePeriodSeconds": 30,
                        "dnsPolicy": "ClusterFirst",
                        "securityContext": {}
                    }
                }
            }
        },
        {
            "kind": "DeploymentConfig",
            "apiVersion": "v1",
            "metadata": {
                "name": "${DATABASE_SERVICE_NAME}",
                "generation": 2,
                "creationTimestamp": null,
                "labels": {
                    "tier": "database"
                },
                "annotations": {
                    "openshift.io/generated-by": "OpenShiftNewApp"
                }
            },
            "spec": {
                "strategy": {
                    "type": "Rolling",
                    "rollingParams": {
                        "updatePeriodSeconds": 1,
                        "intervalSeconds": 1,
                        "timeoutSeconds": 600,
                        "maxUnavailable": "25%",
                        "maxSurge": "25%"
                    },
                    "resources": {}
                },
                "triggers": [
                    {
                        "type": "ConfigChange"
                    },
                    {
                        "type": "ImageChange",
                        "imageChangeParams": {
                            "automatic": true,
                            "containerNames": [
                                "mysql"
                            ],
                            "from": {
                                "kind": "ImageStreamTag",
                                "namespace": "openshift",
                                "name": "mysql:5.6"
                            }
                        }
                    }
                ],
                "replicas": 1,
                "test": false,
                "selector": {
                    "deploymentconfig": "${DATABASE_SERVICE_NAME}",
                    "tier": "database"
                },
                "template": {
                    "metadata": {
                        "creationTimestamp": null,
                        "labels": {
                            "deploymentconfig": "${DATABASE_SERVICE_NAME}",
                            "tier": "database"
                        },
                        "annotations": {
                            "openshift.io/container.${DATABASE_SERVICE_NAME}.image.entrypoint": "[\"container-entrypoint\",\"run-mysqld\"]",
                            "openshift.io/generated-by": "OpenShiftNewApp"
                        }
                    },
                    "spec": {
                        "volumes": [
                            {
                                "name": "mysql-volume-1",
                                "emptyDir": {}
                            }
                        ],
                        "containers": [
                            {
                                "name": "mysql",
                                "image": "registry.access.redhat.com/rhscl/mysql-56-rhel7@sha256:0a710d858de8752b70c4fbebb7c157050549a0cfc1b2e52301a20990648d02a7",
                                "ports": [
                                    {
                                        "containerPort": 3306,
                                        "protocol": "TCP"
                                    }
                                ],
                                "env": [
                                    {
                                        "name": "MYSQL_DATABASE",
                                        "value": "${MYSQL_DATABASE}"
                                    },
                                    {
                                        "name": "MYSQL_PASSWORD",
                                        "value": "${MYSQL_PASSWORD}"
                                    },
                                    {
                                        "name": "MYSQL_USER",
                                        "value": "${MYSQL_USER}"
                                    }
                                ],
                                "resources": {},
                                "volumeMounts": [
                                    {
                                        "name": "${DATABASE_SERVICE_NAME}-volume-1",
                                        "mountPath": "/var/lib/mysql/data"
                                    }
                                ],
                                "terminationMessagePath": "/dev/termination-log",
                                "imagePullPolicy": "IfNotPresent"
                            }
                        ],
                        "restartPolicy": "Always",
                        "terminationGracePeriodSeconds": 30,
                        "dnsPolicy": "ClusterFirst",
                        "securityContext": {}
                    }
                }
            }
        },
        {
            "kind": "Service",
            "apiVersion": "v1",
            "metadata": {
                "name": "${APPLICATION_NAME}",
                "creationTimestamp": null,
                "labels": {
                    "tier": "frontend"
                },
                "annotations": {
                    "openshift.io/generated-by": "OpenShiftNewApp"
                }
            },
            "spec": {
                "ports": [
                    {
                        "name": "8080-tcp",
                        "protocol": "TCP",
                        "port": 8080,
                        "targetPort": 8080
                    }
                ],
                "selector": {
                    "deploymentconfig": "${APPLICATION_NAME}",
                    "tier": "frontend"
                },
                "type": "ClusterIP",
                "sessionAffinity": "None"
            }
        },
        {
            "kind": "Service",
            "apiVersion": "v1",
            "metadata": {
                "name": "${DATABASE_SERVICE_NAME}",
                "creationTimestamp": null,
                "labels": {
                    "tier": "database"
                },
                "annotations": {
                    "openshift.io/generated-by": "OpenShiftNewApp"
                }
            },
            "spec": {
                "ports": [
                    {
                        "name": "3306-tcp",
                        "protocol": "TCP",
                        "port": 3306,
                        "targetPort": 3306
                    }
                ],
                "selector": {
                    "deploymentconfig": "${DATABASE_SERVICE_NAME}",
                    "tier": "database"
                },
                "type": "ClusterIP",
                "sessionAffinity": "None"
            },
            "status": {
                "loadBalancer": {}
            }
        },
        {
            "kind": "Route",
            "apiVersion": "v1",
            "metadata": {
                "name": "${APPLICATION_NAME}",
                "creationTimestamp": null,
                "labels": {
                    "tier": "frontend"
                },
                "annotations": {
                    "openshift.io/host.generated": "true"
                }
            },
            "spec": {
                "host": "${APPLICATION_DOMAIN}",
                "to": {
                    "kind": "Service",
                    "name": "{APPLICATION_NAME}",
                    "weight": 100
                },
                "port": {
                    "targetPort": "8080-tcp"
                }
            }
        }
    ],
    "parameters": [
        {
            "name": "APPLICATION_NAME",
            "displayName": "Application Name",
            "description": "The name assigned to all of the frontend objects defined in this template.",
            "value": "php-mysql-example",
            "required": true
        },
        {
            "name": "GIT_URI",
            "displayName": "Git Repository URL",
            "description": "The URL of the repository with your application source code.",
            "value": "https://github.com/VeerMuchandi/dbtest.git",
            "required": true
        },
        {
            "name": "GIT_REF",
            "displayName": "Git Reference",
            "description": "Set this to a branch name, tag or other ref of your repository if you are not using the default branch."
        },
        {
            "name": "APPLICATION_DOMAIN",
            "displayName": "Application HostName",
            "description": "The exposed hostname that will route to the application service, if left blank a value will be defaulted."
        },
        {
            "name": "GITHUB_WEBHOOK_SECRET",
            "displayName": "GitHub Webhook Secret",
            "description": "A secret string used to configure the GitHub webhook.",
            "generate": "expression",
            "from": "[a-zA-Z0-9]{40}"
        },
        {
            "name": "GENERIC_WEBHOOK_SECRET",
            "displayName": "GitHub Webhook Secret",
            "description": "A secret string used to configure the GitHub webhook.",
            "generate": "expression",
            "from": "[a-zA-Z0-9]{40}"
        },
        {
            "name": "DATABASE_SERVICE_NAME",
            "displayName": "Database Service Name",
            "value": "mysql"
        },
        {
            "name": "MYSQL_DATABASE",
            "displayName": "Database Name",
            "value": "sampledb"
        },
        {
            "name": "MYSQL_USER",
            "displayName": "Database User",
            "generate": "expression",
            "from": "[a-zA-Z0-9]{16}"
        },
        {
            "name": "MYSQL_PASSWORD",
            "displayName": "Database Password",
            "generate": "expression",
            "from": "[a-zA-Z0-9]{40}"
        }
    ]
    
}

```

Now your template is all ready. Save the changes to the json file.

#### Test the template

Create a new project. Let's call it templatetest-UserName. **Remember** to substitute the username.

Now, let us add our template to this project.

```
oc create -f php-mysql-template.json
```

This command adds template to your project and will be local to the project. You cannot use this template from other projects.

You can view the templates by running 

```
oc get templates
NAME                 DESCRIPTION   PARAMETERS     OBJECTS
php-mysql-template                 10 (2 blank)   7
```

Let us use this template from Web Console.

* Go to Web console
* Switch to the project `templatetest-UserName`
* `AddToProject` and you should find the template you just added
* Select the template 
* Study this template and how it looks like
* Fill in the parameters and create your application

This should spin up the MySQL database and start a new build for your application code.


#### Making templates "global"

We will not do this in the lab, but just adding this step here for completeness.

Now that you tested and you are happy with the template, you can make it available for everyone using your OpenShift cluster to use. You need to send your template file to your cluster administrator.

The cluster administrator would run the following command to make the template available to everyone.

```
$ oc create -f php-mysql-template.json -n openshift
```

Adding to the `openshift` project will make the template globally available on your OpenShift Cluster.



Congratulations!! Now you know how to create your own templates and use them.



 






