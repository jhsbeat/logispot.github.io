---
layout: post
title: 'Envoy를 이용한 Laravel 프로젝트 배포 자동화'
author: hosang.jeon
date: 2018-05-27 14:00
categories: [PHP/Laravel]
tags: [php, envoy, deployment]
image: /files/images/laravel_coding.jpg
---

엔보이(Envoy)는 라라벨 팀이 만들어서 배포한 php 도구 중 하나로 원격 서버의 작업을 자동화하기 위해서 사용됩니다. 엄밀히 말하자면 엔보이는 배포 도구라고 말할 수는 없습니다. 그러나 엔보이를 이용하면 원격 서버에서 라라벨 프로젝트의 배포를 위해 수행해야하는 순차적인 작업들을 아주 쉽고 간단하게 자동화할 수 있습니다. 이번 글에서는 엔보이를 이용해서 라라벨 프로젝트의 배포 스크립트를 작성하고, 이를 이용해서 실제 배포작업이 어떤 방식으로 이루어지는 지에 대해서 알아보도록 하겠습니다.

---

## Pre-requisite


라라벨 프로젝트를 개발하고 있는 상황이라면 당연히 갖추어져 있어야할 환경이지만, 서버에서 별도로 엔보이 태스크 스크립트를 실행하고자 하시는 분들을 위하여 필수적으로 필요한 내용을 적어 두었습니다. 엔보이 스크립티를 실행할 서버에 이미 PHP와 composer가 설치되어 있다면 바로 다음 단계로 넘어가시면 됩니다.

- php : 엔보이 태스크 스크립트는 PHP 파일입니다. 서버에는 반드시 PHP가 설치되어 있어야 합니다.
- composer : PHP 의존성 관리도구인 composer가 설치되어 있어야 합니다.

---

## 엔보이 설치

앞에서도 설명했지만, 엔보이는 엄밀히 말하자면 배포 도구는 아닙니다. 하지만, 라라벨 팀에서 내어놓은 엔보이를 이용하면 간단하고 단순한 문법으로 원격 서버의 작업을 기술할 수 있습니다. 엔보이는 블레이드(blade) 문법을 사용하기 때문에 기존에 라라벨 개발에 익숙한 개발자들에게 추가로 문법을 익혀야하는 부담이 없습니다. 하지만, 아쉽게도 엔보이는 Mac과 Linux 운영체제에서만 정상동작합니다.

엔보이 설치를 위한 필수 조건(PHP, composer) 준비가 완료되었다면 아래와 같이 composer 명령어를 이용하여 엔보이를 설치합니다. 엔보이는 특정 프로젝트에만 국한되어 사용되는 도구가 아니기 때문에 전역(global)로 설치합니다.

{% highlight bash %}
$ composer global require laravel/envoy
$ echo 'export PATH=$PATH:$HOME/.composer/vendor/bin' >> ~/.bash_profile
$ source ~/.bash_profile
$ envoy --version
Laravel Envoy 1.4.1
{% endhighlight %}

---

## 원격 접속을 위한 기본 설정

엔보이는 원격 서버에 접속하여 스크립트로 정의된 작업을 수행하는 태스크 러너(Task Runner)입니다. 즉, 기본적으로 원격 서버에 접근할 수 있는 인증정보와 권한정보가 있어야 합니다. 이 내용은 엔보이 자체와 직접적인 관련이 있지는 않지만 순서대로 따라해보시는 분들을 위해서 간단히 설명하고 넘어가도록 하겠습니다.

### ssh 공개키(public key) 복사

엔보이 스크립트를 실행하고자 하는 서버에 ssh 공개키를 배포 대상이되는 원격 서버에 복사해 두어야 합니다. 이를 위해서는 ssh 키를 먼저 생성해야 합니다. 특별히 다르게 지정하지 않았다면 ssh 키의 기본(default) 경로는 `~/.ssh` 입니다. 만약, 이 디렉토리 안에 `.pub`으로 끝나는 파일이 없다면 아직 ssh 키가 만들어지지 않은 것 입니다. 이 경우에는 `ssh-keygen` 명령어를 이용해서 ssh 키를 생성합니다.

