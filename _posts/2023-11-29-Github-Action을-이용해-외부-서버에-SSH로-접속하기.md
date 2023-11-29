---
title: "Github Action을 이용해 외부 서버에 SSH로 접속하기"
date: 2023-11-29 20:50:00 +09:00
categories: [CI/CD, Github Action]
author: scv1702
tags: [ci/cd, github action] # TAG names should always be lowercase
---

여태까지 GET-P 프로젝트는 개발 서버에서 작업을 진행한 후 배포를 할때 직접 배포 서버에 접속해서 `git pull`을 한 후 도커 컨테이너를 빌드하는 방식이었다. 처음에는 배포를 할 일이 별로 없었기 때문에 문제가 없었지만, 개발이 점차 진행되면서 배포를 하려고 할 때마다 귀찮은 짓을 반복해야 했기 때문에 이번 기회에 Github Action을 사용하여 이를 자동화 해보았다.

필자는 Github Action 스크립트를 이용해, `main` 브랜치에 코드가 푸쉬되면 자동으로 배포 서버에 SSH 접속을 해 해당 레포지토리를 `git pull`한 후 배포를 수행하는 `./init.sh`를 실행시켰다.

## 1. Github Action 환경 변수 설정
위 과정을 자동화하기 위해선 배포 서버에 접속하는 과정이 필요하다. 배포 서버에 접속하는 것은 [ssh-action](https://github.com/appleboy/ssh-action)을 이용하면 된다. 위 레포지토리에 들어가보면, SSH 접속을 위한 `USERNAME`, `PASSWORD` 또는 `KEY`, `PORT`, `HOST`에 대한 환경 변수 설정이 필요한 것을 알 수 있다. 이들에 대한 환경 변수를 설정해보자.

다음과 같이 Github 레포지토리의 Settings에 들어간다.

![1](https://github.com/Principes-Artis-Mechanicae/Principes-Artis-Mechanicae.github.io/assets/63400830/0ce6ff54-c036-4335-86fb-f65918114ddd)

왼쪽 메뉴에 보면 Secrets and variables의 Actions가 있다.

![2](https://github.com/Principes-Artis-Mechanicae/Principes-Artis-Mechanicae.github.io/assets/63400830/dfb935fb-4e8f-45e2-9d58-a59976a42351)

New repository secret를 눌러 환경 변수를 추가하자.

![3](https://github.com/Principes-Artis-Mechanicae/Principes-Artis-Mechanicae.github.io/assets/63400830/a54d0960-e798-4dc9-a157-04d1c208c7e1)

참고로 환경 변수명은 `REMOTE_SSH_HOST`, `SSH_HOST` 등 아무거나 해도 상관 없다. 

![4](https://github.com/Principes-Artis-Mechanicae/Principes-Artis-Mechanicae.github.io/assets/63400830/e8e3c04f-1f14-4312-bb71-dba0274c43ae)

본 프로젝트의 경우 개인 소유의 계정에 접속을 하기 때문에 비밀번호가 유출되는 것을 방지하여 SSH Key를 사용하려고 한다. SSH KEY를 사용하는 경우 `ssh-keygen`을 이용해 공개키와 비밀키를 생성한 다음 공개키는 서버의 `~/.ssh/authorized_keys`에 등록한 다음 비밀키를 환경 변수로 설정하면 된다.

## 2. Github Action 스크립트 작성
이제 Github Action 스크립트를 작성해보자. 프로젝트의 루트 경로에서 `.github/workflows/deploy.yml` 파일을 생성하고 다음과 같이 입력하자.

~~~yml
name: deploy
on:
  push:
    branches: ['main']
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3.3.0
      - name: execute remote ssh & deploy backend server
        uses: appleboy/ssh-action@master
        with:
          host: {% raw %}${{ secrets.REMOTE_SSH_USERNAME }}{% endraw %}
          username: {% raw %}${{ secrets.REMOTE_SSH_USERNAME }}{% endraw %}
          key: {% raw %}${{ secrets.REMOTE_SSH_KEY }}{% endraw %}
          port: {% raw %}${{ secrets.REMOTE_SSH_PORT }}{% endraw %}
          script: |
            cd /home/scv1702/get-p-backend
            git pull origin main
            ./init.sh
~~~

먼저 1번 라인의 `name`은 해당 워크플로우의 이름을 말한다. 중요한 것은 밑의 `secrets` 환경 변수 설정인데, 아까 설정했던 환경 변수명을 올바른 위치에 적어준다. 그리고 `scripts`에는 SSH 접속을 한 다음 실행할 명령어들을 적으면 되는데, 필자는 맨 처음 설명한 작업을 하도록 작성했다. 또한 `main` 브랜치가 아니라 `deploy`나 다른 브랜치에 푸쉬가 됐을때 스크립트가 실행되도록 하고 싶으면 `on.push.branches`를 수정하면 된다.

## 3. 제대로 배포가 되는지 확인
스크립트를 작성했으면 `main`에 푸쉬해보자. `main`에 푸쉬를 했을때 다음과 같이 Github Action 워크플로우가 자동으로 실행되면 성공이다.

![5](https://github.com/Principes-Artis-Mechanicae/Principes-Artis-Mechanicae.github.io/assets/63400830/7359f89e-5a59-417e-8dd8-86cf9e8213ea)

## 후기
처음에는 자동 배포를 위해 Jenkins를 사용할지, Github Action을 사용할지 고민했는데 아직까지 현재 단계에서는 Jenkins를 사용할 무거운 작업들이 딱히 없는 것 같아 가볍고 빠르게 자동 배포를 구축할 수 있는 Github Action을 채택했다. 향후에 테스트가 많아지거나 빌드 시간이 오래 걸리는 경우에는 Jenkins로 바꾸는 방법도 고려해보려고 한다.
