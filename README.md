# NGINX Static router turorial
Static file (html, css, js) router example for NGINX

# Introduction

이 자료는 NGINX를 이용하여 static 파일을 제공할때 re-routing을 통하여

1. 확장자를 숨김으로써 보안성 증가
2. 파일구조는 그대로 두고 **Semantic URL**을 지원
3. **Semantic URL** 을 통한 검색엔진 최적화 (SEO)

를 위하여 작성된 자료입니다.

# Background

이번에 구현하는 시스템에서는 다국어 지원을 위한 Web static resource를 적절한 빌드환경 구성을 통하여 다음과 같은 폴더구조로 제작하였습니다.

```
/www/
     pc/
          css/
              index.css
          js/
              lib.js
              index.js
          res/
              image/
                  ...
              video/
                  ...
          html/
              index.ko.html
              index.en.html
              index.jp.html
              ....
     mobile/
          css/
              index.css
          js/
              lib.js
              index.js
          res/
              image/
                  ...
              video/
                  ...
          html/
              index.ko.html
              index.en.html
              index.jp.html
              ....
     404.html
```
본 환경을 위해셔 "pug.js"를 통해 pug로 작성된 파일을 각각의 language로 컴파일 하여 적용하였고,
이를 다음과 같은 URL로 접근하고자 하였습니다.

### Mobile, PC Routing Example
```
http://ko.abc.xyz/       ------->      /www/pc/index.ko.html

http://ko.abc.xyz/m/     ------->      /www/mobile/index.ko.html
```

### Language Routing Example
```
http://abc.xyz/          ------->      /www/pc/index.ko.html
http://www.abc.xyz/      ------->      /www/pc/index.ko.html
http://ko.abc.xyz/       ------->      /www/pc/index.ko.html
http://en.abc.xyz/       ------->      /www/pc/index.en.html
http://jp.abc.xyz/       ------->      /www/pc/index.jp.html
```

또한 허용되지 않은 URL 규칙을 이용한다면 404.html을 호출하고자 하였습니다.


# Build (생략 가능)

만약 전체적인 파일 빌드 프로세스가 궁금하다면, 본 repository의 `/src/`경로를 참고해주시기 바랍니다.

1. 해당 경로에서 다음 명령어를 입력
`$ npm install --production`
2. grunt 빌드를 수행
`$ grunt dev`
3. 최종 결과 확인
`$ cd ../www/`

# Requirements

1. 단순한 static resource를 serving 하므로, 순수 NGINX만 사용하여 단순하게 구현하여야 했습니다.
2. DNS레벨의 Load-balancing을 수행하거나, Proxy를 통한 Load-balancing을 수행하는데에 지장이 없어야 합니다.
3. restful한 URL규칙을 지켜야 하며, GET / POST Parameter의 사용을 최소화 하여야 합니다.
4. 허용되지 않은 접근이라면 404 Not Found Error를 정확하게 띄워야 합니다.
5. ubuntu 16.04 환경에서 동작합니다.
6. 본 예제에서의 default language는 **ko** 입니다.

# Implementing

## 1. NGINX 설치

`sudo apt-get install nginx`

설치가 완료되면, 다음과 같은 경로에 html root가 생성됩니다.

`/var/www/html/`

이곳을 HTML ROOT로 정의하고, 본 repository의 /www/안의 내용을 모두 복사해서 다음의 경로로 옮겨둡니다.

`/var/www/example/`

이동이 완료되었다면 `example`폴더 안에 `mobile`, `pc` 두개의 폴더만 위치하여야 합니다.

## 2. Domain 설정

*이미 개인 도메인이 있다면 생략 가능함*

일반 ip주소를 통해서는 redirect가 안되기 때문에, 테스트를 위해서 직접 테스트용 도메인(*abc.xyz*)을 바인딩 해줄 필요가 있습니다.
따라서 다음과 같은 명령어를 통해 linux host파일을 열어줍니다.

`$ sudo vi /etc/hosts`

아래 내용을 파일의 맨 아래에 추가해줍니다.
```
127.0.0.1       abc.xyz
127.0.0.1       www.abc.xyz
127.0.0.1       m.abc.xyz
```

저장 이후 웹 브라우저를 열어 해당 도메인들에 한번씩 접속해봅니다.

### 주의

위 내용을 따라한 경우에는 모든 tutorial이 끝나고 해당 파일의 추가된 내용을 **꼭** 지워줘야 합니다.

## 3. Site configuration 추가

적절한 문법을 통해 site configuration을 등록해준다.

본 tutorial에서는 이미 작성된 `abc.xyz`파일을 사용한다.

### 3-1. Configure 등록

nginx가 설치 된 폴더 (일반적으로 `/etc/nginx/`)의 `site-enabled` 폴더에 본 소스코드파일의 설정파일들을 다음과 같은 명령으로 복사하여 삽입한다.

```
$ sudo cp conf/abc.xyz /etc/nginx/site-enabled/abc.xyz
$ sudo cp conf/m.abc.xyz /etc/nginx/site-enabled/m.abc.xyz
```

### 3-2. NGINX restart

적용 완료 후 아래의 명령어를 통해 nginx를 재시작 한다.

```
$ sudo service nginx restart
```

## 4. rewrite Rule debugging setting

통상적인 경우 아래의 과정이 필요 없으며, 오류가 발생할 경우에만 수행한다.

### 4-1. rewrite log

nginx.conf 파일 (일반적으로 `/etc/nginx/nginx.conf`)을 열어 `rewrite_log on;`을 아래와 같이 `html`블록에 추가한다.

```
html {
    rewrite_log on;

    ...
}
```

### 4-2. error log 등록

각 사이트의 등록 3번 항목에서 작성한 conf파일의 `error_log`를 찾아 맨 끝의 `error`를 `notice`로 아래와 같이 치환해준다.

```
server {
    ...
    error_log /var/log/nginx/abc.xyz.error.log notice;
    ...
}
```
