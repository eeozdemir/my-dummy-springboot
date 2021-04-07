# Proje : Dummy Spring Boot CI/CD Pipeline

## Açıklama

Dockerfile ile çalışır halde dockerize edilen uygulamanın build ve deployment aşamalarında bir CI/CD tool ile pipeline hazırlanacaktır. Build olan proje hedef sunucuya veya kubernetes ortamına CI/CD tool’u üzerinden deploy edilerek dışarıya servis edilecektir.

## DevOps Pipelines

## Part 1

https://gitlab.com/bcfmkubilay/dummy-spring-boot adresinde bulunan repoyu localime clone ettikten sonra localimde oluşturduğum dosyamı github üzerinde yeni repoma push ederek kendi repomu üzerinden calışmalarıma devam ettim.


## Part 2

CloudFormation Template kullanarak otomatik olarak Jenkins server oluşturdum. Template dosyası github adresimde mevcuttur. Bu Template dosyası ile EC2 içerisine Git, Docker, Jenkins, Java ve AWS CLI kurulumlarını yaptım.


## Part 3 

Kurmuş olduğum EC2'nun PublicIP'sine 8080 ile bağlanarak Jenkins konsola giriş yaptım. Jenkins ana sayfası üzerinde Manage Jenkins>>Manage Plugins>>Available tab içerisinde Github Integration, Docker, Docker Pipeline ve Jacoco kurulumlarını yaptım.


## Part 4

Jenkins konsolu üzerinde New Item>>Freestyle Project seçtim ve onayladım. Hemen sonra açılan sayfada:

  Github Project
    Project url
      https://github.com/mrozdemir4/my-dummy-springboot

  Source Code Management
    .Git
      Repository url
        https://github.com/mrozdemir4/my-dummy-springboot.git

  Branch
    *.main

  Build Environment
    Add timestamps to the Console Output (click)
  

  Build
    Execute Shell (click)
      Command:
        ``` bash
        $ docker run --rm -v $HOME/.m2 -v $WORKSPACE:/app -w /app maven:3.6.3-openjdk-8 mvn clean package
        ```

Komutlar sonrasında Save edip Build Now yaptığımızda programımız çalışmayacak hata verecektir. Hatamızı düzeltmek için /src/test/java/com/gazgeek/helloworld/ klasörü içerisinde 31. satırdaki "Hello from bestcloudforme!" yazısını "Hello from BestCloudForMe!" şeklinde güncelledim. Düzeltme sonrasında Github hesabıma gönderdim ve tekrar Jenkıns Server'daki Item'ı tekrar Build ettim.  


## Part 5

Dockerfile oluşturdum ve Github'a push ettim.

``` Dockerfile
FROM openjdk:8-jre
ADD ./target/*.jar /app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```


## Part 6

Jenkins'de oluşturduğum projemi,

``` bash
$ docker build -t <imagename> .
```
komutu ile Configure ederek Docker Image'imi build ettim.


Build ettiğim image i Dockerfile ile oluşturduğum target dosyası içinde oluştuğunu kontrol ettim.

``` bash
$ docker image ls
```

## Part 7

Dockerfile ve Docker Image oluşturduktan sonra yine Jenkins'de oluşturduğum projemi, 

``` bash
$ docker run -d -p 80:8080 <imagename>
```
komutu ile Configure ederek browser üzerinde <EC2-PublicIP>:8080 ile bağlandim "Hello from BestCloudForMe!" sayfasını görüntüledim.

## Part 8 

DockerHub üzerinden "Hello from BestCloudForMe!" sayfasını görüntülemek için Jenkins'de oluşturduğum projemi,

``` bash
$ docker login -u <username> -p <password>
$ docker push <username>/<imagename>
```
komutları ile tekrar Configure ederek browser üzerinde <EC2-PublicIP>:80 ile "Hello from BestCloudForMe!" sayfasını görüntüledim.

DockerHub'a login olduğum için de "docker run -d -p 80:8080 <imagename>" komutunu,

``` bash
$ docker run -d -p 80:8080 <dockerhub-username>/<imagename>
```
şeklinde yeniden güncelledim.


## Part 9

### Install kubectl

Kubernetes üzerinden orkestrasyon ortamına deploy etmek için Jenkins Server'a Amazon EKS kubectl i download ettim.

```bash
$ curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.17.9/2020-08-04/bin/linux/amd64/kubectl
```

Kubectl yürütme izinlerini uyguladım.

```bash
$ chmod +x ./kubectl
```

Kubectl dosyasını /usr/local/bin klasörüne taşıdım.

```bash
$ sudo mv ./kubectl /usr/local/bin
```