ssh 키가 이미 있거나 새로 생성하였다면 공개키를 서버에 복사합니다. 공개키는 `.pub` 확장자를 가지고 있으며 특별히 암호화 방식이나 파일의 이름을 바꾸지 않았다면 `~/.ssh/id_rsa.pub`과 같을 것입니다. 이 파일을 열어서 내용을 복사한 뒤, 작업을 실행할 원격 서버의 `~/.ssh/authorized_keys` 파일에 붙여넣기 하도록 합니다.     

### ssh config 설정

일반적인 경우에는 원격서버에 접속하기 위해서 ssh config를 추가적으로 설정해줄 필요는 없습니다. 그러나 우리가 작성하려고 하는 엔보이 스크립트는 git에 의해서 관리되는데 이 때 스크립트에 작성한 서버의 호스트 정보 등이 노출 될 수 있습니다. 이 때문에 간단하게 ssh config를 통해 민감한 정보를 숨길 수 있습니다.

먼저, `~/.ssh/config` 파일이 있는지 확인한 뒤 해당 파일이 없다면 새로 생성합니다. 파일이 준비되었다면 아래와 같이 원격 서버의 정보를 기록합니다. 아래의 내용 중 `your_server`는 해당 원격 서버를 지칭하기 위한 이름으로 원하는 어떤 이름이든 사용이 가능합니다. `YOUR_PUBLIC_DNS`와 `SERVER_USER_NAME`은 실제 원격 서버의 호스트명과 사용자명으로 변경하여 사용합니다. 마지막으로, `IdentityFile`은 필수가 아니며 앞에서 공개키 복사 과정 없이 별도의 개인키를 사용하는 경우 지정합니다. 만약, 필요 없다면 이 부분은 삭제해도 무방합니다.

{% highlight linux-config %}
Host your_server
    Hostname YOUR_PUBLIC_DNS
    User SERVER_USER_NAME
    IdentityFile ~/path/to/yours.pem
{% endhighlight %}

만약, ssh 설정이 정상적으로 완료되었다면 아래와 같이 간단히 호스트의 이름(별칭)을 지정하는 것 만으로 원격 서버에 접속이 가능해집니다. 이 때, 원격 서버에 접속하기 위해서 추가적인 비밀번호(passphrase)를 묻는 과정이 있어서는 안됩니다. 그 이유는 스크립트를 통해 자동화를 하는 경우 사람이 직접 입력해야하는 프롬프트가 나타나서는 안되기 때문입니다.

{% highlight bash %}
$ ssh your_server
{% endhighlight %}

---

## 엔보이 태스크 스크립트 작성

엔보이 태스크 스크립트를 실행할 수 있는 준비가 완료되었으니 이제 엔보이 스크립트를 작성해 보도록 하겠습니다. 우선, 엔보이 태스크 스크립트의 동작 원리를 이용하기 위해서 매우 간단한 태스크를 작성합니다. 라라벨 프로젝트 루트 디렉토리에 `Envoy.blade.php`(첫 번째 문자 E는 반드시 대문자) 파일을 생성하고 아래와 같은 간단한 태스크를 작성해 보도록 하겠습니다.

{% highlight bash %}
@servers(['web' => ['your_server']])

@task('ls', ['on' => 'web'])
    ls -la
@endtask
{% endhighlight %}

이 스크립트는 `ls`라는 이름의 태스크를 정의합니다. 이 태스크는 `web`이라는 이름으로 지정된 서버에 접속하여 `ls -la` 명령어를 실행합니다. `web`이라는 이름의 서버의 실제 값에는 원격 서버의 호스트 정보가 들어가야 하지만 우리는 앞서 ssh config 파일을 이용해서 호스트 정보를 숨길 수 있었기 때문에 `your_server`와 같은 호스트 별칭만 사용하면 됩니다. 엔보이 스크립트 작성이 끝났다면 아래와 같이 태스크를 실행합니다.

