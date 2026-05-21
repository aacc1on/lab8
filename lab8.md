# Լաբորատոր Աշխատանք 8
## GitHub Actions-ում CI/CD Pipeline

---

**Առարկա:** Կոնտեյներային տեխնոլոգիաներ և միկրոծառայություններ  
**Թեմա:** CI/CD Pipeline, GitHub Actions, Docker image build և push, Ingress, TLS

---

## Նպատակ

Ծանոթանալ CI/CD (Continuous Integration / Continuous Delivery) հասկացությանը։ Ստեղծել GitHub Actions workflow, որը ավտոմատ կերպով build և push կանի Docker image-ը Docker Hub-ին ամեն անգամ, երբ կոդ push կատարվի `main` branch-ում։ Լրացուցիչ — կազմաձևել Kubernetes Ingress SSL/TLS-ով։

---

## Տեսական Հիմք

| Հասկացություն | Բացատրություն |
|---|---|
| **CI (Continuous Integration)** | Կոդի ավտոմատ build և test ամեն commit-ից հետո |
| **CD (Continuous Delivery)** | Ավտոմատ deploy կամ packaging արտադրողական միջավայրի համար |
| **GitHub Actions** | GitHub-ի ներկառուցված CI/CD գործիք, YAML-ով կոնֆիգուրացված |
| **Workflow** | Ավտոմատ գործընթացների հավաքածու, `.github/workflows/` թղթապանակում |
| **Job** | Workflow-ի ներսում կատարվող առաջադրանքների խումբ |
| **Step** | Job-ի ներսում մեկ հրաման կամ action |
| **Ingress** | Kubernetes-ի HTTP/HTTPS մուտքի կետ կլաստեր |
| **TLS Secret** | Kubernetes-ի Secret, որը պարունակում է SSL վկայական և բանալի |

---

## Նախապայմաններ

- GitHub հաշիվ և repository
- Docker Hub հաշիվ
- Տեղադրված Docker (տեղային ստուգման համար)
- Տեղադրված և գործարկված Minikube կլաստեր (Ingress մասի համար)

---

## Մաս 1 — GitHub Actions CI/CD Pipeline

---

## Քայլ 1 — Flask հավելվածի Ստեղծում

Ստեղծել նոր GitHub repository, ապա ավելացնել հետևյալ ֆայլերը.

**`app.py`**

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello from Docker CI/CD!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

**`requirements.txt`**

```
flask
```

**`Dockerfile`**

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

---

## Քայլ 2 — Docker Hub Credentials-ի Ավելացում GitHub Secrets-ում

GitHub-ի repository-ում.

1. Բացել `Settings` → `Secrets and variables` → `Actions`
2. Սեղմել `New repository secret`
3. Ավելացնել հետևյալ երկու secret-ը.

| Secret անուն | Արժեք |
|---|---|
| `DOCKER_USERNAME` | Docker Hub-ի username |
| `DOCKER_PASSWORD` | Docker Hub-ի password կամ access token |

---

### Screenshot 1
> GitHub repository-ի Settings → Secrets բաժինը — երկու secret-ները ավելացված։

&nbsp;

---

## Քայլ 3 — GitHub Actions Workflow-ի Ստեղծում

Repository-ում ստեղծել թղթապանակ և ֆայլ.

```
.github/
  workflows/
    docker.yml
```

**`.github/workflows/docker.yml`**

```yaml
name: Docker CI

on:
  push:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build Docker image
        run: docker build -t ${{ secrets.DOCKER_USERNAME }}/myapp:latest .

      - name: Push Docker image
        run: docker push ${{ secrets.DOCKER_USERNAME }}/myapp:latest
```

---

## Քայլ 4 — Workflow-ի Գործարկում

Ֆայլերը commit և push անել `main` branch-ում.

```bash
git add .
git commit -m "Add CI/CD pipeline"
git push origin main
```

GitHub-ում բացել `Actions` tab — workflow-ը ավտոմատ կսկսի։

Pipeline-ի հոսք.

```
Push to main
     |
     v
Checkout code
     |
     v
Login to Docker Hub
     |
     v
docker build -t username/myapp:latest .
     |
     v
docker push username/myapp:latest
```

---

### Screenshot 2
> GitHub Actions tab — workflow-ը հաջողությամբ ավարտված (կանաչ checkmark)։

&nbsp;

---

## Քայլ 5 — Docker Hub-ի Ստուգում

