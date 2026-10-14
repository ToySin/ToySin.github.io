---
title: "[Container] 도커없이 컨테이너 만들기 4"
date: 2026-10-14 12:00:00 +0900
categories: [Infra, Container]
tags: [container, linux, cgroup, namespace, overlayfs, veth]
---

이제 격리는 끝났고, 자원을 할당해야 한다.

여기서 **Cgroup**(Control Groups)이라는 개념이 등장한다.

CPU, Mem 같은 시스템 자원을 묶어서 그룹을 짓는다.
그리고 프로세스를 그 그룹에 포함시키면, 그 프로세스는 그룹에 정해진 만큼의 자원만 사용할 수 있다.

![](/assets/img/posts/container-without-docker-4/01.png)

CPU에 스트레스를 줬더니 100%를 찍는다. `-c 1`은 1코어를 쓴다는 뜻이다.

![](/assets/img/posts/container-without-docker-4/02.png)

우분투 24는 cgroup v2를 써서 설정 방식이 달라(`cpu.cfs_quota_us` 대신 `cpu.max`) 찾아봤는데, 30% 제한을 걸었더니 30%까지만 먹는 것을 볼 수 있다.

cgroup은 파일시스템으로 관리된다. 제어 그룹을 만들고, 리소스를 설정하고, 그 그룹에 프로세스를 할당하면 된다.

이번에는 컨테이너 두 개를 직접 만들고 통신 테스트까지 해보는 실습을 진행했다.

![](/assets/img/posts/container-without-docker-4/03.png)

실습을 위해 필요한 명령어 몇 개를 더 복사했다.
`ping`은 통신 테스트를 위해서, `stress`는 cgroup 테스트를 위해서, `rm`과 `umount`는 `put_old`를 없애기 위해서 넣어줬다.

![](/assets/img/posts/container-without-docker-4/04.png)

네트워크 네임스페이스를 만들고 가상 장치(veth)를 연결한 뒤, IP를 할당하고 전원을 켜줬다.
네트워크 정보만 관리되고 있을 뿐, 아직 컨테이너가 네트워크에 붙은 건 아니다.

![](/assets/img/posts/container-without-docker-4/05.png)

이번에는 cgroup을 만들고 CPU 40%, 메모리 200MB, 스왑 X로 설정해줬다. 마찬가지로 아직 프로세스는 할당되지 않았다.

![](/assets/img/posts/container-without-docker-4/06.png)

`unshare`로 드디어 컨테이너 환경을 띄웠다. 띄우면서 네트워크 네임스페이스도 붙여줬다.
각종 네임스페이스들을 격리하고 cgroup에도 프로세스를 할당했다. 여기서 `echo "1"`을 했는데, PID를 fork한 환경이라는 걸 리눅스 커널이 알아서 판단하고 호스트에서의 프로세스 ID를 써준다. (아래 사진)
그리고 overlay fs도 구성했다.

![](/assets/img/posts/container-without-docker-4/07.png)

![](/assets/img/posts/container-without-docker-4/08.png)

실제로 `/redfs/merge`가 마운트 포인트가 되므로, `put_old`를 `/redfs/merge/put_old`에 만들었다.
피봇해준 뒤 `put_old`를 없애면 되는데, 컨테이너 안에서 호스트 파일시스템에 접근하는 걸 막아야 하기 때문이다.
`umount`를 하려면 `proc`을 어딘가 붙여줘야 해서, 먼저 붙이고 `umount`한 뒤 삭제하면 된다.
그러면 컨테이너는 자신만의 루트 파일시스템을 가지고(진짜 루트가 여기로 바뀐 거다), 기존 호스트 파일시스템으로 나갈 마운트 포인트는 잃어버린 것이다. **완전한 격리.**
그렇게 RED, BLUE 두 개를 준비했다.

![RED / BLUE 컨테이너 전체 구성도](/assets/img/posts/container-without-docker-4/red-blue-overview.svg){: w="960" h="640" }
_RED / BLUE 컨테이너 전체 구성도_

![](/assets/img/posts/container-without-docker-4/09.png)

`ping`도 잘 나간다. 각자 네트워크에 있는 가상 장치로 통신한다.
RED에서는 CPU stress를 주고, BLUE에서는 mem stress를 줬다.
CPU는 40%에서 제한되는 게 보이고, mem은 195M부터 차차 늘려가며 테스트했는데 205M를 줬을 때 프로세스를 종료해버린 것을 볼 수 있다.
호스트에서 `dmesg`를 해보니 메모리 제한을 넘겨서 죽여버린 것을 확인할 수 있다.
