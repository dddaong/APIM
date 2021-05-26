---
layout: post
title:  "NGINX Controller-Enabled NGINX Plus Container Image build"
date:   2021-05-26 18:45:00 +0900
categories: Kubernetes NGINXPlus Instance docker
---

Kubernetes / Openshift에서 사용할 NGINX Controller 연동 NGINX Plus Instance 의 이미지 빌드 과정을 설명한 문서입니다.



## NGINX Controller-Enabled NGINX Plus Container Image build

- NGINX Controller는 Control-plane으로서, NGINX+ Instance를 Dataplane을 담당하는 API Gateway로서 관리합니다.
- N+ API Gateway는 실제 트래픽을 처리하는 NGINX Plus와 
  이 NGINX Plus와 NGINX Controller와 연동을 위한 Controller-agent가 반드시 필요합니다.

  

- 초기의 API GW 이미지에서는 NGINX Plus를 설치하고, Controller-agent를 Entrypoint 스크립트에서 설치하도록 설계됐었습니다.

- 이 문서에서 소개하는 방법은 업데이트된 방식으로, Controller-agent를 이미지에 통합 설치하여 Image Build를 수행하고, Entrypoint 스크립트에서 사용 환경에 맞도록 Config Update를 수행하도록 되어 있습니다.  

  이 방법은 Kubernetes 환경의 API GW deploy 과정에서 Controller agent 설치 과정이 생략되어 즉각적인 사용이 가능하며, 혹시 모를 이슈 상황에서 Downtime이 훨씬 줄어드는 결과를 볼 수 있었습니다.



### NGINX Plus Controller-agent 통합 이미지 빌드

#### Requirements

- NGINX Plus 설치를 위한 NGINX Plus License 파일
  - nginx-repo.crt, nginx-repo.key