Մուտք գործել [hub.docker.com](https://hub.docker.com) և ստուգել, որ `myapp` image-ը հայտնվել է։

Տեղային ստուգում.

```bash
docker pull your-username/myapp:latest
docker run -p 5000:5000 your-username/myapp:latest
```

Բրաուզերում բացել `http://localhost:5000` — պետք է երևա `Hello from Docker CI/CD!`

---

### Screenshot 3
> Docker Hub-ում myapp image-ը — վերջին push-ի ժամանակ և tag-ը տեսանելի։

&nbsp;

---

## Մաս 2 — Kubernetes Ingress և SSL/TLS

---

## Քայլ 6 — Ingress Controller-ի Միացում (Minikube)

```bash
minikube addons enable ingress
```

Ստուգել, որ ingress controller-ը գործարկված է.

```bash
kubectl get pods -n ingress-nginx
```

Ակնկալվող արդյունք — `ingress-nginx-controller` Pod-ը Running կարգավիճակում։

---

### Screenshot 4
> `kubectl get pods -n ingress-nginx` — controller-ը Running կարգավիճակում։

&nbsp;

---

## Քայլ 7 — Self-Signed TLS Վկայականի Ստեղծում

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout cert.key \
  -out cert.crt \
  -subj "/CN=myapp.local/O=myapp"
```

Ստեղծել Kubernetes TLS Secret.

```bash
kubectl create secret tls my-tls \
  --cert=cert.crt \
  --key=cert.key
```

Ստուգել Secret-ի ստեղծումը.

```bash
kubectl get secrets
```

---

### Screenshot 5
> `kubectl get secrets` — my-tls secret-ը kubernetes.io/tls տիպով ստեղծված։

&nbsp;

---

## Քայլ 8 — Nginx Deployment և Service (Ingress-ի Հիմք)

Ստեղծել `nginx-deploy.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-deployment
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

Կիրառել.

```bash
kubectl apply -f nginx-deploy.yaml
```

---

## Քայլ 9 — Ingress Ռեսուրսի Ստեղծում TLS-ով

Ստեղծել `ingress.yaml`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - myapp.local
    secretName: my-tls
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-deployment
            port:
              number: 80
```

Կիրառել.

```bash
kubectl apply -f ingress.yaml
```

Ստուգել Ingress-ը.

```bash
kubectl get ingress
```

---

### Screenshot 6
> `kubectl get ingress` — my-ingress ստեղծված, ADDRESS սյունակում IP-ն երևում է։

&nbsp;

---

## Քայլ 10 — Տեղային DNS Կարգաբերում և Ստուգում

Ստանալ Minikube IP-ն.

```bash
minikube ip
```

Ավելացնել `/etc/hosts` ֆայլում (Linux/Mac).

```bash
sudo echo "$(minikube ip) myapp.local" >> /etc/hosts
```

Windows-ում բացել `C:\Windows\System32\drivers\etc\hosts` administrator-ով և ավելացնել.

```
<minikube-ip>   myapp.local
```

Ստուգել բրաուզերում.

```
https://myapp.local
```

Քանի որ self-signed վկայական է, բրաուզերը կցուցաբերի անվտանգության զգուշացում — ընդունել և շարունակել։

---

### Screenshot 7
> Բրաուզերում `https://myapp.local` — nginx-ի default էջը HTTPS-ով բեռնված։

&nbsp;

---

## Ամփոփ — Ողջ Pipeline-ի Հոսք

```
Developer pushes code to GitHub (main branch)
                |
                v
        GitHub Actions trigger
                |
                v
        docker build -t myapp .
                |
                v
        docker push → Docker Hub
                |
                v
    (Manual) kubectl apply → Kubernetes
                |
                v
        Ingress (HTTPS) → Service → Pod
```

---

## Ստուգաթերթ

| # | Կատարված առաջադրանք | Կարգավիճակ |
|---|---|---|
| 1 | Flask հավելված և Dockerfile ստեղծված | |
| 2 | Docker Hub secrets ավելացված GitHub-ում | |
| 3 | `.github/workflows/docker.yml` ստեղծված | |
| 4 | Push կատարվել է, workflow հաջողությամբ ավարտվել | |
| 5 | Docker Hub-ում image-ը հայտնվել է | |
| 6 | Ingress controller-ը Minikube-ում ակտիվ | |
| 7 | TLS Secret ստեղծված openssl-ով | |
| 8 | Ingress ռեսուրս ստեղծված TLS կոնֆիգուրացիայով | |
| 9 | `https://myapp.local` հասանելի է բրաուզերից | |

---

## Եզրակացություն

Լաբորատոր աշխատանքի ընթացքում.

- Ստեղծվեց **GitHub Actions workflow**, որն ավտոմատ build և push է անում Docker image-ը
- Կոնֆիգուրացվեց **Docker Hub** authentication GitHub Secrets-ի միջոցով
- Ստեղծվեց **TLS Secret** self-signed վկայականով
- Կոնֆիգուրացվեց **Kubernetes Ingress** HTTPS աջակցությամբ
- Հաստատվեց, որ կոդի ամեն push-ը ավտոմատ trigger է անում pipeline-ը

CI/CD pipeline-ն ապահովում է, որ կոդի փոփոխությունները **արագ, ավտոմատ և կրկնվող** կերպով հասնեն արտադրողական միջավայր, նվազեցնելով մարդկային սխալների հավանականությունը։

---

*Կոնտեյներային տեխնոլոգիաներ և միկրոծառայություններ | Լաբ 8*