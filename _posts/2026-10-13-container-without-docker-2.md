---
title: "[Container] 도커없이 컨테이너 만들기 2"
date: 2026-10-13 12:00:00 +0900
categories: [Infra, Container]
tags: [container, linux, overlayfs, overlay, mount]
---

패키징과 루트 격리는 잘할 수 있다.
이제 문제는 중복된 파일들이다.

나는 데이터베이스가 필요하고, 너는 데이터베이스랑 웹서버가 필요하면

- 내꺼: 우분투, 데이터베이스
- 너꺼: 우분투, 데이터베이스, 웹서버

이런 식으로 우분투와 데이터베이스가 두 번 중복돼서 같은 호스트 머신에 저장되어야 한다.
우리의 패키징 방식을 생각하면 필요한 것들을 한 디렉토리에 몰아서 `cp` 해뒀으니, 각자 필요한 것들을 `cp`해서 패키징해 보면 중복된 것들이 나오게 된다.

중복을 해결하기 위해 **overlay filesystem**을 사용한다.
도커에서 다운받는 걸 기준으로 설명해 보자면, 도커에서 이미지를 받으면 Lower layer를 받는다.
컨테이너를 실행하면 Lower layer 위에 Upper layer를 얹어서 실행한다.
그리고 컨테이너에 실제로 마운트되는 건 모든 게 합쳐진 Merged View다.
Lower layer는 Read-only이고, 뭔가 수정하게 되면 Upper layer에 수정 기록이 남는다.

![overlay filesystem 레이어 구조](/assets/img/posts/container-without-docker-2/overlay-layers.svg){: w="960" h="440" }
_overlay filesystem 레이어 구조_

![](/assets/img/posts/container-without-docker-2/01.png)

`myroot` 위에 얹을 `tools`를 하나 만들었다. `rm`과 `which`를 옮겨뒀다.

![](/assets/img/posts/container-without-docker-2/02.png)

overlay를 할 디렉토리를 만든다.

- `container`는 upper layer 역할
- `merge`는 실제 mount point가 될 곳
- `work`는 `container`에 atomic한 write를 하기 위해 존재한다.

`myroot`와 `rootfs/merge`를 비교해 보면 `usr/bin/{which,rm}`이 차이 나는 것을 확인할 수 있다.

![](/assets/img/posts/container-without-docker-2/03.png)

`merge`에서 `escape_chroot`를 삭제했더니 `merge`에서는 없어졌지만 `myroot`에는 그대로 남아 있었다.
대신 `container`에 `escape_chroot`가 삭제되었다는 마킹이 남았다.

지금까지 정리하자면, 컨테이너의 루트 파일시스템을 격리시켜서 호스트와 관계없이 사용할 수 있었고, 중복되는 파일들은 오버레이 파일시스템을 사용해서 해결할 수 있었다.
도커에서 pull 받으면 레이어를 하나씩 받는 게 보인다.

탈옥과 중복은 해결했다.
