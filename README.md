## 1. 목적

- DB 연결이 필요 없는 단순 Spring Boot 웹앱을 Docker Hub에 이미지로 올리고 Kubernetes에서 배포
- Deployment, Service, Ingress를 이용하여 **3개 Pod로 실행 및 외부 통신 확인**
- 윈도우에서 브라우저 또는 Postman으로 접근 가능하게 설정
- Ingress 충돌 방지 및 환경변수 설정 실습 포함

---

## 2. 관련 개념

## 1️⃣ Docker 관련 개념

1. **이미지(Image) vs 컨테이너(Container)**
    - 이미지: 실행 가능한 애플리케이션과 환경 설정이 포함된 패키지
    - 컨테이너: 이미지를 실제로 실행한 상태
    - `docker run` → 컨테이너 생성 및 실행
2. **Dockerfile**
    - 이미지를 만들기 위한 선언적 파일
    - `FROM`, `COPY`, `WORKDIR`, `EXPOSE`, `ENTRYPOINT` 같은 명령어 사용
3. **Docker Hub**
    - 이미지 저장소
    - `docker push` → 이미지 업로드, `docker pull` → 이미지 다운로드
4. **컨테이너 포트 매핑**
    - `docker run -p 8080:8080`: 호스트 8080 → 컨테이너 8080

---

## 2️⃣ Kubernetes 핵심 개념

1. **Pod**
    - Kubernetes에서 가장 작은 배포 단위
    - 하나 이상의 컨테이너를 포함
    - 단일 Pod 내 컨테이너는 같은 네트워크와 스토리지를 공유
2. **Deployment**
    - Pod를 선언형으로 관리
    - ReplicaSet을 생성하여 지정한 개수(replica)만큼 Pod 유지
    - 롤링 업데이트, 롤백 가능
3. **Service**
    - Pod의 네트워크 접근을 안정적으로 제공
    - 타입:
        - ClusterIP: 클러스터 내부 접근 전용
        - NodePort: 외부에서 IP:Port로 접근 가능
        - LoadBalancer: 클라우드 환경에서 외부 LoadBalancer 제공
    - Pod Selector로 대상 Pod 지정
4. **Ingress**
    - HTTP/HTTPS 요청을 라우팅
    - host + path 기반으로 Service와 연결
    - 여러 Ingress가 겹치면 충돌 발생
    - Nginx Ingress Controller가 필요
5. **NodePort vs Ingress**
    - NodePort: 단순 포트 매핑, 테스트용
    - Ingress: 도메인 기반 접근, 외부 통신 시 권장

---

## 3️⃣ Kubernetes 관리 명령어

| 작업 | 명령어 |
| --- | --- |
| 리소스 생성 | `kubectl apply -f <file.yaml>` |
| Pod 상태 확인 | `kubectl get pods` |
| 로그 확인 | `kubectl logs <pod-name>` |
| 스케일 조정 | `kubectl scale deployment <name> --replicas=<n>` |
| 리소스 삭제 | `kubectl delete -f <file.yaml>` |
| Service 확인 | `kubectl get svc` |
| Ingress 확인 | `kubectl get ingress` |

---

## 4️⃣ 환경변수 사용 개념

- Pod 안 컨테이너에서 실행 환경을 설정하는 방법
- YAML에서 `env:`를 사용하여 key/value 지정
- 예: `SERVER_PORT`, `APP_ENV`
- 코드에서 `System.getenv("APP_ENV")` 등으로 접근 가능

---

## 5️⃣ 실습 중 유의할 개념

1. **Ingress 충돌 방지**
    - 호스트(host) + 경로(path) 중복 X
    - 기존 Ingress가 있는 경우 삭제하거나 새 이름/호스트 사용
2. **Pod CrashLoopBackOff**
    - 컨테이너가 바로 죽는 상태
    - 로그 확인 필요 (`kubectl logs` / `-previous`)
3. **NodePort 포트 충돌**
    - NodePort는 클러스터 전체에서 중복 X
    - 30000~32767 범위 내 사용
4. **Docker Hub 연동**
    - 반드시 `docker build → docker push → kubectl Deployment image` 순서

---

## 3. Docker 이미지 빌드 및 푸시

### 3.1 Dockerfile

```docker
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
```

### 3.2 이미지 빌드

```bash
docker build -t <DOCKER_HUB_ID>/springapp:1.0 .
```

### 3.3 Docker Hub 로그인 및 푸시

```bash
docker login
docker push <DOCKER_HUB_ID>/springapp:1.0
```

---

## 4. Kubernetes 선언형 YAML (환경변수 + Ingress 충돌 방지)

### 4.1 Deployment (`deploy.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springapp-deployment
  labels:
    app: springapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: springapp
  template:
    metadata:
      labels:
        app: springapp
    spec:
      containers:
        - name: springapp
          image: sumin07/springapp:1.0
          ports:
            - containerPort: 8085
          env:
            - name: SERVER_PORT
              value: "8082"

---
apiVersion: v1
kind: Service
metadata:
  name: springapp-service
spec:
  selector:
    app: springapp
  ports:
    - port: 8085       
      targetPort: 8085 
      nodePort: 30090
  type: NodePort
```

---

### 4.2 Ingress (`ingress.yaml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: springapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: springapp.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: springapp-service
                port:
                  number: 8085
```

> 🔹 팁: Ingress를 여러 개 만들 때는 metadata.name과 host를 항상 새로 지정해서 충돌을 방지하세요.
> 

---

## 5. Kubernetes 리소스 생성 / 관리 / 삭제

```bash
# 생성
kubectl apply -f deploy.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml

# Pod 상태 확인
kubectl get pods
kubectl logs <pod-name>

# Pod 수 조절
kubectl scale deployment springapp-deployment --replicas=5

# 삭제
kubectl delete -f deploy.yaml
kubectl delete -f service.yaml
kubectl delete -f ingress.yaml
```

---

## 6. 외부 통신 확인

### 6.1 NodePort 접속

브라우저 / Postman:

```
http://<클러스터노드IP>:30090
```

### 6.2 Ingress 접속

1. 로컬 PC `/etc/hosts` 수정:

```
<클러스터노드IP> springapp01.loca
```

1. 브라우저 / Postman:
```
http://springapp01.local
```
