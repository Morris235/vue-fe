# Vue + Spring Boot Kubernetes 배포 (Apple Silicon M1/M2)

## 📋 목차

1. [프로젝트 개요](#프로젝트-개요)
2. [Docker 이미지 빌드 및 Push](#docker-이미지-빌드-및-push)
   - [Vue](#vue)
   - [Spring Boot](#spring-boot)
3. [Kubernetes 리소스 생성](#kubernetes-리소스-생성)
   - [서비스 설정 (ClusterIP)](#서비스-설정-clusterip)
   - [Ingress 설정 (CORS + 최적화)](#ingress-설정-cors--최적화)
4. [배포 이미지 적용 (Rollout)](#배포-이미지-적용-rollout)
5. [접속 및 테스트](#접속-및-테스트)
6. [결과 정리](#결과-정리)

---

## 📌 프로젝트 개요

- 기능: Vue에서 숫자 입력 → Spring API로 덧셈 요청 → 결과 반환
- 환경: macOS Apple Silicon (M1/M2)
- 배포: Docker Hub + Kubernetes + NGINX Ingress + Rollout
- 도메인: `vue.localtest.me`, `api.localtest.me` (`*.localtest.me`는 127.0.0.1로 자동 연결됨)

---

## 🐳 Docker 이미지 빌드 및 Push

### ✅ Vue

**1. `Dockerfile`**

```dockerfile
FROM node:20-bullseye AS build-stage  # 베이스 이미지 설정
WORKDIR /app  # 작업 디렉토리 설정
COPY package*.json ./  # 파일 복사
RUN npm install  # 의존성 설치
COPY . .  # 파일 복사
RUN npm run build  # Vue 앱 빌드

FROM nginx:1.25.3-bookworm AS production-stage  # 베이스 이미지 설정
COPY --from=build-stage /app/dist /usr/share/nginx/html  # 파일 복사
COPY ./nginx.conf /etc/nginx/conf.d/default.conf  # 파일 복사
EXPOSE 80  # 컨테이너 노출 포트 설정
CMD ["nginx", "-g", "daemon off;"]  # 컨테이너 실행 명령
```

**2. `nginx.conf` (최적화 적용)**

```nginx
server {
    listen 80;  # 80번 포트에서 요청 수신
    server_name localhost;  # 서버 도메인 이름 설정

    root /usr/share/nginx/html;  # 정적 파일 루트 디렉토리 설정
    index index.html;

    location ~* \.(?:ico|css|js|gif|jpe?g|png|woff2?|eot|ttf|svg)$ {
        access_log off;
        expires 6M;  # 브라우저 캐시 유효기간 설정
        add_header Cache-Control "public, max-age=15552000, immutable";  # Cache-Control 헤더 추가
    }

    location / {
        try_files $uri $uri/ /index.html;  # SPA 라우팅 처리
    }
}
```

**3. 빌드 및 푸시**

```bash
cd vue-fe
docker build -t morris235/vue-fe:latest .  # 도커 이미지 빌드
docker push morris235/vue-fe:latest  # 도커 이미지 푸시
```

---

### ✅ Spring Boot

**1. `Dockerfile`**

```dockerfile
FROM eclipse-temurin:17-jre-bookworm  # 베이스 이미지 설정
WORKDIR /app  # 작업 디렉토리 설정
COPY build/libs/*.jar app.jar  # 파일 복사
ENTRYPOINT ["java", "-jar", "app.jar"]  # 컨테이너 실행 명령
```

**2. 빌드 및 푸시**

```bash
cd spring-be
./gradlew build
docker build -t morris235/spring-be:latest .  # 도커 이미지 빌드
docker push morris235/spring-be:latest  # 도커 이미지 푸시
```

---

## ☸️ Kubernetes 리소스 생성

### ✅ 서비스 설정 (ClusterIP)

**spring-be-service.yaml**
```yaml
apiVersion: v1  # 리소스 API 버전 지정
kind: Service  # 리소스 종류 (Service, Ingress 등)
metadata:  # 메타데이터 정의 시작
  name: spring-be-service  # 리소스 이름 설정
spec:  # 실제 설정 시작
  selector:  # 연결할 Pod의 라벨 선택
    app: spring-be
  ports:  # 포트 정의 블록 시작
    - port: 8080  # 서비스가 노출할 포트
      targetPort: 8080  # 실제 컨테이너 내부 포트
```

**vue-fe-service.yaml**
```yaml
apiVersion: v1  # 리소스 API 버전 지정
kind: Service  # 리소스 종류 (Service, Ingress 등)
metadata:  # 메타데이터 정의 시작
  name: vue-fe-service  # 리소스 이름 설정
spec:  # 실제 설정 시작
  selector:  # 연결할 Pod의 라벨 선택
    app: vue-fe
  ports:  # 포트 정의 블록 시작
    - port: 80  # 서비스가 노출할 포트
      targetPort: 80  # 실제 컨테이너 내부 포트
```

**적용:**
```bash
kubectl apply -f spring-be-service.yaml  # 쿠버네티스 리소스 적용
kubectl apply -f vue-fe-service.yaml  # 쿠버네티스 리소스 적용
```

---

### ✅ Ingress 설정 (CORS + 최적화)

**app-ingress.yaml**
```yaml
apiVersion: networking.k8s.io/v1  # 리소스 API 버전 지정
kind: Ingress  # 리소스 종류 (Service, Ingress 등)
metadata:  # 메타데이터 정의 시작
  name: app-ingress  # 리소스 이름 설정
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "http://vue.localtest.me"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "Content-Type"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
spec:  # 실제 설정 시작
  ingressClassName: nginx
  rules:  # 인그레스 도메인/경로 라우팅 규칙
    - host: vue.localtest.me
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: vue-fe-service  # 리소스 이름 설정
                port:  # 서비스가 노출할 포트
                  number: 80
    - host: api.localtest.me
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: spring-be-service  # 리소스 이름 설정
                port:  # 서비스가 노출할 포트
                  number: 8080
```

**적용:**
```bash
kubectl apply -f app-ingress.yaml  # 쿠버네티스 리소스 적용
```

---

## 🔄 배포 이미지 적용 (Rollout)

```bash
kubectl rollout restart deployment vue-fe-deployment  # 디플로이먼트 재시작
kubectl rollout restart deployment spring-be-deployment  # 디플로이먼트 재시작
```

---

## 🌐 접속 및 테스트

> 반드시 `Ingress Controller`가 정상 설치되어 있어야 함

### ✅ Ingress 주소 확인
```bash
kubectl get ingress
```

### ✅ 브라우저 접속

- [http://vue.localtest.me](http://vue.localtest.me)
- [http://api.localtest.me/plus](http://api.localtest.me/plus)

### ✅ API 직접 테스트

```bash
curl -X POST http://api.localtest.me/plus \  # API 호출 테스트
  -H "Content-Type: application/json" \
  -d '{"num1":1,"num2":2}'
```

---

## ✅ 결과 정리

| 항목 | 처리 방식 |
|------|-----------|
| Docker 빌드 | `Dockerfile`로 multi-stage 빌드 |
| Vue 정적 리소스 최적화 | Nginx 설정 (`expires`, `cache-control`, `gzip`) |
| API 요청 속도 개선 | Ingress proxy-buffering 비활성화 |
| CORS 문제 | Ingress에서 해결 (`nginx.ingress.kubernetes.io/enable-cors`) |
| 배포 방식 | Docker Hub → Kubernetes Rollout |
| 도메인 테스트 | `vue.localtest.me`, `api.localtest.me` 사용 (127.0.0.1 자동 연결) |

---

> 이 문서는 Apple Silicon 환경에서 Kubernetes 기반으로 Vue + Spring Boot를 배포하려는 개발자를 위한 최적화된 실전 배포 가이드입니다.