This lab builds upon lab 1

## 7. Build the Docker Images and push them to ECR

First create the ECR repositories for the 2 applications.

```
aws ecr create-repository --repository-name colorteller

aws ecr create-repository --repository-name colorgateway
```

In the terminal of Cloud9, clone the code

```
git clone https://github.com/tohwsw/aws-app-mesh-examples.git
```

Retrieve the login command to use to authenticate your Docker client to your registry.

```
$(aws ecr get-login --no-include-email --region ap-southeast-1)
```

Go to the folder examples/apps/colorapp/src/colorteller. Execute a docker build with the respective repository uri for colorteller and push it to the repository.

```
docker build -t 284245693010.dkr.ecr.ap-southeast-1.amazonaws.com/colorteller .

docker push 284245693010.dkr.ecr.ap-southeast-1.amazonaws.com/colorteller:latest
```

Go to the folder examples/apps/colorapp/src/gateway. Execute a docker build with the respective repository uri for colorgateway and push it to the repository.

```
docker build -t 284245693010.dkr.ecr.ap-southeast-1.amazonaws.com/colorgateway .

docker push 284245693010.dkr.ecr.ap-southeast-1.amazonaws.com/colorgateway:latest
```

## 8. Create AppMesh and the required configuration

The following diagram represents the abstract view in terms of App Mesh resources:

![img1]

[img1]:https://github.com/tohwsw/aws-eks-workshop/blob/master/Lab2-AppMesh-with-ColorTeller/img/appmesh.png

Create the App Mesh

```
aws appmesh create-mesh --mesh-name APP_MESH_DEMO
```

Create the virtual nodes

```
aws appmesh create-virtual-node  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "spec": {
        "listeners": [
            {
                "portMapping": {
                    "port": 9080,
                    "protocol": "http"
                }
            }
        ],
        "serviceDiscovery": {
            "dns": {
                "hostname": "colorgateway.default.svc.cluster.local"
            }
        },
        "backends": [
			{
                "virtualService": {
                    "virtualServiceName": "colorteller.default.svc.cluster.local"
                }
            }          
        ]
    },
    "virtualNodeName": "colorgateway-vn"
}'

aws appmesh create-virtual-node  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "spec": {
        "listeners": [
            {
                "portMapping": {
                    "port": 9080,
                    "protocol": "http"
                }
            }
        ],
        "serviceDiscovery": {
            "dns": {
                "hostname": "colorteller.default.svc.cluster.local"
            }
        }
    },
    "virtualNodeName": "colorteller-vn"
}'

```

Copy and paste these three virtual node definitions into your terminal to create the black, blue, and red collorteller virtual nodes:

```
aws appmesh create-virtual-node  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "spec": {
        "listeners": [
            {
                "portMapping": {
                    "port": 9080,
                    "protocol": "http"
                }
            }
        ],
        "serviceDiscovery": {
            "dns": {
                "hostname": "colorteller-black.default.svc.cluster.local"
            }
        }
    },
    "virtualNodeName": "colorteller-black-vn"
}'

aws appmesh create-virtual-node  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "spec": {
        "listeners": [
            {
                "portMapping": {
                    "port": 9080,
                    "protocol": "http"
                }
            }
        ],
        "serviceDiscovery": {
            "dns": {
                "hostname": "colorteller-blue.default.svc.cluster.local"
            }
        }
    },
    "virtualNodeName": "colorteller-blue-vn"
}'


aws appmesh create-virtual-node  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "spec": {
        "listeners": [
            {
                "portMapping": {
                    "port": 9080,
                    "protocol": "http"
                }
            }
        ],
        "serviceDiscovery": {
            "dns": {
                "hostname": "colorteller-red.default.svc.cluster.local"
            }
        }
    },
    "virtualNodeName": "colorteller-red-vn"
}'
```


Create the virtual routers
```
aws appmesh create-virtual-router  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "spec": {
        "listeners": [
            {
                "portMapping": {
                    "port": 9080,
                    "protocol": "http"
                }
            }
        ]
    },
    "virtualRouterName": "colorgateway-vr"
}'

aws appmesh create-virtual-router  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "spec": {
        "listeners": [
            {
                "portMapping": {
                    "port": 9080,
                    "protocol": "http"
                }
            }
        ]
    },
    "virtualRouterName": "colorteller-vr"
}'
```