{% highlight bash %}
$ envoy run ls
[your_server]: total 128
[your_server]: drwxr-xr-x   12 your_user your_user  4096 May 20 06:28 .
[your_server]: drwxr-xr-x    4 root     root      4096 Apr 25 12:11 ..
[your_server]: -rw-------    1 your_user your_user 10342 May 24 00:13 .bash_history
[your_server]: -rw-r--r--    1 your_user your_user   220 Apr 25 12:11 .bash_logout
[your_server]: -rw-r--r--    1 your_user your_user  3771 Apr 25 12:11 .bashrc
...
{% endhighlight %}

아주 간단한 태스크 스크립트지만, 이를 통해 엔보이가 어떠한 역할로 사용되며 또, 어떤식으로 동작하는지를 이해할 수 있습니다. 엔보이는 앞에서도 설명하였지만 원격 서버의 작업을 실행할 수 있도록 도와주는 역할을 합니다. 그럼 이제 본격적으로 실제 배포에 필요한 태스크를 작성해 보도록 하겠습니다.

---

## @setup 작성

`@setup` 지시자(directives)는 엔보이 태스크가 실행되기 전에 먼저 수행되어야하는 PHP 코드를 정의합니다. 이 부분에는 주로 변수 선언 등의 작업이 포함됩니다. 아래와 같이 `@setup` 부분을 추가하고 태스크 스크립트를 수정해 보도록 하겠습니다.

{% highlight bash %}
@servers(['web-1' => ['your_server1'], 'web-2' => ['your_server2']])

@setup
    $environment = isset($env) ? $env : "web-1";
@endsetup

@task('ls', ['on' => $environment])
    ls -la
@endtask
{% endhighlight %}

`@setup` 블록에서 정의한 변수가 어떻게 사용되는지 설명하기에 앞서서 수정된 태스크 스크립트를 먼저 실행해보도록 하겠습니다. 참고로 엔보이 스크립트 실행시 변수 값을 전달하기 위해서는 `--` 옵션을 사용할 수 있습니다. 아래의 명령어 실행 중 두 번째에 사용된 `--env=web-2` 부분을 참고하시기 바랍니다.

{% highlight bash %}
$ envoy run ls
[your_server1]: total 128
[your_server1]: drwxr-xr-x   12 your_user your_user  4096 May 20 06:28 .
[your_server1]: drwxr-xr-x    4 root      root      4096 Apr 25 12:11 ..
[your_server1]: -rw-------    1 your_user your_user 10342 May 24 00:13 .bash_history
[your_server1]: -rw-r--r--    1 your_user your_user   220 Apr 25 12:11 .bash_logout
[your_server1]: -rw-r--r--    1 your_user your_user  3771 Apr 25 12:11 .bashrc
...

$ envoy run ls --env=web-2
[your_server2]: total 128
[your_server2]: drwxr-xr-x   12 your_user your_user  4096 May 20 06:28 .
[your_server2]: drwxr-xr-x    4 root     root      4096 Apr 25 12:11 ..
[your_server2]: -rw-------    1 your_user your_user 10342 May 24 00:13 .bash_history
[your_server2]: -rw-r--r--    1 your_user your_user   220 Apr 25 12:11 .bash_logout
[your_server2]: -rw-r--r--    1 your_user your_user  3771 Apr 25 12:11 .bashrc
...
{% endhighlight %}

두번의 엔보이 스크립트가 모두 정상적으로 실행되었다면 출력 결과에서 차이점을 발견하실 수 있습니다. 그것은 바로 태스크가 실행된 원격 서버가 다르다는 점입니다. 첫 번재 실행시에는 `env` 변수 값을 지정해 주지 않았기 때문에 기본 값인 `web-1`이 사용되었고, 두 번째 실행시에는 `--env` 옵션을 이용해서 `web-2`라는 값을 넘겨 주었기 때문에 작업이 실행되는 원격 서버가 바뀌게 되는 것입니다.

---

## 실제 배포 스크립트

