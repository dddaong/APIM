---
layout: post
title:  "Arcadia and NGINX Controller Configuration"
date:   2021-07-28 15:05:00 +0900
categories: NGINXController Hands-on
---

Arcadia Finance Application 기준 "NGINX Controller Configuration 따라하기" 입니다.




# Arcadia and NGINX Controller Configuration

Arcadia Finance Application 설치 및 NGINX Controller ADC, APIM Config Hands-on 예제입니다.

- Test Application인 Arcadia Finance는 Docker 설치와 Kubernetes 설치를 각각 안내하며, 
  Docker, Kubernetes 환경 자체의 구성 방법은 이 문서에서 다루지 않습니다.
- 이 문서에는 NGINX Controller의 Service Config에 대해 안내하며, 
  Instance 추가 같은 Infrastructure 메뉴에 대해서는 다루지 않습니다.

- NGINX Controller는 버전에 큰 관계가 없으나 이 문서에서는 3.18버전을 사용합니다. 리눅스는 CentOS 7을 사용합니다.

 

## Arcadia Deployment

Arcadia Finance application은 Stock Budget, Money Transfer, Refer Friend 등의 기능을 4 개의 Micro Service 형식으로 구현해놓은 Lab 환경 구축용 어플리케이션입니다. 

<u>아래 링크에서 디테일을 확인할 수 있습니다.</u>