Kubectl'i kurduktan sonra local klasörü içerisinde sürümünü doğruladım.

```bash
$ cd /usr/local
$ kubectl version --short --client
```

### Install eksctl

eksctl'in son sürümünü yükledim.

```bash
$ curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```

eksctl'i /usr/local/bin klasörü içerisine taşıdım.

```bash
$ sudo mv /tmp/eksctl /usr/local/bin
```

Aşağıdaki komutla kurulumun başarılı olup olmadığını test ettim.

```bash
$ eksctl version
```

## Part 10

EKS üzerinde Kubernetes Cluster oluşturmak için aşagıdaki komutları girdim.


AWS kimlik bilgilerini yapılandırdım.

```bash
$ aws configure
```

Eksctl aracılığıyla bir EKS kümesi oluşturdum.

```bash
$ eksctl create cluster --region us-east-1 --node-type t2.medium --nodes 1 --nodes-min 1 --nodes-max 2 --node-volume-size 8 --name mycluster
```

## Part 11

Kubernetes'te dağıtım güncellemesi ve geri alma(image in kubernetes ekosisteminde nasıl dağıtılacağı) özellikleri için springboot-deployment.yaml dosyası oluşturdum ve aşağıdaki metni girdim.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-springboot
  labels:
    app: my-springboot
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-springboot
  template:
    metadata:
      labels:
        app: my-springboot
    spec:
      containers:
      - name: my-springboot
        image: ozdemire/springboot
        ports:
        - containerPort: 8080
```


Podları dış dünyaya bağlamak(podlara gelen isteklerin karşılanması ve yönlendirilmesi) için springboot-service.yaml dosyasını oluşturdum ve aşağıdaki metni girdim.

```yaml
apiVersion: v1
kind: Service   
metadata:
  name: my-springboot-svc
  labels:
    app: my-springboot-svc
spec:
  type: LoadBalancer 
  ports:
  - port: 8080  
    targetPort: 8080
  selector:
    app: my-springboot
```

Oluşturduğum bu yeni dosyaları github repoma gönderdim.

```bash
$ git add .
$ git commit -m "added yaml folders"
$ git push
```

Lokalde oluşturup github repoma gönderdiğim yaml dosyalarını Jenkins Server'a çektim.

```yaml
$ git clone https://github.com/mrozdemir4/my-dummy-springboot
```


## Part 12

Kurmuş olduğum EKS cluster içerisinde bulunan nodeları listeledim.

```bash
kubectl get node
```

my-dummy-springboot klasörü içerisinde oluşturduğum springboot-deployment ve springboot-service hizmetlerini uygulama komutlarıyla yapılandırdım. 

```bash
kubectl apply -f springboot-deployment.yaml
kubectl apply -f springboot-service.yaml
```

Oluşturduğum konteynırların hazır olup olmadığını kontrol ettim.

```bash
kubectl get deploy
```

Oluşan podları listeledim.

```bash
kubectl get pod
```

Servisleri listemek ve EXTERNAL-IP ye ulaştım.

```bash
kubectl get service
```

EXTERNAL-IP:8080 ile "Hello from BestCloudForMe!" sayfasını görüntüledim.


## Part 13

Localde oluşturduğum dummy-spring-boot dosyası içerisine ingress-service.yaml dosyası oluşturdum ve aşağıdaki metni girdim.


```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-service
  annotations:
    kubernetes.io/ingress.class: 'nginx'
    nginx.ingress.kubernetes.io/use-regex: 'true'
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
    - http:
        paths:
          - path: /?(.*)
            backend:
              serviceName: my-springboot-svc
              servicePort: 8080
```

ingress-service.yaml dosyasını github repoma push ettim.

```bash
$ git add .
$ git commit -m "ingress-service"
$ git push
```

ingress-service.yaml dosyasını EC2 Jenkins Server da git pull yaparak github repomdan çektim.

my-dummy-springboot klasörü içerisinde oluşturduğum ingress-service hizmetini uygulama komutlarıyla yapılandırdım.

```bash
kubectl apply -f ingress-service.yaml
```

ingress servisinin kontrolünü sağlamak ve address bilgisine ulaşmak için aşağıdaki komutu Jenkins Server da çalıştırdım.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.35.0/deploy/static/provider/aws/deploy.yaml
```

ingress servisini listeledim ve adres bilgisine "/" ekleyerek browser da "Hello from BestCloudForMe!" sayfasını görüntüledim.

```bash
kubectl get ingress
```

Cluster'ı görüntülemek ve çalışmayı tamamladıktan sonra silmek için aşagıdaki komutları girdim.

```bash
$ eksctl get cluster --region us-east-1
$ eksctl delete cluster mycluster --region us-east-1
```