지금까지는 엔보이 태스크 스크립트를 실행하기 위한 준비, 개념에 대한 이해, 간단한 예제 태스크 실행에 대해서 알아보았습니다. 그럼 실제로 라라벨 프로젝트의 배포는 어떤 식으로 이루어지는 알아보도록 하겠습니다. 참고로, 같은 라라벨 프로젝트라고 해도 각 회사별로 상황에 맞는 배포 전략과 방식을 가지고 있습니다. 이 글에서 설명하는 배포 방식 및 전략은 그 중 한가지 방법이라고 생각하시면 됩니다.

우리는 원격 서버에서 소스코드를 다운로드 한 뒤 해당 소스코드로부터 서버 환경을 구성하고 심볼릭 링크 변경하는 방식으로 배포 태스크 스크립트를 작성하도록 하겠습니다. 이러한 방식의 배포 전략의 장점은 우선, 서비스의 중단(downtime) 없이 무중단 배포가 가능하다는 점과, 배포 이후에 예측하지 못했던 이슈 발생시 간단히 심볼릭 링크를 변경하는 것만으로 배포 이전의 상태로 원복이 가능하다는 점입니다. (물론, 마이그레이션 롤백 등의 과정이 추가로 필요한 경우도 있습니다). 결국, 우리가 라라벨 프로젝트를 배포하기 위해서 실행할 원격 서버의 작업은 다음과 같은 순서를 같습니다.

1. 배포 준비 (디렉토리 생성 등)
2. 소스코드 복제(clone)
3. 공용(shared) 리소스 연결 (`.env`, `storage` 등)
4. 의존성 설치 (composer, node packages)
5. 프론트엔드 에셋 빌드
6. DB 마이그레이션 실행
7. 프로젝트 루트 디렉토리 심볼릭 링크 변경
8. 웹 서버, php fpm(FastCGI Process Manager) 재실행

다시 말하지만, 회사별로 또는 프로젝트 별로 배포 절차와 방식, 전략은 모두 다를 수 있습니다. 이 글에서 설명하는 배포 순서는 하나의 예시일 뿐이며 상황에 맞추어 각자 적합한 방식으로 수정해서 사용할 수 있습니다.

위에서 설명한 순서를 대략적으로 엔보이 태스크 스크립트로 정리하면 다음과 같습니다. 참고로, 이 스크립트의 일정 부분은 "라라벨로 배우는 실전 PHP 웹 프로그래밍" 서적을 참고하여 작성되었습니다.

{% highlight bash %}
@servers(['staging' => ['staging_server'],'production' => ['production_server']])

@setup
    // 서버 정보
    $username = 'your_username';
    $usergroup = 'your_usergroup';

    // 프로젝트 정보
    $environment = isset($env) ? $env : "staging";  // 배포할 환경
    $base_dir = "/home/{$username}/www";  // 배포를 진행할 최상위 루트 경로
    $project_root = "{$base_dir}/your_project_name";  // 프로젝트 루트 경로
    $shared_dir = "{$base_dir}/shared"; // 공용 파일의 경로
    $release_dir = "{$base_dir}/releases";  // 배포된 디렉토리들의 부모 경로
    $distname = 'release_' . date('YmdHis');  // 현재 시점의 배포 디렉토리 명

    // Git 설정
    $remote = 'git@github.com:your_name/your_project.git';
    $branch = 'master';  // 반드시 master가 아니어도 상관 없습니다.

    // 배포를 위해서 반드시 필요한 디렉토리들
    $required_dirs = [
        $shared_dir,
        $release_dir,
    ];

    // 공유 파일, 또는 디렉토리 들
    $shared_items = [
        "{$shared_dir}/.env" => "{$release_dir}/{$distname}/.env",
        "{$shared_dir}/storage" => "{$release_dir}/{$distname}/storage",
        "{$shared_dir}/cache" => "{$release_dir}/{$distname}/bootstrap/cache",
        "{$shared_dir}/files" => "{$release_dir}/{$distname}/public/files",
    ];
@endsetup

