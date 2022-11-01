# SpringBoot SSL 인증서 적용
## 1. TLS 인증서 발급 및 PEM키 발급

<br>
HTTPS 적용을 위해서 여러 가지 방법이 존재할 수 있는데, 그 중 **Let’s Encrypt**라는 비영리 기관을 통해서 무료로 TLS 인증서를 발급받을 수 있다.

[Let’s Encrypt](https://letsencrypt.org/)

그리고 인증서 발급을 위해선 `Certbot`을 활용해서 발급받을 수 있다.

[Certbot](https://certbot.eff.org/)

Certbot의 설치는 공식 홈페이지에서 권장하는 `snap` 을 통해 설치한다.
`snap`을 통해 설치하는 과정은 마치 `homebrew`를 통해 각종 라이브러리 등을 설치하는 과정과 유사하다.

Certbot이 인증서 발급을 해주기 위해선 다음과 같은 과정이 필요하다.

> 1. Certbot 인증서 요청
> 2. 요청한 도메인 소유주 확인
> 3. 인증서 발급

그리고 아래 3가지 방법을 통해서 소유주가 자기 자신임을 입증할 수 있다. 

> 1. **standalone** - 가상 웹서버를 가동하여 도메인소유주 확인
> 2. **webroot** - 자신의 웹서버가 제공하는 특정 파일로 도메인소유주 확인
> 3. **dns** - dns 레코드에 특정 값을 작성하여 도메인소유주 확인

일반적으로 1. standalone과 2. webroot 방식을 통해서 입증이 가능한데, standalone 방식이 webroot 방식에 비해 훨씬 편리한 반면 80번 포트가 개방되어 있어야 한다는 단점이 있다. 즉 다시 말해서 현재 running중인 WAS를 중지해야 standalone 방식을 사용할 수 있다.

해당 방식을 통해 도메인 소유주 확인이 끝나면, **privkey.pem** 파일과 **fullchain.pem** 파일을 제공받을 수 있다. 

<br>

---
## 2. Web Server에 HTTPS 적용하기

<br>
이제 발급받은 pem 파일을 사용해서 HTTPS를 적용해야 하는데, 웹서버를 사용하고 있다면 보다 편리하게 적용할 수 있다. 필자의 경우 Nginx를 사용하고 있기에, Nginx 기준으로 작성하였다.

<br>

```bash
 server {
        listen       443 ssl http2;
        listen       [::]:443 ssl http2;
        
        server_name  _;
        root         /usr/share/nginx/html;

        ssl_certificate /etc/letsencrypt/live/2ntrip.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/2ntrip.com/privkey.pem;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;
        include /etc/nginx/conf.d/service-url.inc;

        location / {
                proxy_pass $service_url;

                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection “upgrade”;
                proxy_set_header Origin “”;

                proxy_set_header X-Real-IP*$remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header Host $http_host;
        }


        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }

    server {
        listen     80;
        listen     [::]:80;

        location / {
                proxy_pass https://localhost;
        }
    }
} 
```
  
필자의 `nginx.conf` 파일이다. 웹 서버가 2개의 포트에 대해서 listening을 하고 있는 것을 볼 수 있는데, 첫 번째로는 아래의 80포트에 대해서 간략하게 설명하겠다.

일반적으로 사용자가 홈페이지 창에 `2ntrip.com` 과 같이 접속하게 된다면, `http://2ntrip.com` 주소로 접속하게 된다. 즉, 80번 포트로 접속을 시도하게 되는 것이다. 우리는 이제 HTTPS를 적용하고자 함으로, 80번 포트로 들어오는 접속도 모두 HTTPS를 적용하고자 `https://localhost` 로 proxy를 전달한다.

두 번째로, 443 포트에 대한 listening 하고 있는 부분이다. 여기서 우리가 주목해서 봐야 할 문장은 바로 이곳이다.
```bash
ssl_certificate /etc/letsencrypt/live/2ntrip.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/2ntrip.com/privkey.pem;
```

nginx를 설치한 이후, 최초로 conf 파일에 접속해보면 80포트에 대해서는 conf 파일이 활성화 되어 있는 반면 443 포트에 대해서는 주석 처리 되어 있는 것을 확인 할 수 있다. 해당 부분의 주석을 해제하고, 443 포트로 들어오는 모든 요청들에 대해서 다음과 같이 HTTPS 인증서 (2개의 pem파일) 을 입력한다.

이후 443포트로 들어온 요청이 성공적으로 HTTPS 인증서를 거쳐서 $service_url 로 전달되는 모습을 볼 수 있다. 이때 해당 환경변수를 적절히 활용한다면 Blue-Green, Port-Routing 등의 기술도 적용할 수 있을 것이다.

추가로 현재 $service_port, 80, 443 총 3개 이상의 포트를 사용하게 되는데, 해당 포트들에 대해서 EC2 인스턴스의 인바운드 규칙을 편집하지 않으면 아무리 편집해도 소용없게 된다. 해당 포트들로 요청이 정상적으로 들어올 수 있도록 EC2 인스턴스의 인바운드 규칙에서 사용하는 포트들에 대해서 모든 ipv4 및 ipv6 요청에 대해 개방하도록 한다.

<br>

---
## 3. SpringBoot에 SSL 인증서 적용하기

<br>
위 과정을 통해서 인증서를 발급받고, 들어오는 요청들을 웹서버에서 HTTPS 인증서를 거칠 수 있도록 하더라도 결국 WAS에서 SSL을 적용하지 않으면 불가능하다. 따라서 해당 인증서를 SpringBoot 내에도 적용해야 하는데, 문제는 SpringBoot에서는 pem파일로 되어 있는 인증서를 사용할 수 없다는 점이다.

따라서 이를 해결하기 위해서 pem 파일의 인증서를 .p12(pkcs12) 파일로 변환해야 한다. 변환을 위해 **openssl**를 사용 할 것이다.

```
openssl pkcs12 -inkey [privatekey] -in [certificate] -export -out [출력할 파일명]
```

다음과 같은 명령어로 pem 파일을 pkcs12 파일로 변환할 수 있다. 해당 변환 과정에서 
ssl key의 암호를 입력하라고 할텐데, 해당 암호는 반드시 기억하고 있어야 한다.

이후 변환된 pkcs12 파일을 프로젝트 내부 또는 외부에 저장해야 하는데, 외부에 저장하는 경우 다른 라이브러리를 통해 또 다시 당겨와야 한다. 필자는 로컬에서 개발하는 프로젝트의 properties 파일과 EC2 상의 properties 파일이 다르고, key 파일은 오직 EC2 리눅스 서버 내부에만 존재하기 때문에, pkcs12 파일을 프로젝트 resources 내부에 저장하였다.

저장한 이후에는 properties (또는 yml)에서 ssl 키에 대한 정보를 제공해야 한다. 아래와 같이 properties 파일을 수정한다.

```
server.ssl.key-store=[key 파일 위치]
server.ssl.key-store-type=PKCS12
server.ssl.key-store-password=[key 파일 password]
server.http2.enabled=true
``` 

다음과 같은 작업이 모두 끝난 다음에는 아래와 같은 과정이 모두 진행된 것이다.

> 1. 도메인에 HTTPS 인증서 발급 완료
> 2. 웹서버 (Nginx)를 통해 HTTP로 들어오는 요청 모두 HTTPS로 라우팅, HTTPS로 들어오는 모든 요청에 pem 키파일로 인증 활성화
> 3. WAS (Tomcat)에서 HTTP가 아닌 HTTPS를 사용하는 서버 오픈 

