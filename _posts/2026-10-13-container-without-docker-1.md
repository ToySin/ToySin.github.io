---
title: "[Container] 도커없이 컨테이너 만들기 1"
date: 2026-10-13 12:00:00 +0900
categories: [Infra, Container]
tags: [container, linux, chroot, pivot_root, namespace]
---

![](/assets/img/posts/container-without-docker-1/01.png)

`chroot`는 실행한 프로세스부터 그 하위 프로세스들까지의 루트 파일시스템을 바꾸는 명령어다.
일단 sudo 권한이 필요하다는 걸 알 수 있고, `/bin/sh`을 실행하려고 했지만 루트가 바뀌었기 때문에 명령어를 찾지 못한다.

![](/assets/img/posts/container-without-docker-1/02.png)

루트가 `myroot`로 바뀌니까 그에 맞춰 `myroot/bin/sh`을 복사해 넣어줬다.
실행에 필요한 동적 라이브러리도 같이 복사해줘야 한다.
참고로 Go는 기본적으로 정적 컴파일이라서 같이 복사해줄 동적 라이브러리가 딱히 없다.
`linux-vdso.so.1`은 커널이 알아서 매핑해준다고 한다.
실행은 됐고, `ls`는 당연히 실패한다.

![](/assets/img/posts/container-without-docker-1/03.png)

필요한 걸 넣어주고 실행했다.
원래 호스트에서 실행한 것과 차이가 확실히 보인다.
이번에는 `ps`로 실행 중인 프로세스를 확인해보자.

![](/assets/img/posts/container-without-docker-1/04.png)

`ps`를 하기까지는 `mount`도 필요하고 `mkdir`도 필요했다. 사실 호스트에서 만들어도 되긴 하지만..
`ps` 결과는 다르게 보인다.
`/proc`은 디스크에 실제로 존재하지 않고, 커널이 메모리에서 쓰는 정보들이다.

![](/assets/img/posts/container-without-docker-1/05.png)

`docker export`로 nginx 이미지를 가져와서 실행해봤다.
`chroot`도 잘 먹혔고, nginx를 실행하니 로컬에서 80포트로 웹서버가 조회됐다.
종료하니까 웹서버도 접근이 안 됐다.

그런데 `chroot`는 사용자 프로세스를 속이는 것이다.
"너의 루트는 여기야~" 하고 속이는 방식이라, 실제로 루트가 바뀐 건 아니다.
그래서 탈옥을 할 수 있다.

```go
package main

import (
	"os"
	"syscall"
)

func main() {
	// 탈출용 임시 디렉토리 생성
	os.Mkdir("breakout", 0755)

    // 현재 디렉토리를 가리키는 핸들러 생성
	dir, _ := os.Open(".")
	defer dir.Close()

	// 임시 디렉토리로 chroot를 수행하여 현재 디렉토리 핸들(dir)을 fakeroot보다 밖에 위치 시킴
	syscall.Chroot("breakout")

	// fake root 밖으로 이동 후 상위 디렉토리로 이동
	dir.Chdir()
	for i := 0; i < 100; i++ {
		syscall.Chdir("..")
	}

	// 실제 시스템의 최상위 루트로 chroot해서 탈옥
	syscall.Chroot(".")
	
	// 탈출 성공 후 실제 호스트의 셸 실행
	syscall.Exec("/bin/sh", []string{"sh"}, os.Environ())
}
```

![](/assets/img/posts/container-without-docker-1/06.png)

`chroot`는 그저 fake root를 알려주는 것뿐이라서 실제로 파일시스템이 바뀌지는 않는다.
그래서 이런 편법으로 탈옥이 가능하다.
그래도 `chroot`는 sudo 권한이 있어야 하므로, 권한 제한으로 1차적으로 막을 수 있다.

그리고 `pivot_root`라는 시스템콜은 진짜로 파일시스템을 바꾸는 기능이다.

루트 파일시스템은 루트 디렉토리에 마운트되는 시스템이다.
시스템 바이너리, 장치, 설정, 라이브러리 같은 것들이 들어 있다.
그래서 이 루트 파일시스템을 피봇하면 호스트가 망가져 버린다.

호스트에 영향을 주지 않으면서 루트 파일시스템을 바꿀 수만 있다면, `chroot`보다 훨씬 안전하고 확실하게 루트를 바꿔서 격리할 수 있다.

마운트 개념이 뭐냐면, 어떤 파일시스템을 특정 위치에 갖다 붙이는 거라고 생각하면 된다.
USB를 꽂으면 `/dev/sdb1` 같은 장치로 잡히고, 이걸 `/media/usb` 같은 디렉토리에 마운트하는 식이다.
루트 파일시스템조차도 루트 디렉토리(`/`)에 마운트된 것이다.

그럼 이 마운트 정보만 호스트와 컨테이너가 공유하지 않으면, 즉 호스트의 마운트 정보와 컨테이너의 마운트 포인트 정보를 따로 유지할 수 있으면 컨테이너의 루트 파일시스템 마운트만 바꿔버릴 수 있다.

그래서 나온 개념이 **마운트 네임스페이스**다.
네임스페이스는 격리를 위한 개념이고, 마운트 네임스페이스는 마운트 정보를 격리하는 거라고 생각하면 된다.

![](/assets/img/posts/container-without-docker-1/07.png)

먼저 호스트에서 `unshare --mount /bin/sh`으로 마운트 포인트를 공유하지 않는 셸을 하나 실행했다.
처음에는 부모의 마운트 포인트 정보를 복사해 오므로 같은 파일시스템을 가진다.
새 디렉토리를 만들고(디렉토리 생성까지는 공유된다) 마운트를 실행하면, 마운트한 셸에서만 마운트 정보가 보이고 다른 곳에서는 안 보인다.
그 안에 파일을 만들어도 마운트한 셸에서만 디렉토리 아래에 파일들이 들어온 게 보인다.

![](/assets/img/posts/container-without-docker-1/08.png)

기존 루트 파일시스템을 마운트할 `put_old`를 만들고, `new_root`로 이동해서 `pivot_root`를 실행했다.
(루트 파일시스템이 바뀌기 때문에 미리 여기로 가 있어야 한다.)
`ls /`로 확인하면 `new_root`에 있던 파일들이 보이고, `ls put_old`를 하면 진짜 호스트 루트가 보인다.
탈옥을 시도해도 루트가 바뀌지 않는다. 실제로 루트 파일시스템이 바뀐 것이다.

![](/assets/img/posts/container-without-docker-1/09.png)

궁금해서 루트 파일시스템에 들어가지 않은 상태에서 `pivot_root`를 해봤는데, `cd`로 움직일 수가 없었다.
꼭 마운트를 격리해서 수행하자. **호스트에 영향이 가지 않도록.**

컨테이너 파일시스템을 정리하자면, 프로세스 실행에 필요한 것만 패키징해서 그 경로를 프로세스 실행 환경의 루트 파일시스템으로 격리해 버리는 것이다.
그 방법으로는 `chroot`와 `pivot_root`가 있다.