- [Arcadia Finance Application Gitlab](https://gitlab.com/arcadia-application) - Arcadia Gitlab
- [f5 NGINX Controller 3.x with BIG-IP Lab](https://rtd-apim-adc-controller.docs.emea.f5se.com/en/latest/index.html) - BIG-IP와 NGINX Controller의 연동 (Arcadia App 예시)



<u>각 마이크로서비스의 상세는 아래와 같습니다.</u>

![Arcadia_Micro_Services_architecture](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/Arcadia_Micro_Services_architecture.png)

| App     | URI    | Container Port : Host Port |
| ------- | ------ | -------------------------- |
| Main    | /      | 80 : 30511                 |
| Backend | /files | 80 : 31584                 |
| App2    | /api   | 80 : 30362                 |
| App3    | /app3  | 80 : 31662                 |

Host Port는 변경되어도 무방하지만, Gitlab에서 제공하는 Kubernetes Material YAML과 동일하도록 작성했습니다.





### Arcadia (Docker)

Arcadia Gitlab에서는 NGINX OSS까지 설치하도록 안내하고 있지만, NGINX Controller 및 NGINX API Gateway가 그 부분을 대체하도록 하기 위해 NGINX OSS는 설치하지 않으며, 각각의 마이크로 서비스가 별도 포트로 동작하도록 합니다. 


```bash
docker network create internal

docker run -dit --net=internal -h mainapp --name=mainapp -p 30511:80 \
  registry.gitlab.com/arcadia-application/main-app/mainapp:latest

docker run -dit --net=internal -h backend --name=backend -p 31584:80 \
  registry.gitlab.com/arcadia-application/back-end/backend:latest

docker run -dit --net=internal -h app2 --name=app2 -p 30362:80 \
  registry.gitlab.com/arcadia-application/app2/app2:latest

docker run -dit --net=internal -h app3 --name=app3 -p 31662:80 \
  registry.gitlab.com/arcadia-application/app3/app3:latest
```

- Docker는 Service Discovery를 위한 Embedded DNS(127.0.0.11)을 갖고 있으며, Arcadia 각 Container Image 내부의 php 스크립트가 Embedded DNS를 사용합니다. 아래 명령어로 확인할 수 있습니다.

  ```bash
  docker exec mainapp ping app3
  ```

- 즉, Container의 name을 기준으로 참조 및 API Query 하기 때문에 Network 생성 및 `--net=internal` 플래그 지정이 반드시 필요합니다.

- 오류가 있거나 기타 Container 재시작이 필요한 경우 아래 커맨드를 사용합니다.

  ```bash
  docker stop `docker ps -a | grep -i gitlab | gawk '{ print $1 }'`
  docker rm `docker ps -a | grep -i gitlab | gawk '{ print $1 }'`
  ```



### Arcadia (Kubernetes)

Kubernetes에 Arcadia를 배포하는 방법은 간단합니다.
아래 명령어로 yaml 파일 자체를 Gitlab에서 그대로 따올 수 있습니다.

```bash
kubectl apply -f https://gitlab.com/arcadia-application/kubernetes-materials/-/blob/master/kubernetes_deployments/arcadia-all.yaml
```

참고로 YAML 파일의 자세한 내용은 아래와 같습니다.

```bash
#########################################################################################
# FILES - BACKEND
#########################################################################################
apiVersion: v1
kind: Service
metadata:
  name: backend
  labels:
    app: backend
    service: backend
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 31584
    name: backend-80
  selector:
    app: backend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: default
  labels:
    app: backend
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
      version: v1
  template:
    metadata:
      labels:
        app: backend
        version: v1
    spec:
      containers:
      - env:
        - name: service_name
          value: backend
        image: registry.gitlab.com/arcadia-application/back-end/backend:latest
        imagePullPolicy: IfNotPresent
        name: backend
        ports:
        - containerPort: 80
          protocol: TCP
---
#########################################################################################
# MAIN
#########################################################################################
apiVersion: v1
kind: Service
metadata:
  name: main
  namespace: default
  labels:
    app: main
    service: main
spec:
  type: NodePort
  ports:
  - name: main-80
    nodePort: 30511
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: main
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: main
  namespace: default
  labels:
    app: main
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: main
      version: v1
  template:
    metadata:
      labels:
        app: main
        version: v1
    spec:
      containers:
      - env:
        - name: service_name
          value: main
        image: registry.gitlab.com/arcadia-application/main-app/mainapp:latest
        imagePullPolicy: IfNotPresent
        name: main
        ports:
        - containerPort: 80
          protocol: TCP
---
#########################################################################################
# APP2
#########################################################################################
apiVersion: v1
kind: Service
metadata:
  name: app2
  namespace: default
  labels:
    app: app2
    service: app2
spec:
  type: NodePort
  ports:
  - port: 80
    name: app2-80
    nodePort: 30362
  selector:
    app: app2
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
  namespace: default
  labels:
    app: app2
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app2
      version: v1
  template:
    metadata:
      labels:
        app: app2
        version: v1
    spec:
      containers:
      - env:
        - name: service_name
          value: app2
        image: registry.gitlab.com/arcadia-application/app2/app2:latest
        imagePullPolicy: IfNotPresent
        name: app2
        ports:
        - containerPort: 80
          protocol: TCP
---
#########################################################################################
# APP3
#########################################################################################
apiVersion: v1
kind: Service
metadata:
  name: app3
  namespace: default
  labels:
    app: app3
    service: app3
spec:
  type: NodePort
  ports:
  - port: 80
    name: app3-80
    nodePort: 31662
  selector:
    app: app3
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app3
  namespace: default
  labels:
    app: app3
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app3
      version: v1
  template:
    metadata:
      labels:
        app: app3
        version: v1
    spec:
      containers:
      - env:
        - name: service_name
          value: app3
        image: registry.gitlab.com/arcadia-application/app3/app3:latest
        imagePullPolicy: IfNotPresent
        name: app3
        ports:
        - containerPort: 80
          protocol: TCP
---
```





## NGINX Controller ADC Configuration

이번 단계에서는 NGINX Controller 설정을 통해 API GW로 아래와 같이 Config를 수행합니다.

![topology](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/topology.png)

Main App과 Backend는 PHP 스크립트 상, Embedded DNS를 사용하도록 URI가 구성되어, 직접 통신되어야 하므로, 실제 NGINX Controller에서 Config를 수행할 내용은, App2와 App3입니다.





### 1. Environment

가장 먼저 Environment를 생성합니다.

NGINX Controller의 Environment는 Kubernetes의 Namespace와 유사한 개념으로, 서로 다른 Environment에 속한 요소들은 서로를 참조할 수 없습니다.

![step1_environment](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/step1_env.png)




### 2. Gateway
다음으로는 Gateway를 생성합니다.

Instance를 직접 참조하는 Static한 방식과, Instance Group을 참조하여 해당 Instance Group에 속하는 모든 Instance에 같은 Config를 Push하는 Dynamic한 방식을 선택할 수 있습니다.

이 과정에서는 Static하게 Instance를 선택합니다.
![step2_gateway](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/step2_gw_1.png)
![step2_gateway](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/step2_gw_2.png)
![step2_gateway](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/step2_gw_3.png)

Gateway의 `Hostname (URI Formatted)`는 NGINX Config의 Server Block에 해당합니다.

Config는 NGINX Config에서 server_name Directive의 값으로 지정됩니다.



### 3. App

다음으로는 Component가 속하게 될 App을 생성합니다.

Service 구성 요소 중, 지금 생성할 App은 Components의 컨테이너 역할을 수행하며, 
App의 생성 자체만으로는 아무런 트래픽 제어 효과가 없습니다.

실제 트래픽 제어 및 각종 Config는 이 다음 과정의 Component 생성에서 수행하게 됩니다.

![step3_App](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/step3_app.png)





### 4. App - Component ( Main app, /)

먼저 `/ (Root)` 경로에서 표시할 Webpage인 Main app의 Component를 생성합니다.

![step4_ADC_Component_main](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/step4_comp_mainapp_1.png)

여기서 참조하는 Gateway에 속한 Instance들에 Config Push가 진행됩니다.



![step4_ADC_Component_main](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/step4_comp_mainapp_2.png)

URI 항목에 작성하는 `/ (Root)`는 NGINX Config에서 Location Block으로 들어가게 됩니다.
이 Config는 NGINX Config에서 Location Directive로 지정됩니다.



![step4_ADC_Component_main](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/step4_comp_mainapp_3.png)

`Workload Group`의 `Backend Workload URIs` 항목은 NGINX Config의 `Upstream`에 해당하는 부분입니다.
`advanced Configuration`에서 상세설정을 수행할 수 있습니다. 이 과정에서는 필요한 설정만을 수행합니다.



이상의 과정을 수행하고 나면, Gateway에 설정한 Hostname, 또는 IP Address로 접속하면 아래와 같은 결과를 확인할 수 있습니다.

> 아래 페이지는 메인 화면에서 `Login`, ID matt / PW ilovef5 를 사용해 로그인한 후에 보이는 화면입니다.

![step4_ADC_Component_main](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/step4_comp_mainapp_4.png)


페이지가 완전하지 않고 일부 Coming Soon 표기된 구역이 확인됩니다.



### 5. App - Component ( App2, /api )

불완전한 서비스를 완전하게 하는 과정으로, 다음 Component를 생성하는 과정을 진행합니다.

`Component URI` 부분과 `Backend URI`을 Arcadia Finance Application 설계에 맞춰 설정하는 과정입니다.

![step5_ADC_Component_app2](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/step5_comp_app2_1.png)
![step5_ADC_Component_app2](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/step5_comp_app2_2.png)

![step5_ADC_Component_app2](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/step5_comp_app2_3.png)
![step5_ADC_Component_app2](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/step5_comp_app2_4.png)

`/api` 경로의 요청에 대해 App2 Backend로 Request를 전달하는 Config가 Push되고 난 후, 하단과 우측 패널이 더이상 Coming Soon이 아니라 정상적으로 페이지로 표기되는 것이 확인됩니다.



### 6. App - Component ( App3, /app3 )

마지막으로, 하나 남은 Coming Soon 구역을 정상적인 서비스가 될 수 있도록 App3를 Gateway에서 활성화 해주는 과정을 수행합니다.

![step6_ADC_Component_app3](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/step6_comp_app3_1.png)
![step6_ADC_Component_app3](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/step6_comp_app3_2.png)
![step6_ADC_Component_app3](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/step6_comp_app3_3.png)
![step6_ADC_Component_app3](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/step6_comp_app3_4.png)

페이지 전체에 Coming Soon 구역 없이 정상적인 서비스가 수행되는 것이 확인됩니다.



## NGINX Controller Service Resource 계층 구조

지금까지 App Component의 Configuration을 완료했습니다.

다음 과정에서는 API에 대한 Configuration을 수행할텐데, API Component는 App Component와는 구조가 다소 다르기 때문에 App, API Component 간의 구조 비교 및  Service 메뉴 구조의 이해를 돕기 위해 계층 구조를 확인해보겠습니다.

![Service Resource Hierarchy](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/Service_Resources.png)



NGINX Controller의 Component는 App Component, API Component로 구분되며, 
API Component 역시 전 과정에서 설정한 App Component와 동일하게 실제 Config로 반영되는 부분입니다.

단, API Component는 Published API에서 작성하는 부분으로, 상세 Configuration 과정에는 차이가 있습니다.



## NGINX Controller APIM Configuration

이 과정에서는 Arcadia Finance Application에서 제공하는 API Spec으로 각 API에 대한 설정을 수행합니다.

엄밀히 말하면, ADC Component 설정만으로도 이미 Arcadia Finance App에서는 API가 정상적으로 Route되며, 잘 동작하고 있을 것입니다. 

그럼에도 불구하고 이 APIM 과정을 수행해 API Component를 별도로 설정하는 것은, API에서 필요한 Authentication, Ratelimit 등의 상세 설정 부분 및 전체 Application 구조가 철저히 MSA 설계를 따라, 별도의 API Backend Server에서 동작하는 경우 등을 위한 것으로 이해하면 좋겠습니다.



이 과정에서는 API Definition을 생성하고, API Version에서 API Spec을 작성, Published API(Component) 생성으로 NGINX에 Config 반영되는 부분을 살펴봅니다.



> Arcadia Finance Application은 OAS3 포맷으로 API Spec을 제공합니다.
>
> 원문 OAS3 포맷의 YAML 파일은 아래와 같습니다.
> 이 YAML API Spec에는 4가지 API URI에 대한 상세가 기록되어 있습니다.
>
> 출처는 [Swagger App - Arcadia-OAS3 Schema](https://app.swaggerhub.com/apis/F5EMEASSA/Arcadia-OAS3/2.0.1-schema)입니다.

**Arcadia API Specification - OAS3**

```yaml
openapi: 3.0.3
info:
  description: Arcadia OpenAPI
  title: API Arcadia Finance
  version: 2.0.1-schema
paths:
  /trading/rest/buy_stocks.php:
    post:
      summary: Add stocks to your portfolio
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/buy'
            example:
              trans_value: '312'
              qty: '16'
              company: MSFT
              action: buy
              stock_price: '198'
      responses:
        '200':
          description: 200 response
          content:
            application/json:
              example:
                status: success
                name: Microsoft
                qty: '16'
                amount: '312'
                transid: '855415223'
                
  /trading/rest/sell_stocks.php:
    post:
      summary: Sell stocks that you own
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/sell'
            example:
              trans_value: '212'
              qty: '16'
              company: MSFT
              action: sell
              stock_price: '158'
      responses:
        '200':
          description: 200 response
          content:
            application/json:
              example:
                status: success
                name: Microsoft
                qty: '16'
                amount: '212'
                transid: '658854124'

  /trading/transactions.php:
    get:
      summary: Get the latests transactions that have happened
      responses:
        '200':
          description: 200 response
          content:
            application/json:
              example:
                YourLastTransaction: MFST 2000

  /api/rest/execute_money_transfer.php:
    post:
      summary: Transfer money to a friend
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/money_transfer'
            example:
              amount: '92'
              account: '2075894'
              currency: GBP
              friend: Vincent
      responses:
        '200':
          description: 200 response
          content:
            application/json:
              example:
                name: Vincent
                status: success
                currency: GBP
                transid: '524569855'
                msg: The money transfer has been successfully completed


components:
  schemas:
    buy:
      type: object
      required:
        - trans_value
        - qty
        - company
        - action
        - stock_price
      properties:
        trans_value:
          type: integer
          format: int64
        qty:
          type: integer
          format: int64
        company:
          type: string
        action:
          type: string          
        stock_price:
          type: integer
          format: int64
    sell:
      type: object
      required:
        - trans_value
        - qty
        - company
        - action
        - stock_price
      properties:
        trans_value:
          type: integer
          format: int64
        qty:
          type: integer
          format: int64
        company:
          type: string
        action:
          type: string          
        stock_price:
          type: integer
          format: int64
    money_transfer:
      type: object
      required:
        - amount
        - account
        - currency
        - friend
      properties:
        amount:
          type: integer
          format: int64
        account:
          type: integer
          format: int64
        currency:
          type: string
        friend:
          type: string
```



### 1. API Definition 생성

API Definition은 ADC 과정의 App과 유사하게 API Version 및 Published API의 컨테이너 역할을 수행하며, API Definition의 생성 자체만으로는 아무런 트래픽 제어 효과가 없습니다.

![API_Definition](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/api_1_apidefinition.png)



### 2. API Version 생성

생성한 API Definition에 속할 API Version을 생성합니다.

API Version에서 작성할 내용의 대부분은 API Spec을 작성한 OAS3, WSDL Spec 등의 문서로 갈음할 수 있으며, 수동 입력도 가능합니다.

이 과정에서는 앞서 소개한 OAS3 YAML 파일을 사용합니다.

![API_Version_OAS3_YAML](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/api_2_apiversion.png)

직전 과정에서 생성한 API Definition을 선택하고, Copy and paste specification text 선택 후, YAML의 내용을 복사 붙여넣기 하면, 이하의 정보들이 자동 입력되는 것을 확인할 수 있습니다.



![API_Version_Resource_page](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/api_2_apiversion_resource.png)

Resource 페이지 각 항목의 편집 화면에서 상세 내용을 확인하면 각 API에 대한 Documentation 및 Required Method 등 상세 설정이 자동 입력된 모습이 확인됩니다. 
입력된 상세 내용은 Dev Portal에서도 자동 반영됩니다.

이렇게 해서, API Spec 파일을 사용해 API Version 설정을 간편하게 완료해 봤습니다.



### 3. Published API 생성

이 과정이 실제 API Gateway로 동작하는 NGINX Plus에 Config를 반영하는 과정입니다.



![API_Published_API](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/api_3_publishedapi.png)



API Definition과 Version 항목에서 Publish 대상을 선택합니다.
Base Path는 API 설계에 따라 달라지며 Arcadia Finance API 설계 및 API Version에 작성된 URI 구조가 Root 아래 Full Path 였으므로 `/ (Root)`를 Base Path로 지정합니다. 



![API_Published_API](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/api_3_publishedapi_deployment.png)

이 과정에서 생성할 Component가 속할 Environment, App를 선택합니다.

> API 전용 Gateway를 선택해 API 전용 Gateway를 사용하는 것도 좋은 설계 방식이 될 수 있습니다.



![API_Published_API_Routing_1](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/api_3_publishedapi_routing_1.png)

Routing 화면을 확인해보면, API Version에 작성된 API 리스트가 자동으로 화면에 나타난 것을 볼 수 있습니다.
각 항목을 살펴보면, API Path가 `/api`로 시작하는 것과 `/trading`으로 시작하는 것으로 구분됩니다.

`/trading`으로 시작하는 API Path는 Main app에서 처리하는 부분이고, `/api`로 시작하는 API Path는 App2에서 처리하는 API이므로, 각각 Route 설정을 해주겠습니다.



화면 우측에 Component 항목의 `Add New`를 선택해 Component 생성과정으로 진입합니다.



![API_Published_API_Routing_1](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/api_3_publishedapi_routing_2.png)



![API_Published_API_Routing_1](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/api_3_publishedapi_routing_3.png)

![API_Published_API_Routing_1](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/api_3_publishedapi_routing_4.png)



![API_Published_API_Routing_1](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/api_3_publishedapi_routing_5.png)



![API_Published_API_Routing_1](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/api_3_publishedapi_routing_6.png)



![API_Published_API_Routing_1](https://raw.githubusercontent.com/dddaong/APIM/main/_posts/images/api_3_publishedapi_routing_7.png)



### 4. NGINX Config 반영 확인

APIM Configuration은 웹페이지에서 즉시 확인할 수 있는 부분은 아니며, 모든 API에 대해 Request 수행 및 결과를 표시하기에는 많이 길기 때문에 NGINX Config 반영 여부 및 Config 그 자체를 확인하는 것으로 갈음하려 합니다.



아래는 API Gateway Instance로 사용된 NGINX의 `nginx.conf` 파일로 Location 블록으로 Component가 표현되어 있는 것이 확인됩니다.

```nginx
server {
                server_name xxx.xxx.xxx.xxx;
                listen 80 reuseport;
                status_zone server_81916eec-168d-3f38-8fc7-19c36b0ff81e;
                set $f5_gateway gw-arcadia;
                f5_metrics_marker gateway $f5_gateway;
                set $f5_environment env-arcadia;
                f5_metrics_marker environment $f5_environment;
                location / {
                        error_log /dev/null;
                        access_log off;
                        set $f5_app app-arcadia;
                        f5_metrics_marker app $f5_app;
                        set $f5_component comp-arcadia-mainapp;
                        f5_metrics_marker component $f5_component;
                        proxy_set_header X-Forwarded-For $remote_addr;
                        proxy_set_header Host $host;
                        proxy_set_header Connection '';
                        proxy_http_version 1.1;
                        proxy_pass http://wg-arcadia-mainapp_http_30c07e22-dc09-46b3-a9f4-4e9c394654d2;
                }
                location /api {
                        error_log /dev/null;
                        access_log off;
                        set $f5_app app-arcadia;
                        f5_metrics_marker app $f5_app;
                        set $f5_component comp-arcadia-app2;
                        f5_metrics_marker component $f5_component;
                        proxy_set_header X-Forwarded-For $remote_addr;
                        proxy_set_header Host $host;
                        proxy_set_header Connection '';
                        proxy_http_version 1.1;
                        proxy_pass http://wg-arcadia-app2_http_0a6737d2-a4c8-4a2e-b748-a25594a98bcf;
                }
                location /app3 {
                        error_log /dev/null;
                        access_log off;
                        set $f5_app app-arcadia;
                        f5_metrics_marker app $f5_app;
                        set $f5_component comp-arcadia-app3;
                        f5_metrics_marker component $f5_component;
                        proxy_set_header X-Forwarded-For $remote_addr;
                        proxy_set_header Host $host;
                        proxy_set_header Connection '';
                        proxy_http_version 1.1;
                        proxy_pass http://wg-arcadia-app3_http_a291b0f6-92fc-4302-ab15-d011b73f6b08;
                }
                location = /trading/rest/buy_stocks.php {
                        error_log /dev/null;
                        access_log off;
                        set $f5_app app-arcadia;
                        f5_metrics_marker app $f5_app;
                        set $f5_component api_comp_arcadia;
                        f5_metrics_marker component $f5_component;
                        set $f5_published_api pubapi-arcadia;
                        f5_metrics_marker published_api $f5_published_api;
                        proxy_set_header X-Forwarded-For $remote_addr;
                        proxy_set_header Host $host;
                        proxy_set_header Connection '';
                        proxy_http_version 1.1;
                        proxy_pass http://wg-arcadia-api-main_http_40c55845-54b4-48ad-b88c-b8bccbd47664;

```

Location 블록의 Value로 `= (prefix)`를 사용함으로, API Call이 아닌 트래픽에 대한 영향이 없도록 API Component가 Config로 반영되어 있는 것이 확인됩니다.





API Request 및 Response의 Security 설정이나 Access_log 설정 등을 통해 어떤 Location을 사용했는지 확인하는 방법으로 좀더 명확한 Traffic 처리 구조를 파악할 수 있겠습니다.





이상으로 NGINX Controller Configuration + Arcadia Finance Application Hands-on 문서를 마칩니다.