@task('deploy', ['on' => [$environment]])
    echo "=============== 1. 배포 준비 (디렉토리 생성 등)..."
    @foreach ($required_dirs as $dir)
        [ ! -d {{ $dir }} ] && mkdir -p {{ $dir }};
    @endforeach

    echo "=============== 2. 소스코드 복제(clone)..."
    cd {{ $release_dir }} && git clone -b {{ $branch }} {{ $remote }} {{ $distname }};

    echo "=============== 3. 공용(shared) 리소스 연결 (.env, storage 등)..."
    [ ! -f {{ $shared_dir }}/.env ] && cp {{ $release_dir }}/{{ $distname }}/.env.example {{ $shared_dir }}/.env;
    [ ! -d {{ $shared_dir }}/storage ] && cp -R {{ $release_dir }}/{{ $distname }}/storage {{ $shared_dir }};
    [ ! -d {{ $shared_dir }}/cache ] && cp -R {{ $release_dir }}/{{ $distname }}/bootstrap/cache {{ $shared_dir }};
    [ ! -d {{ $shared_dir }}/files ] && cp -R {{ $release_dir }}/{{ $distname }}/public/files {{ $shared_dir }};

    @foreach($shared_items as $global => $local)
        [ -f {{ $local }} ] && rm {{ $local }};
        [ -d {{ $local }} ] && rm -rf {{ $local }};
        ln -nfs {{ $global }} {{ $local }};
    @endforeach

    chmod -R 777 {{ $shared_dir }}/storage;
    chmod -R 777 {{ $shared_dir }}/cache;
    chmod -R 777 {{ $shared_dir }}/files;
    chgrp -h -R {{ $usergroup }} {{ $release_dir }}/{{ $distname }};

    echo "=============== 4. 의존성 설치 (composer, node packages)..."
    cd {{ $release_dir }}/{{ $distname }} && composer install --prefer-dist --no-scripts --no-dev;
    cd {{ $release_dir }}/{{ $distname }} && npm install;

    echo "=============== 5. 프론트엔드 에셋 빌드..."
    cd {{ $release_dir }}/{{ $distname }} && npm run production;

    echo "=============== 6. DB 마이그레이션 실행..."
    php artisan migrate

    echo "=============== 7. 프로젝트 루트 디렉토리 심볼릭 링크 변경..."
    ln -nfs {{ $release_dir }}/{{ $distname }} {{ $project_root }};

    echo "=============== 8. 웹 서버, php fpm(FastCGI Process Manager) 재실행
..."
    sudo service nginx restart;
    sudo service php7.2-fpm restart
@endtask
{% endhighlight %}

물론, 이 태스크 스크립트를 그대로 복사해서 작성한다고 해서 배포 스크립트가 실행되는 것은 아닙니다. `@setup` 등의 구문에서 임의로 지정한 이름(ex. `your_username`)의 값을 실제 여러분의 환경에 맞게 변경해야만 합니다. 만약, 이와 동일한 방식으로 배포가 완료되었다면 여러분들이 지정한 경로에 구동 가능한 버전의 라라벨 프로젝트가 준비되어있을 것입니다.

---

## 맺음말

이 글에서는 실제 라라벨 프로젝트의 동작을 위한 추가적인 작업(웹서버 설정, php 설정 등)에 대해서는 다루지 않았습니다. 다만, 엔보이를 이용해서 변경된 소스코드의 내용을 서비스의 중단 없이 서버에 간편하게 배포하는 방법에 대해서 알아보았습니다. 엔보이를 젠킨스(Jenkins)와 함께 사용하면 배포 현황과 상태를 손쉽게 모니터링 할 수 있으며 테스트 및 스테이징 서버에 주기적인 테스트와 배포를 가능하게 합니다.

사실 원격 서버의 작업을 자동화할 수 있는 방법은 엔보이 태스크 러너 이외에도 존재합니다. 그러나 엔보이는 라라벨 프로젝트에서 사용하는 blade, php 문법을 그대로 이용해서 스크립트를 적성할 수 있다는 점과 사용하기 매우 간단하다는 장점이 있습니다. 또한, 라라벨 팀에서 직접 내놓은 도구이기 때문에 라라벨 프로젝트와 관련된 배포, 원격 작업 실행 방법에 대해서 따라할 수 있는 참고 사례(Best Practice)가 많다는 점도 큰 장점 중 하나입니다.
