# istio-troubleshooting
Troubleshooting Istio with demo

# Create self signed certificate

1. Create a root certificate and private key to sign the certificate for your services
~~~
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
~~~
2. Create a certificate and a private key for nginx.example.com
~~~
openssl req -out nginx.example.com.csr -newkey rsa:2048 -nodes -keyout nginx.example.com.key -subj "/CN=nginx.example.com/O=some organization"
openssl x509 -req -sha256 -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in nginx.example.com.csr -out nginx.example.com.crt
~~~

# Deploy an NGINX server

1. Create a Kubernetes Secret to hold the certificates that will help in the connection termination
~~~
oc create secret tls nginx-server-certs --key nginx.example.com.key --cert nginx.example.com.crt -n istio-system
~~~
2. Deploy the nginx application in the appropriate dataplane namespace. Two annotations are being added to the deployment
   > oc create -f nginx-deployment.yaml
~~~
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: dataplane
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      annotations:
        sidecar.istio.io/inject: 'true'
        sidecar.jaegertracing.io/inject: 'true'
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: quay.io/rhn_support_rrajaram/nginx:latest
        ports:
        - containerPort: 30210
          containerPort: 30211

~~~
3. Deploy the associated service 
   > oc create -f nginx-service.yaml
~~~
apiVersion: v1
kind: Service
metadata:
  namespace: dataplane
  name: nginx-service
  labels:
    app: nginx
spec:
  ports:
  - port: 30210
    protocol: TCP
    name: tcp
  - port: 30211
    protocol: TCP
    name: tls
  selector:
    app: nginx
~~~
4. Create gateway in the appropriate dataplane namespace. Gateway definition is available in the repo
~~~
oc create -f gateway.yaml -n dataplane
~~~
5. Create the virtual service in the appropriate dataplane namespace. virtual Service definition is available in the repo
~~~
oc create -f virtualservice.yaml
~~~
6. Create the tomcat deployment in the appropriate namespace. It could on the default namespace. tomcat will be outside of the servicemesh.  
   > oc create -f tomcat-deployment.yaml
~~~
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deployment
  namespace: default
  labels:
    app: tomcat
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      containers:
      - name: tomcat
        image: quay.io/rhn_support_rrajaram/tomcat:latest
        ports:
        - containerPort: 3000
~~~
7. Create the associated service for tomcat 
   > oc create -f tomcat-service.yaml
~~~
apiVersion: v1
kind: Service
metadata:
  namespace: default
  name: tomcat-svc
  labels:
    app: tomcat
spec:
  ports:
  - port: 3000
    protocol: TCP
    name: tcp
  selector:
    app: tomcat

~~~
