Service Mesh
A service mesh is a networking layer that is designed to help manage communication between applications in a microservice architecture by providing a single, unified solution to the following problems:
Security
In Part 6, you deployed microservices in Kubernetes that were able to talk to each other via HTTP requests. In fact, not only could these microservices talk to each other, but anyonecould talk to them, as these services responded blindly to any HTTP request that came in. Putting these microservices in a private network provides some protection (the castle-and-moat model), but as your company scales, you will most likely want to harden the security around your services (the zero-trust model) by enforcing encryption, authentication, and authorization.
Observability
As you saw in Part 6, microservice architectures introduce many new failure modes and moving parts that can make debugging harder than with a monolith. In a large services architecture, understanding how a single request is processed can be a challenge, as that one request may result in dozens of API calls to dozens of services. This is where observability tools such as distributed tracing, metrics, and logging become essential. You’ll learn more about these topics in Part 10.
Resiliency
If you’re at the scale where you are running many services, you’re at a scale where bugs, performance issues, and other errors happen many times per day. If you had to deal with every issue manually, you’d never be able to sleep. In order to have a maintainable and resilient microservice architecture, there are a number of tools and techniques you can use to recover from or avoid errors automatically, including retries, timeouts, circuit breakers, and rate limiting.
Traffic management
As you saw in Part 6, breaking a monolith into services means you are now managing a distributed system. With distributed systems, you often need a lot of fine-grained control over network traffic, including load balancing between services, canary deployments (as you saw in Part 5), and traffic mirroring (i.e., sending a duplicate of traffic to an extra endpoint for analysis or testing).
Almost all of these are problems of scale. If you only have two or three services, a small team, and not a lot of load, these problems are not likely to affect you, and a service mesh may be an unnecessary overhead. If you have hundreds of services owned by dozens of teams and high load, these are problems you’ll be dealing with every day. If you try to solve these problems one at a time, you’ll find it is a huge amount of work, and that the solution to one has an impact on the other: e.g., how you manage encryption affects your ability to do tracing and traffic mirroring. Moreover, the simple solutions you’re likely to try first may require you to make code changes to every single app, and as you learned in Part 6, rolling out global changes across many services can take a long time.
This is where a service mesh can be of use. It gives you an integrated, all-in-one solution to these problems, and just as importantly, it can solve most of these problems in a way that is transparent, and does not require you to change your app code.
    Key takeaway #7
A service mesh can improve security, observability, resiliency, and traffic management in a microservices architecture, without having to update the application code of each service.
When things are working, a service mesh can feel like a magical way to upgrade the security and debuggability of your microservice architecture. However, when things aren’t working, the service mesh itself can be difficult to debug, as it introduces many new moving parts (encryption, authentication, authorization, routing, firewalls, tracing). Moreover, understanding, installing, configuring, and managing a service mesh can be a lot of overhead. If you’re at the scale where you need solutions to the problems listed earlier, it’s worth it; if you’re a tiny startup, it’ll only slow you down.
Service mesh tools can be broken down into three buckets. The first bucket is the service mesh tools designed for use with Kubernetes, which includes Linkerd (the project that coined the term "service mesh"), Istio, and Cilium. The second bucket is managed service mesh tools from cloud providers, such as AWS App Mesh and Google Cloud Service Mesh. The third bucket is service mesh tools that can be used with any orchestration approach (e.g., Kubernetes, EC2, on-prem servers, etc.), such as Consul service mesh and Kuma (full list).
The best way to get a feel for what a service mesh does is to try one out, so let’s go through an example of using Istio with Kubernetes.
Example: Istio Service Mesh with Kubernetes Microservices
Istio is a popular service mesh for Kubernetes that was originally created by Google, IBM, and Lyft, and open sourced in 2017. Let’s see how Istio can help you manage the two microservices you deployed with Kubernetes in Part 6. One of those microservices was a backend app that exposed a simple JSON-over-HTTP REST API and the other microservice was a frontend app that made service calls to the backend, using the service discovery mechanism built into Kubernetes, and then rendered the data it got back using HTML. First, make a copy of those two sample apps into the folder you’re using for this blog post’s examples:
$ cd fundamentals-of-devops

$ cp -r ch6/sample-app-frontend ch7/

