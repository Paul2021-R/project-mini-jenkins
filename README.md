# project-mini-jenkins

# My Jenkins Server Project

이 프로젝트는 Docker와 JCasC(Jenkins Configuration as Code)를 사용하여 Jenkins 서버를 코드 기반으로 관리합니다.

## 🚀 시작하기

1.  **Docker Desktop**이 설치되어 있는지 확인하세요.
2.  아래 명령어를 실행하여 Jenkins 서버를 시작합니다.

    ```bash
    docker-compose up --build -d
    ```

3.  웹 브라우저에서 `http://localhost:8080`으로 접속하세요.

## 📁 프로젝트 구조

-   `docker-compose.yml`: Docker 컨테이너 실행 환경을 정의합니다.
-   `jenkins/Dockerfile`: Jenkins 커스텀 이미지를 빌드합니다.
-   `jenkins/casc.yaml`: Jenkins의 시스템 설정을 코드(YAML)로 관리합니다.
-   `jenkins/plugins.txt`: 설치할 플러그인 목록을 관리합니다. **(For Machine)**

---

## 🔌 플러그인 관리

Jenkins에 설치된 플러그인 목록과 그 용도는 아래와 같습니다. 플러그인을 추가/삭제할 경우, 반드시 아래 **테이블과 `jenkins/plugins.txt` 파일을 함께 수정**해야 합니다. **(For Human)**

| 플러그인 ID | 핵심 역할 | 비고 |
| :--- | :--- | :--- |
| `configuration-as-code` | Jenkins 설정을 YAML 파일로 관리(JCasC) | 필수 플러그인 |
| `git` | Git 저장소와 연동하여 소스 코드 가져오기 | |
| `workflow-aggregator` | `Jenkinsfile`을 사용하는 파이프라인 잡 기능 제공 | |
| `blueocean` | 파이프라인 실행 과정을 시각적으로 보여주는 UI | |

---

## 📝 변경 이력

-  

## Error handling 

### 커멘드 에러

`RUN jenkins-plugin-cli --file /usr/share/jenkins/ref/plugins.txt`
-> 
`RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt` 

- 최초 AI 가 알려줌. 근데 이자식이 옛날 커맨드로...

### 컨테이너 종료됨. 로그 확인하기

- 오타 문제 해결 완료 

---

## 향후 할 일 

-----

### Next.js 무중단 배포 CI/CD 파이프라인 구축 가이드

이 문서는 운영 중인 Jenkins 서버를 활용하여 Next.js 애플리케이션의 빌드, 테스트, 배포 전 과정을 자동화하는 CI/CD 파이프라인 구축 방법을 기술한다. 배포 전략은 Nginx 리버스 프록시를 이용한 Blue/Green 배포 방식을 따른다.

-----

### 1. 프로젝트 레포지토리 구조

CI/CD 파이프라인을 적용할 Next.js 프로젝트는 다음과 같은 구조를 가진다. 이 구조는 애플리케이션 코드, 런타임 환경 설정, 그리고 배포 파이프라인 정의를 명확히 분리한다.

```
/my-nextjs-project/
├── nextjs-app/             # Next.js 애플리케이션 소스 코드
│   ├── ... (pages, public 등)
│   └── Dockerfile          # Next.js 앱 빌드 및 실행용
│
├── nginx/                  # Nginx 설정
│   └── nginx.conf.template # 배포 시 동적으로 생성될 Nginx 설정 템플릿
│
├── docker-compose.yml      # 💡 로컬 개발 및 테스트 환경 구성용
│
└── Jenkinsfile             # 👑 전체 CI/CD 프로세스를 정의하는 파이프라인 스크립트
```

  - **`docker-compose.yml`**: 이 파일은 Jenkins가 직접 사용하지 않는다. 개발자가 로컬 환경에서 Nginx와 Next.js를 함께 실행하여 실제 운영 환경을 모의 테스트하는 용도로만 사용된다.

-----

###  2. 핵심 컴포넌트 설정

#### `nextjs-app/Dockerfile`

Next.js 애플리케이션을 빌드하고, 프로덕션 환경에서 실행하기 위한 Docker 이미지 설계도이다. 멀티-스테이지 빌드를 사용하여 최종 이미지의 용량을 최적화한다.

```dockerfile
# 1. Builder Stage: 의존성 설치 및 소스 코드 빌드
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# 2. Runner Stage: 빌드 결과물만 포함하여 최종 이미지 생성
FROM node:18-alpine AS runner
WORKDIR /app
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

EXPOSE 3000
CMD ["node", "server.js"]
```

#### `nginx/nginx.conf.template`

Nginx 리버스 프록시의 설정 파일 템플릿이다. Jenkins는 배포 시점에 `TARGET_PORT` 변수를 실제 애플리케이션 컨테이너가 사용하는 포트 번호(예: `3001`, `3002`)로 교체한다.

