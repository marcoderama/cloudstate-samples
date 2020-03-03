# Building and deploying the Sample application

### Fiends Service
```
cd friends
npm install
npm run prestart

```

This will compile the protobuf and `user-function.desc`.

Build a docker image with the right registry and tag
```
docker build . -t <my-registry>/friends:latest
```

Push the docker image to the registry
```
docker push <my-registry>/friends:latest
```

Deploy the image by changing into the deploy folder and editing the `friends.yaml` to point to your docker image that you just pushed.
```
cd ../deploy
cat friends.yaml
apiVersion: cloudstate.io/v1alpha1
kind: StatefulService
metadata:
  name: friends
spec:
  containers:
  - image: coreyauger/friends:latest    # <-- Change this to your image
    name: friends
```

Deploy the service to your project namespace
```
kubectl apply -f friends.yaml -n <project_name>
statefulservice.cloudstate.io/friends created
````

### presence Service
```
cd ../presence
npm install
npm run prestart

```

This will compile the protobuf and `user-function.desc`
Build a docker image with the right registry and tag
```
docker build . -t <my-registry>/presence:latest
```

Push the docker image to the registry
```
docker push <my-registry>/presence:latest
```

Deploy the image by changing into the deploy folder and editing the `presence.yaml` to point to your docker image that you just pushed.
```
cd ../deploy
cat presence.yaml
apiVersion: cloudstate.io/v1alpha1
kind: StatefulService
metadata:
  name: presence
spec:
  containers:
  - image: coreyauger/presence:latest    # <-- Change this to your image
    name: presence
```

Deploy the service to your project namespace
```
kubectl apply -f presence.yaml -n <project-name>
statefulservice.cloudstate.io/presence created
````

### Verify they are running.
Check that the services are running
```
kubectl get statefulservices -n <project-name>
NAME       REPLICAS   STATUS
friends    1          Ready
presence   1          Ready
```

To redeploy a new image to the cluster you must delete and then redeploy using the yaml file.  
For example if we updated the friends docker image we would do the following.
````
kubectl delete statefulservice friends -n <project-name>
statefulservice.cloudstate.io "friends" deleted
kubectl apply -f friends.yaml -n <project-name>    
statefulservice.cloudstate.io/friends created
````


### Chat
The chat service is a front end web application written in typescript.  It is backed by a `stateless` service that will serve the compiled javacript, html and images.

This service makes `grpc-web` calls directly to the other services to get the data that it needs.  In order to do this we need to compile the proto definitions from the other two services as well as generate the grpc-web clients.  This is all done with a shell script `protogen.sh`.  Let's first run the protogen script, then compile the service definition and finally compile our typescript.

```
cd ../chat
./protogen.sh
npm install
npm run prestart
npm run-script build
```

We now have everything in place to build our docker image
```
docker build . -t <my-registry>/chat:latest
```

Push the docker image to the registry
```
docker push <my-registry>/chat:latest
```

Deploy the image by changing into the deploy folder and editing the `chat.yaml` to point to your docker image that you just pushed.
```
cd ../deploy
cat chat.yaml
apiVersion: cloudstate.io/v1alpha1
kind: StatefulService
metadata:
  name: chat
spec:
  containers:
  - image: coreyauger/chat:latest    # <-- Change this to your image
    name: chat
```

Deploy the service to your project namespace
```
kubectl apply -f chat.yaml -n <project-name>
statefulservice.cloudstate.io/chat created
```

## Routes
The last thing that is required is to provide the public routes needed for both the front end and grpc-web calls.  These exist in the `routes.yaml` file.

```
cat routes.yaml
apiVersion: cloudstate.io/v1alpha1
kind: Route
metadata:
  name: "chat-routes"
spec:
  http:
  - name: "friends-routes"
    match:
    - uri:
        prefix: "/cloudstate.samples.chat.friends.Friends/"
    route:
      service: friends
  - name: "presence-routes"
    match:
    - uri:
        prefix: "/cloudstate.samples.chat.presence.Presence/"
    route:
      service: presence
  - name: "chat-routes"
    match:
    - uri:
        prefix: "/"
    route:
      service: chat
```

Add these routes by performing
```
kubectl apply -f routes.yaml -n <project-name>
```