$ cp -r ch6/sample-app-backend ch7/
Second, you’ll need a Kubernetes cluster. The easiest one to use for learning and testing is the one that comes with Docker Desktop, so just as you did in Part 3, fire up that cluster, and make sure you’re authenticated to it:
$ kubectl config use-context docker-desktop
Third, download and install the latest Istio release (minimum version 1.22). The release will be in a folder called istio-<VERSION>, where <VERSION> is the version of Istio you installed, and it will include a samples folder which has some useful sample code (which you’ll use shortly), and istioctl, a CLI tool that has useful helper functions for working with Istio (which you should add to your PATH). Use istioctl to install Istio in your Kubernetes cluster as follows:
$ istioctl install --set profile=minimal -y
This uses a minimal profile to install Istio, which is good enough for learning and testing (see theinstall instructions for profiles you can use for production). The way Istio works is to inject its own sidecar into every Pod you deploy into Kubernetes. That sidecar is what provides all the security, observability, resiliency, and traffic management features, without you having to change your application code. To configure Istio to inject its sidecar into all Pods you deploy into the default namespace, run the following command:
$ kubectl label namespace default istio-injection=enabled
Istio supports a number of integrations with observability tools. For this example, let’s use the sample add-ons that come with the Istio release, which include a dashboard for Istio called Kiali, a database for monitoring data called Prometheus, a UI for visualizing monitoring data called Grafana, and a distributed tracing tool called Jaeger:
$ cd istio-<VERSION>

$ kubectl apply -f samples/addons

$ kubectl rollout status deployment/kiali -n istio-system
At this point, you can verify everything is installed correctly by running the verify-installcommand:
$ istioctl verify-install
If everything looks good, deploy the frontend and backend apps as you did before:
$ cd ../sample-app-backend

$ kubectl apply -f sample-app-deployment.yml

$ kubectl apply -f sample-app-service.yml

$ cd ../sample-app-frontend

$ kubectl apply -f sample-app-deployment.yml

$ kubectl apply -f sample-app-service.yml
After a few seconds, you should be able to make a request to the frontend as follows:
$ curl localhost

<p>Hello from <b>backend microservice</b>!</p>
At this point, everything should be working exactly as before. So is Istio doing anything? One way to find out is to open up the Kiali dashboard you installed earlier:
$ istioctl dashboard kiali
This command will open the dashboard in your web browser. Click Traffic Graph in the nav on the left, and you should see something similar to Figure 69:
 
Figure 69. The traffic graph in Istio’s Kiali dashboard
If the Traffic Graph doesn’t show you anything, run curl localhost several more times, and then click the refresh button in the top right of the dashboard. You should see a visualization of the path your requests take through your microservices, including through the Services and Pods. Right away, you see one of the key benefits of a service mesh: observability. You get not only this service mesh visualization, but also aggregated logs (click on Workloads in the left nav, select sample-app-backend, and click the Logs tab), metrics (run istioctl dashboard grafana), and distributed traces (run istioctl dashboard jaeger). When you’re done experimenting with Kiali, hit CTRL+C to exit.
Another key benefit of service meshes is security, including support for automatically encrypting, authenticating, and authorizing all requests within the service mesh. By default, to make it possible to install Istio without breaking everything, Istio initially allows unencrypted, unauthenticated, and unauthorized requests to go through. However, you can change this by configuring policies in Istio. Create a new folder called istio, and within it, a file called istio-auth.yml, with the content shown in Example 132:
Example 132. Istio authentication and authorization policies (ch7/istio/istio-auth.yml)
apiVersion: security.istio.io/v1beta1

kind: PeerAuthentication               (1)

metadata:

  name: require-mtls

  namespace: default

spec:

  mtls:

    mode: STRICT



---                                    (2)

apiVersion: security.istio.io/v1

kind: AuthorizationPolicy              (3)

metadata:

  name: allow-nothing

  namespace: default

spec:

  {}
This code does the following:
1	Create an authentication policy that requires all service calls to use mTLS (mutual TLS), which is a way to enforce that every connection is encrypted and authenticated (you’ll learn more about TLS inPart 8). One of the benefits of Istio is that it handles mTLS for you, completely transparently.
2	Note the use of ---: this is a divider that allows you to put multiple Kubernetes configurations in a single YAML file.
3	Create an authorization policy which blocks all service calls by default, so your services don’t respond to anyone who happens to have network access. You can then add additional authorization policies to allow just the service communication that you know is valid.
Deploy these policies as follows:
$ cd ../istio

$ kubectl apply -f istio-auth.yml
Now, look what happens if you try to access the frontend app again:
$ curl localhost

curl: (52) Empty reply from server
Since your request to the frontend wasn’t using mTLS, Istio rejected the connection immediately. Enforcing mTLS makes sense for backends, as they should only be accessible to other services, but your frontend should be accessible to users outside your company, so you can disable the mTLS requirement for the frontend as shown in Example 133:
Example 133. Authentication policy to disable the mTLS requirement for the frontend (ch7/sample-app-frontend/kubernetes-config.yml)
apiVersion: security.istio.io/v1beta1