```nginx
# nginx/nginx.conf.template
events {}
http {
    upstream nextjs_app {
        # Jenkins가 이 부분을 'server 127.0.0.1:3001;' 등으로 동적 생성한다.
        server 127.0.0.1:TARGET_PORT;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://nextjs_app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

-----

### 3. `Jenkinsfile`: CI/CD 파이프라인 로직

이 파일은 CI/CD 프로세스의 모든 단계를 정의하는 핵심 스크립트이다. Jenkins는 이 파일의 내용을 읽어 순차적으로 작업을 수행한다.

```groovy
// Jenkinsfile
pipeline {
    // Jenkins에 설정된 Docker Agent를 사용하도록 지정
    agent { label 'docker-node-with-host-docker' }

    environment {
        // GitHub PAT Credential ID
        GITHUB_CREDENTIALS_ID = 'github-pat-for-packages'
        // GitHub 사용자명 (또는 조직명)
        GITHUB_OWNER = 'your-github-username'
        // GitHub 레포지토리 이름 (동적으로 추출 가능)
        REPO_NAME = 'my-nextjs-repo'
    }

    stages {
        stage('Build & Push Image') {
            steps {
                script {
                    // Jenkins Credential 플러그인을 통해 PAT를 안전하게 불러옴
                    withCredentials([string(credentialsId: GITHUB_CREDENTIALS_ID, variable: 'GITHUB_PAT')]) {
                        // 빌드 번호를 포함한 유니크한 이미지 태그와 latest 태그를 함께 생성
                        def imageTag = "ghcr.io/${GITHUB_OWNER}/${REPO_NAME}:${env.BUILD_NUMBER}"
                        def latestTag = "ghcr.io/${GITHUB_OWNER}/${REPO_NAME}:latest"

                        // 1. Docker 이미지 빌드
                        sh "docker build -t ${imageTag} -t ${latestTag} ./nextjs-app"

                        // 2. GitHub Container Registry(ghcr.io)에 로그인
                        sh "echo ${GITHUB_PAT} | docker login ghcr.io -u ${GITHUB_OWNER} --password-stdin"
                        
                        // 3. 빌드된 이미지 푸시
                        sh "docker push ${imageTag}"
                        sh "docker push ${latestTag}"
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    // 4. Blue/Green 배포를 위한 포트 및 컨테이너 이름 결정
                    def targetPort, oldPort
                    if (env.BUILD_NUMBER.toInteger() % 2 == 0) {
                        targetPort = 3001
                        oldPort = 3002
                    } else {
                        targetPort = 3002
                        oldPort = 3001
                    }
                    
                    def appName = "my-nextjs-app"
                    def newContainerName = "${appName}-${targetPort}"
                    def oldContainerName = "${appName}-${oldPort}"
                    def imageToDeploy = "ghcr.io/${GITHUB_OWNER}/${REPO_NAME}:${env.BUILD_NUMBER}"

                    // 5. 신규 버전의 컨테이너를 새로운 포트로 실행 (전용 네트워크 사용)
                    sh "docker run -d --name ${newContainerName} -p ${targetPort}:3000 --network my-app-network --restart always ${imageToDeploy}"
                    
                    // 6. 새 컨테이너 Health Check (시작될 때까지 대기 후 확인)
                    sleep 15
                    sh "curl -f http://localhost:${targetPort} || (docker rm -f ${newContainerName} && exit 1)"

                    // 7. Nginx 템플릿을 기반으로 새로운 설정 파일 생성 및 적용
                    sh "sed 's/TARGET_PORT/${targetPort}/g' ./nginx/nginx.conf.template > ./nginx/generated_nginx.conf"
                    sh "docker cp ./nginx/generated_nginx.conf nginx-reverse-proxy:/etc/nginx/nginx.conf"

                    // 8. Nginx 리로드로 트래픽을 신규 컨테이너로 전환 (무중단)
                    sh "docker exec nginx-reverse-proxy nginx -s reload"

                    // 9. 이전 버전의 컨테이너를 안전하게 중지하고 제거
                    sh """
                    if [ \$(docker ps -q -f name=^/${oldContainerName}\$) ]; then
                        docker stop ${oldContainerName}
                        docker rm ${oldContainerName}
                    fi
                    """
                }
            }
        }
    }
    post {
        always {
            // 파이프라인 종료 시 항상 로그아웃하여 자격 증명 노출 방지
            sh 'docker logout ghcr.io'
        }
    }
}
```

-----

### 4. Jenkins 서버 요구사항

이 파이프라인이 정상적으로 동작하기 위해, 운영 중인 Jenkins 서버에는 다음과 같은 설정이 사전에 준비되어 있어야 한다.

  - **필수 플러그인**: `Docker`, `Docker Pipeline`, `Credentials Binding`.
  - **Credential 등록**: GitHub Container Registry에 이미지를 푸시하기 위한 Personal Access Token(PAT)이 Jenkins Credential에 `Secret text` 타입으로 등록되어 있어야 한다. (`ID`: `github-pat-for-packages`)
  - **Docker 에이전트 설정**: `Jenkinsfile`에서 사용할 `docker-node-with-host-docker` 라벨을 가진 Docker 에이전트 템플릿이 필요하다. 이 에이전트는 내부에 Docker CLI를 포함하고, 호스트의 Docker 데몬을 제어할 수 있도록 Docker 소켓(`/var/run/docker.sock`)이 마운트되어야 한다.
  - **공유 Docker 네트워크**: 배포 서버에 Nginx와 Next.js 컨테이너가 통신할 수 있는 `my-app-network`라는 이름의 Docker 네트워크가 생성되어 있어야 한다. (`docker network create my-app-network`)