+ Docker Image Build를 위한 dockerfile, entrypoint.sh, nginx-plus-api.conf
  (아래 예시 문서 참조: 예시 문서는 모두 [NGINX Inc Github](https://github.com/nginxinc/docker-nginx-controller)에서 가져왔습니다.)
+ Controller-agent 설치를 위한 NGINX Controller

  

**dockerfile**

```dockerfile
FROM debian:stretch-slim

LABEL maintainer="NGINX Controller Engineering"

# e.g '1234567890'
ARG API_KEY
ENV ENV_CONTROLLER_API_KEY=$API_KEY

# e.g https://<fqdn>/install/controller-agent
ARG CONTROLLER_URL
ENV ENV_CONTROLLER_URL=$CONTROLLER_URL

# e.g True or False
ARG STORE_UUID=False
ENV ENV_CONTROLLER_STORE_UUID=$STORE_UUID

# e.g Instance location already defined in Controller
ARG LOCATION
ENV ENV_CONTROLLER_LOCATION=$LOCATION

# e.g Instance group already defined in Controller
ARG INSTANCE_GROUP
ENV ENV_CONTROLLER_INSTANCE_GROUP=$INSTANCE_GROUP

# NGXIN Plus release e.g 23
ARG NGINX_PLUS_VERSION=23

# Download certificate (nginx-repo.crt) and key (nginx-repo.key) from the customer portal (https://cs.nginx.com)
# and copy to the build context
COPY nginx-repo.* /etc/ssl/nginx/
COPY nginx-plus-api.conf /etc/nginx/conf.d/
COPY entrypoint.sh /

# Install NGINX Plus
RUN set -ex \
  && apt-get update && apt-get upgrade -y \
  && apt-get install --no-install-recommends --no-install-suggests -y curl sudo procps apt-utils apt-transport-https ca-certificates gnupg1 distro-info-data libmpdec2 \
  lsb-release python python-minimal binutils net-tools \
  && \
  NGINX_GPGKEY=573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62; \
  found=''; \
  for server in \
    ha.pool.sks-keyservers.net \
    hkp://keyserver.ubuntu.com:80 \
    hkp://p80.pool.sks-keyservers.net:80 \
    pgp.mit.edu \
  ; do \
    echo "Fetching GPG key $NGINX_GPGKEY from $server"; \
    apt-key adv --keyserver "$server" --keyserver-options timeout=10 --recv-keys "$NGINX_GPGKEY" && found=yes && break; \
  done; \
  test -z "$found" && echo >&2 "error: failed to fetch GPG key $NGINX_GPGKEY" && exit 1; \
  echo "Acquire::https::plus-pkgs.nginx.com::Verify-Peer \"true\";" >> /etc/apt/apt.conf.d/90nginx \
  && echo "Acquire::https::plus-pkgs.nginx.com::Verify-Host \"true\";" >> /etc/apt/apt.conf.d/90nginx \
  && echo "Acquire::https::plus-pkgs.nginx.com::SslCert     \"/etc/ssl/nginx/nginx-repo.crt\";" >> /etc/apt/apt.conf.d/90nginx \
  && echo "Acquire::https::plus-pkgs.nginx.com::SslKey      \"/etc/ssl/nginx/nginx-repo.key\";" >> /etc/apt/apt.conf.d/90nginx \
  && printf "deb https://plus-pkgs.nginx.com/debian stretch nginx-plus\n" > /etc/apt/sources.list.d/nginx-plus.list \
  # NGINX Javascript module needed for APIM
  && apt-get update && apt-get install -y nginx-plus=${NGINX_PLUS_VERSION}* nginx-plus-module-njs=${NGINX_PLUS_VERSION}*  \
  && rm -rf /var/lib/apt/lists/* \
  # Install Controller Agent
  && curl -k -sS -L ${CONTROLLER_URL} > install.sh \
  && sed -i 's/^assume_yes=""/assume_yes="-y"/' install.sh \
  && sed -i 's,-n "${NGINX_GPGKEY}",true,' install.sh \
  && sh ./install.sh -y \
  # cleanup sensitive nginx-plus data
  && rm /etc/ssl/nginx/nginx-repo.* \
  && rm /etc/apt/sources.list.d/nginx-plus.list \
  && rm /etc/apt/apt.conf.d/90nginx \
  && apt-key del "$NGINX_GPGKEY"

# Forward request logs to Docker log collector
RUN ln -sf /proc/1/fd/1 /var/log/nginx-controller/agent.log \
  && ln -sf /proc/1/fd/2 /var/log/nginx/error.log

EXPOSE 80

STOPSIGNAL SIGTERM

ENTRYPOINT ["sh", "/entrypoint.sh"]
```

dockerfile를 큰 부분으로 구분하면, dockerfile 구성 순서에 따라 아래와 같이 구분할 수 있습니다.

1. 변수 설정 파트
2. N+ 라이선스 및 Repo 설정 파트
3. NGINX Plus 설치 파트
4. NGINX Controller-agent 설치 파트
5. 민감성/불필요 정보 제거 파트
6. Log 연결 및 기타 설정 및 Entrypoint 지정 파트입니다.



  

**entrypoint.sh**

```bash
#!/bin/sh
#
# This script launches nginx and the NGINX Controller Agent.
#

# Variables
agent_conf_file="/etc/controller-agent/agent.conf"
agent_log_file="/var/log/nginx-controller/agent.log"
nginx_status_conf="/etc/nginx/conf.d/stub_status.conf"
api_key=""
instance_name="$(hostname -f)"
controller_api_url=""
location=""

handle_term()
{
    echo "received TERM signal"
    echo "stopping controller-agent ..."
    kill -TERM "${agent_pid}" 2>/dev/null
    echo "stopping nginx ..."
    kill -TERM "${nginx_pid}" 2>/dev/null
}

trap 'handle_term' TERM

# Launch nginx
echo "starting nginx ..."
nginx -g "daemon off;" &

nginx_pid=$!

wait_workers()
{
    while ! pgrep -f 'nginx: worker process' >/dev/null 2>&1; do
        echo "waiting for nginx workers ..."
        sleep 2
    done
}

wait_workers

test -n "${ENV_CONTROLLER_API_KEY}" && \
    api_key=${ENV_CONTROLLER_API_KEY}

# if instance_name is defined in the env vars, use it
test -n "${ENV_CONTROLLER_INSTANCE_NAME}" && \
    instance_name=${ENV_CONTROLLER_INSTANCE_NAME}

test -n "${ENV_CONTROLLER_API_URL}" && \
    controller_api_url=${ENV_CONTROLLER_API_URL}

test -n "${ENV_CONTROLLER_LOCATION}" && \
    location=${ENV_CONTROLLER_LOCATION}

test -n "${ENV_CONTROLLER_INSTANCE_GROUP}" && \
    instance_group=${ENV_CONTROLLER_INSTANCE_GROUP}

if [ -n "${api_key}" -o -n "${instance_name}" -o -n "${controller_api_url}" -o -n "${location}" -o -n "${instance_group}" ]; then
    echo "updating ${agent_conf_file} ..."

    if [ ! -f "${agent_conf_file}" ]; then
	test -f "${agent_conf_file}.default" && \
	cp -p "${agent_conf_file}.default" "${agent_conf_file}" || \
	{ echo "no ${agent_conf_file}.default found! exiting."; exit 1; }
    fi

    test -n "${api_key}" && \
    echo " ---> using api_key = ${api_key}" && \
    sh -c "sed -i.old -e 's/api_key.*$/api_key = $api_key/' \
	${agent_conf_file}"

    test -n "${controller_api_url}" && \
    echo " ---> using controller api url = ${controller_api_url}" && \
    sh -c "sed -i.old -e 's@^api_url.*@api_url = $controller_api_url@' \
	${agent_conf_file}"

    test -n "${instance_name}" && \
    echo " ---> using instance_name = ${instance_name}" && \
    sh -c "sed -i.old -e 's/instance_name.*$/instance_name = $instance_name/' \
	${agent_conf_file}"

    test -n "${location}" && \
    echo " ---> using location = ${location}" && \
    sh -c "sed -i.old -e 's/location_name.*$/location_name = $location/' \
	${agent_conf_file}"

    test -n "${instance_group}" && \
    echo " ---> using instance group = ${instance_group}" && \
    sh -c "sed -i.old -e 's/instance_group.*$/instance_group = $instance_group/' \
	${agent_conf_file}"

    test -f "${agent_conf_file}" && \
    chmod 644 ${agent_conf_file} && \
    chown nginx ${agent_conf_file} > /dev/null 2>&1

    test -f "${nginx_status_conf}" && \
    chmod 644 ${nginx_status_conf} && \
    chown nginx ${nginx_status_conf} > /dev/null 2>&1
fi

if ! grep '^api_key.*=[ ]*[[:alnum:]].*' ${agent_conf_file} > /dev/null 2>&1; then
    echo "no api_key found in ${agent_conf_file}! exiting."
    exit 1
fi

echo "starting controller-agent ..."
/usr/bin/nginx-controller-agent > /dev/null 2>&1 < /dev/null &

agent_pid=$!

if [ $? != 0 ]; then
    echo "couldn't start the agent, please check ${agent_log_file}"
    exit 1
fi

wait_term()
{
    wait ${agent_pid}
    trap - TERM
    kill -QUIT "${nginx_pid}" 2>/dev/null
    echo "waiting for nginx to stop..."
    wait ${nginx_pid}
}

wait_term

echo "controller-agent process has stopped, exiting."
```

entrypoint.sh파일은 크게 두 부분으로 구분할 수 있습니다.

1. API GW 이미지 내의 agent.conf 파일 수정 (변수 Override)

2. conf 수정에 따른 NGINX와 Controller-agent의 재시작



Controller 설치 과정에서 생성된 api_key, controller_api_url, location 등의 설정 값을 Entrypoint.sh를 통해,  ENV에서 설정한 값으로 재반영하도록 작성한 것으로 보입니다.

- 따라서, Kubernetes Deployment YAML 파일을 통해 API Key, Location, 그리고 5월 말 기준 아직 베타 기능인 instance group까지 변수를 override하여 사용할 수 있겠습니다.

Override해서 사용할 수 있는 변수는 아래와 같습니다.

```
ENV_CONTROLLER_API_KEY
ENV_CONTROLLER_API_URL
ENV_CONTROLLER_INSTANCE_NAME
ENV_CONTROLLER_LOCATION
ENV_CONTROLLER_INSTANCE_GROUP
```



  

**nginx-plus-api.conf**

```nginx
server {
    listen 8080;
    location /api/ {
        api write=on;
    }
    location = /dashboard.html {
        root /usr/share/nginx/html;
    }
    # Redirect requests for "/" to "/dashboard.html"
    location / {
        return 301 /dashboard.html;
    }
    # Redirect requests for pre-R14 dashboard
    location /status.html {
        return 301 /dashboard.html;
    }
}
# vim: syntax=nginx
```

Controller-agent는 각 API Gateway의 Metric 값을 API 조회를 통해 가져옵니다. 이를 위한 NGINX Server Context config입니다.



**Docker Build**

Build 명령어는 다음과 같습니다.

```bash
sudo docker build --build-arg CONTROLLER_URL=https://<fqdn>/install/controller-agent --build-arg API_KEY='abcdefxxxxxx' -t nginx-agent .
```

Build-argument인 CONTROLLER_URL은 Controller 버전에 따라 다를 수 있습니다.

- NGINX Controller v3.10 이전  
  `CONTROLLER_URL=https://<fqdn>:8443/1.4/install/controller`

- NGINX Controller v3.11 이후  
  `CONTROLLER_URL=https://<fqdn>/install/controller-agent`