kind: PeerAuthentication

metadata:

  name: allow-without-mtls

  namespace: default

spec:

  selector:

    matchLabels:

      app: sample-app-frontend-pods (1)

  mtls:

    mode: DISABLE                   (2)
This is an authentication policy that works as follows:
1	Target the frontend Pods.
2	Disable the mTLS requirement for the frontend Pods.
You can put the YAML in Example 133 into a new YAML file, but dealing with too many YAML files for the frontend is tedious and error-prone. Let’s instead use the --- divider to combine the frontend’s sample-app-deployment.yml, sample-app-service.yml, and the YAML you just saw in Example 133 into a single file called kubernetes-config.yml, with the structure shown in Example 134:
Example 134. Combine multiple Kubernetes configurations into a single YAML file (ch7/sample-app-frontend/kubernetes-config.yml)
apiVersion: apps/v1

kind: Deployment

# ... (other params omitted) ...



---

apiVersion: v1

kind: Service

# ... (other params omitted) ...





---

apiVersion: security.istio.io/v1beta1

kind: PeerAuthentication

# ... (other params omitted) ...
With all of your YAML in a single kubernetes-config.yml, you can delete the sample-app-deployment.yml and sample-app-service.yml files, and deploy changes to the frontend app with a single call to kubectl apply:
$ cd ../sample-app-frontend

$ kubectl apply -f kubernetes-config.yml
Try accessing the frontend again, adding the --write-out flag so that curl prints the HTTP response code after the response body:
$ curl --write-out '\n%{http_code}\n' localhost

RBAC: access denied

403
You got an error again, but this time, it’s a different error. That’s because there are two policies at play: an authentication policy and an authorization policy. You added an authentication policy that allows the frontend to be accessed without mTLS, so Istio is no longer blocking your request entirely, but you still got a 403 response code (Forbidden) because the allow-nothingauthorization policy is still blocking all requests. To fix this, you need to add authorization policies to the backend and the frontend.
This requires that Istio has some way to identify the frontend and backend. Istio uses Kubernetes service accounts as identities, automatically providing a TLS certificate to each application based on its service account, and using mTLS to provide mutual authentication (i.e., the backend will verify the request is coming from the frontend, and the frontend will verify it is really talking to the backend). Istio will handle all the TLS details for you, so all you need to do is associate the frontend and backend with their own service accounts and add an authorization policy to each one.
Start with the frontend, updating its kubernetes-config.yml as shown in Example 135:
Example 135. Configure the frontend with a service account and authorization policy (ch7/sample-app-frontend/kubernetes-config.yml)
apiVersion: apps/v1

kind: Deployment

spec:

  replicas: 2

  template:

    metadata:

      labels:

        app: sample-app-frontend-pods

    spec:

      serviceAccountName: sample-app-frontend-service-account (1)

      containers:

        - name: sample-app-frontend

# ... (other params omitted) ...



---

apiVersion: v1

kind: ServiceAccount

metadata:

  name: sample-app-frontend-service-account                   (2)



---

apiVersion: security.istio.io/v1

kind: AuthorizationPolicy                                     (3)

metadata:

  name: sample-app-frontend-allow-http

spec:

  selector:

    matchLabels:

      app: sample-app-frontend-pods                           (4)

  action: ALLOW                                               (5)

  rules:                                                      (6)

  - to:

    - operation:

        methods: ["GET"]
Here are the updates to make to the frontend:
1	Configure the frontend’s Deployment to use the service account created in (2).
2	Create a service account for the frontend.
3	Add an authorization policy for the frontend.
4	The authorization policy targets the frontend’s Pods.
5	The authorization policy will allow requests that match the rules in (6).
6	Define rules for the authorization policy, where each rule can optionally contain from (sources) and to (destinations) to match. The preceding code allows the frontend to receive HTTP GET requests from all sources.
Run apply to deploy these changes to the frontend:
$ kubectl apply -f kubernetes-config.yml
Next, head over to the backend, and combine its Deployment and Service definitions into a single kubernetes-config.yml file, separated by ---, just like you did for the frontend (and then delete sample-app-deployment.yml and sample-app-service.yml). Once that’s done, update the backend’s kubernetes-config.yml as shown in Example 136:
Example 136. Configure the backend with a service account and authorization policy (ch7/sample-app-backend/kubernetes-config.yml)
apiVersion: apps/v1

kind: Deployment

spec:

  replicas: 3

  template:

    metadata:

      labels:

        app: sample-app-backend-pods

    spec:

      serviceAccountName: sample-app-backend-service-account (1)

      containers:

        - name: sample-app-backend

# ... (other params omitted) ...



---

