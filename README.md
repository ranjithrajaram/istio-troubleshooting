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
kubectl create secret tls nginx-server-certs --key nginx.example.com.key --cert nginx.example.com.crt
~~~
2. Deploy the nginx application in the appropriate namespace. Two annotations are being added to the deployment
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
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: quay.io/rhn_support_rrajaram/nginx:latest
        ports:
        - containerPort: 30210
~~~
