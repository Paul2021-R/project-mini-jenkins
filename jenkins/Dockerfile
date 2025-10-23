# Jenkins 공식 이미지 활용 
FROM jenkins/jenkins:lts-jdk21

# root 권한으로 플러그인 설치 수행
USER root

# plugin.txt 파일을 이미지 안에 복사
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt

# plugin 설치 
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

# ssh-client 패키지 설치 
RUN apt-get update && apt-get install -y openssh-client

# casc.yaml 파일을 참조(ref) 디렉토리로 복사
COPY casc.yaml /usr/share/jenkins/ref/casc.yaml

# jenkins 유저로 전환하여 보안 유지
USER jenkins