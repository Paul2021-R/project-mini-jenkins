# Jenkins 공식 이미지 활용 
FROM jenkins/jenkins:lts-jdk21

# root 권한으로 플러그인 설치 수행
USER root

# --- Docker CLI 설치 시작 ---
# 1. 패키지 목록 업데이트 및 필수 패키지 설치
RUN apt-get update && apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    openssh-client  

# 2. Docker 공식 GPG 키 추가
RUN mkdir -p /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 3. Docker APT 저장소 설정
RUN echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

# 4. 패키지 목록 다시 업데이트 및 Docker CLI 설치 (서버 데몬 제외)
RUN apt-get update && apt-get install -y docker-ce-cli

# 5. (선택 사항) apt 캐시 정리하여 이미지 크기 줄이기
RUN rm -rf /var/lib/apt/lists/*

# DooD 구조를 위함. Docker 그룹에 jenkins 유저 추가
# TODO: 시스템마다 다르므로 하드코딩 수정할 것!
ARG DOCKER_GID=984
RUN groupadd -g ${DOCKER_GID} docker
RUN usermod -aG docker jenkins

# plugin.txt 파일을 이미지 안에 복사
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt

# plugin 설치 
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

# casc.yaml 파일을 참조(ref) 디렉토리로 복사
COPY casc.yaml /usr/share/jenkins/ref/casc.yaml

# jenkins 유저로 전환하여 보안 유지
USER jenkins