Create the routes

```
aws appmesh create-route  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "routeName": "colorgateway-route",
    "spec": {
        "httpRoute": {
            "action": {
                "weightedTargets": [
                    {
                        "virtualNode": "colorgateway-vn",
                        "weight": 100
                    }
                ]
            },
            "match": {
                "prefix": "/"
            }
        }
    },
    "virtualRouterName": "colorgateway-vr"
}'

aws appmesh create-route  --mesh-name APP_MESH_DEMO --cli-input-json '{
   "routeName": "colorteller-route",
   "spec": {
       "httpRoute": {
           "action": {
               "weightedTargets": [
                   {
                       "virtualNode": "colorteller-vn",
                       "weight": 1
                   }
               ]
           },
           "match": {
               "prefix": "/"
           }
       }
   },
   "virtualRouterName": "colorteller-vr"
}'
```

Create the virtual service

```
aws appmesh create-virtual-service  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "meshName": "APP_MESH_DEMO", 
        "virtualServiceName": "colorgateway.default.svc.cluster.local", 
        "spec": {
            "provider": {
                "virtualRouter": {
                    "virtualRouterName": "colorgateway-vr"
                }
            }
        }
}'


aws appmesh create-virtual-service --mesh-name APP_MESH_DEMO --cli-input-json '{
    "meshName": "APP_MESH_DEMO", 
        "virtualServiceName": "colorteller.default.svc.cluster.local", 
        "spec": {
            "provider": {
                "virtualRouter": {
                    "virtualRouterName": "colorteller-vr"
                }
            }
        }
}'


```

Up to this point, we’ve created all required components of the app mesh (vitual services, virtual nodes, virtual routers, and routes) to support our application.

## 9. Create AppMesh and the required configuration

To deploy the app, download color.yaml and and deploy it.

**Important** You will need to update color.yaml with the ECR respository uri of your colorgateway and colorteller docker images.

```
curl -O https://raw.githubusercontent.com/tohwsw/aws-eks-workshop/master/Lab2-AppMesh-with-ColorTeller/colorapp.yaml

kubectl apply -f colorapp.yaml
```

In addition to deploying the application, we’ll also deploy Curler, which is a simple image that provide curl functionality. To deploy the curler pods, copy and paste the following:

```
kubectl run -it curler --image=tutum/curl /bin/bash
```

You can verify that the pods are running properly.

```
kubectl get pods

```

## 9. Test the application

In the terminal, run the following command to connect to the curler pod with a bash shell:

```
kubectl run -it curler --image=tutum/curl /bin/bash
```

Next, with a shell open to the curler pod, paste the following to repeatedly request the colorgateway service:

```
while [ 1 ]; do  curl -s --connect-timeout 2 http://colorgateway.default.svc.cluster.local:9080/color;echo;sleep 1; done
```

Every second, you should see the response white.

This is because colorgateway always forwards to colorteller, which via the colorteller-route, always routes to colorteller-vn (which will always respond with white).

Let’s modify the colorteller-route so it instead routes to the blue, red, and black colorteller virtual nodes, each at a 30% weighted ratio.

To modify colorteller-route, copy and paste the following:

```
aws appmesh update-route  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "routeName": "colorteller-route",
    "spec": {
        "httpRoute": {
            "action": {
                "weightedTargets": [
                    {
                            "virtualNode": "colorteller-blue-vn",
                            "weight": 3
                        },
                        {
                            "virtualNode": "colorteller-red-vn",
                            "weight": 3
                        },
                        {
                            "virtualNode": "colorteller-black-vn",
                            "weight": 3
                        }
                ]
            },
            "match": {
                "prefix": "/"
            }
        }
    },
    "virtualRouterName": "colorteller-vr"
}'
```

If you look at the curler terminal, you should now see an equal distribution of traffic to the blue, red, and black virtual nodes. This shows that App Mesh is now controlling the distribution of traffic!

## 10. Lab Cleanup

To clean up the lab, please delete the CloudFromation stacks in the console.