apiVersion: v1

kind: ServiceAccount

metadata:

  name: sample-app-backend-service-account                   (2)



---

apiVersion: security.istio.io/v1                             (3)

kind: AuthorizationPolicy

metadata:

  name: sample-app-backend-allow-frontend

spec:

  selector:

    matchLabels:

      app: sample-app-backend-pods                           (4)

  action: ALLOW

  rules:                                                     (5)

  - from:

    - source:

        principals:

          - "cluster.local/ns/default/sa/sample-app-frontend-service-account"

    to:

    - operation:

        methods: ["GET"]
Here are the updates to make to the backend:
1	Configure the backend’s Deployment to use the service account created in (2).
2	Create a service account for the backend.
3	Add an authorization policy for the backend.
4	Apply the authorization policy to the backend’s Pods.
5	Define rules that allow HTTP GET requests to the backend from the service account of the frontend.
Run apply to deploy these changes to the backend:
$ cd ../sample-app-backend

$ kubectl apply -f kubernetes-config.yml
And now test the frontend one more time:
$ curl --write-out '\n%{http_code}\n' localhost

<p>Hello from <b>backend microservice</b>!</p>

200
Congrats, you got a 200 (OK) response code and the expected HTML response body, which means you now have microservices running in Kubernetes, using service discovery, and communicating securely via a service mesh! With the authentication and authorization policies you have in place, you have significantly improved your security posture: all communication between services (such as the request the frontend successfully made to the backend) is now encrypted, authenticated, and authorized—all without you having to modify the Node.js source code of either app. Moreover, you have access to all the other service mesh benefits, too: observability, resiliency, and traffic management.
    Get your hands dirty
Here are a few exercises you can try at home to go deeper:
•	Try out some of Istio’s traffic management functionality, such a request timeouts, circuit breaking, and traffic shifting.
•	Consider if Istio’s ambient mode is a better fit for your workloads than the default sidecar mode.
When you’re done testing, you can run delete on the kubernetes-config.yml files of the frontend and backend to clean up the apps. If you wish to uninstall Istio, first, remove the global authorization and authentication policies:
$ cd ../istio

$ kubectl delete -f istio-auth.yml
Next uninstall the addons:
$ cd ../istio-<VERSION>

$ kubectl delete -f samples/addons
And finally, uninstall Istio itself, including deleting its namespace, and removing the default labeling behavior:
$ istioctl uninstall -y --purge

$ kubectl delete namespace istio-system

$ kubectl label namespace default istio-injection-
One of the benefits of software-defined networking is that it’s fast and easy to try out different networking approaches. Instead of having to spend hours or days setting up physical routers, switches, and cables, you can try out a tool like Istio in minutes, and if it doesn’t work for you, it only takes a few more minutes to uninstall Istio, and try something else.
Conclusion
You’ve now seen the central role networking plays in connectivity and security, as per the 7 key takeaways from this blog post:
•	You get public IP addresses from network operators such as cloud providers and ISPs.
•	DNS allows you to access web services via memorable, human-friendly, consistent names.
•	Use a defense-in-depth strategy to ensure you’re never one mistake away from a disaster.
•	Deploy all your servers into private networks by default, exposing only a handful of locked-down servers to the public Internet.
•	In the castle-and-moat model, you create a strong network perimeter to protect all the resources in your private network; in the zero-trust model, you create a strong perimeter around each individual resource.
•	As soon as you have more than one service, you will need to figure out a service discovery solution.
•	A service mesh can improve security, observability, resiliency, and traffic management in a microservices architecture, without having to update the application code of each service.
Putting these all together, you should now be able to picture the full network architecture you’re aiming for, as shown in Figure 70. Inside your data center, you have a private network, such as a VPC. Within this network, almost all of your servers are in private subnets. The only exceptions are highly-locked down servers designed to accept traffic directly from customers, such as load balancers, and highly-locked down bastion hosts for your employees, such as an access proxy.
 
Figure 70. Full network architecture
When a customer visits your website, their computer looks up your domain name using DNS, gets the public IP addresses of your load balancers, makes a request to one of those IPs, and the load balancer routes that request to an app server in the private subnets. That app server processes the request, communicates with other services—using service discovery to find those services, and a service mesh to enforce authentication, authorization, and encryption—and returns a response. When an employee needs to access something on your internal network, such as a wiki, they authenticate to the access proxy, which checks the user, their device, and access policies, and if the employee is authorized, the proxy gives them access to just that wiki.
As you went through this blog post, you repeatedly came across several key security concepts such as authentication and secrets. These concepts affect not only networking, but all aspects of software delivery, so let’s move on to Part 8, where we do a deeper dive on security.

