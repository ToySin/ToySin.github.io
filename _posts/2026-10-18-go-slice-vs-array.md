---
title: "[Go] Slice와 Array 차이 이해하기"
date: 2026-10-18 12:00:00 +0900
categories: [Language, Go]
tags: [go, slice, array, escape-analysis, memory]
---

메모리 공부를 하다가 문득 궁금해졌다. **Go에서의 Slice와 Array는 어떻게 다른가?**

```go
package main

import "unsafe"

func main() {
	numSlice1 := make([]int, 2, 4)
	numSlice2 := make([]int, 2)
	numSlice3 := []int{1, 2}
	numSlice4 := make([]int, 0, 256)
	slices := [][]int{numSlice1, numSlice2, numSlice3, numSlice4}

	numArray := [2]int{1, 2}

	println("=== 배열 ===")
	println("변수 주소:", &numArray)
	println("첫 원소 주소:", &numArray[0])
	println("배열 크기:", unsafe.Sizeof(numArray), "bytes")

	println("\n=== 슬라이스 ===")
	println("슬라이스0 변수 주소:", &numSlice1)
	println("슬라이스1 변수 주소:", &numSlice2)
	println("슬라이스2 변수 주소:", &numSlice3)
	println("슬라이스3 변수 주소:", &numSlice4)

	for i, s := range slices {
		println("\n=== 슬라이스", i, "===")
		println("슬라이스", i, "헤더 크기:", unsafe.Sizeof(s), "bytes")
		println("슬라이스", i, "backing array 시작 주소:", unsafe.SliceData(s))
		println("슬라이스", i, "len:", len(s), "cap:", cap(s))
		println("슬라이스", i, "backing array 크기", unsafe.Sizeof(s[0])*uintptr(cap(s)), "bytes")
	}
}
```

가장 큰 차이점은 slice는 대괄호로만 표현되고, array는 대괄호 안에 배열의 크기를 함께 선언한다는 것이다. array는 익숙한 그 녀석이다. 연속된 메모리에 값을 저장하고 있는 자료형. **slice는 사실 `array`, `len`, `cap`을 들고 있는 구조체다.**

![array와 slice 헤더](/assets/img/posts/go-slice-vs-array/slice-header.svg){: w="960" h="390" }
_array는 값을 그 자리에, slice는 backing array를 가리키는 헤더를 들고 있다_

> [src/runtime/slice.go - The Go Programming Language](https://go.dev/src/runtime/slice.go)

이 특성을 알고 탐구를 시작하자. 이렇게 할당되었을 때 각 변수의 메모리 할당 상태를 알아보자.

```
=== 배열 ===
변수 주소: 0x7eea3da9a600
첫 원소 주소: 0x7eea3da9a600
배열 크기: 16 bytes

=== 슬라이스 ===
슬라이스0 변수 주소: 0x7eea3da9aea8
슬라이스1 변수 주소: 0x7eea3da9ae90
슬라이스2 변수 주소: 0x7eea3da9ae78
슬라이스3 변수 주소: 0x7eea3da9ae60

=== 슬라이스 0 ===
슬라이스 0 헤더 크기: 24 bytes
슬라이스 0 backing array 시작 주소: 0x7eea3da9ae38
슬라이스 0 len: 2 cap: 4
슬라이스 0 backing array 크기 32 bytes

=== 슬라이스 1 ===
슬라이스 1 헤더 크기: 24 bytes
슬라이스 1 backing array 시작 주소: 0x7eea3da9ae28
슬라이스 1 len: 2 cap: 2
슬라이스 1 backing array 크기 16 bytes

=== 슬라이스 2 ===
슬라이스 2 헤더 크기: 24 bytes
슬라이스 2 backing array 시작 주소: 0x7eea3da9ae18
슬라이스 2 len: 2 cap: 2
슬라이스 2 backing array 크기 16 bytes

=== 슬라이스 3 ===
슬라이스 3 헤더 크기: 24 bytes
슬라이스 3 backing array 시작 주소: 0x7eea3da9a618
슬라이스 3 len: 0 cap: 256
슬라이스 3 backing array 크기 2048 bytes
```

array와 cap이 2로 생성된 slice들은 그 크기가 16 bytes이다. slice는 `array`, `len`, `cap`을 담고 있었는데 각각 8 bytes씩(pointer, int, int) 차지하므로 slice 헤더 크기가 24 bytes이다. slice의 cap만큼 `slice.array`의 크기가 할당되어 있다.

slice 할당에서 재미있는 점은 slice0 → slice3으로 갈수록 slice 변수 주소가 큰 위치에서 -24씩 감소한다는 것이다. 즉 **스택 영역에 할당되는 것이 아닐까** 유추해볼 수 있다. slice0.array → slice1.array → slice2.array의 시작 주소도 backing array의 크기만큼 차이 난다. 크기가 큰 slice3.array(2048 bytes)는 떨어진 위치(배열 바로 뒤)에 있다.

두 가지 실험을 해보자.

1. `slice.cap` 이상으로 인자를 넣는 경우 메모리는 어떻게 되는가
2. 다른 함수에서 슬라이스를 생성하고 반환하는 경우 메모리는 어떻게 되는가

```go
package main

import "unsafe"

func main() {
	slice1 := []int{1, 2}
	slice2 := []int{3, 4}

	println("slice1 array addr:", unsafe.SliceData(slice1))
	println("slice2 array addr:", unsafe.SliceData(slice2))

	slice1 = append(slice1, slice2...)

	println("slice1 array addr:", unsafe.SliceData(slice1))
	println("slice2 array addr:", unsafe.SliceData(slice2))

	slice2 = append(slice2, slice2...)

	println("slice1 array addr:", unsafe.SliceData(slice1))
	println("slice2 array addr:", unsafe.SliceData(slice2))
}
```

```
slice1 array addr: 0xf88276e4708
slice2 array addr: 0xf88276e46f8
slice1 array addr: 0xf88276a20e0
slice2 array addr: 0xf88276e46f8
slice1 array addr: 0xf88276a20e0
slice2 array addr: 0xf88276a2100
```

`slice.cap` 이상으로 값을 넣었더니 어딘가 멀~리에 새로 할당되었다. 다른 slice도 array를 새로 할당할 때 붙여서 만들어졌다. 근데 이게 스택을 뜻하는 건 아니다.

![append 후 재할당](/assets/img/posts/go-slice-vs-array/append-realloc.svg){: w="960" h="410" }
_cap을 넘긴 append 전후 backing array 위치_

```go
func growslice(oldPtr unsafe.Pointer, newLen, oldCap, num int, et *_type) slice {
	// ...

	var p unsafe.Pointer
	if !et.Pointers() {
		p = mallocgc(capmem, nil, false)
		// The append() that calls growslice is going to overwrite from oldLen to newLen.
		// Only clear the part that will not be overwritten.
		// The reflect_growslice() that calls growslice will manually clear
		// the region not cleared here.
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		// Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
		p = mallocgc(capmem, et, true)
		if lenmem > 0 && writeBarrier.enabled {
			// Only shade the pointers in oldPtr since we know the destination slice p
			// only contains nil pointers because it has been cleared during alloc.
			//
			// It's safe to pass a type to this function as an optimization because
			// from and to only ever refer to memory representing whole values of
			// type et. See the comment on bulkBarrierPreWrite.
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(oldPtr), lenmem-et.Size_+et.PtrBytes, et)
		}
	}
	memmove(p, oldPtr, lenmem)

	return slice{p, newLen, newcap}
}
```

slice의 크기가 커져야 될 땐 `runtime.growslice()`를 호출한다. 함수 구현 마지막을 보면 `mallocgc()`를 호출한다. **slice가 grow될 땐 heap 영역에 array가 할당된다.** 주소가 붙어 있는 건 Go 할당기가 같은 크기 클래스(여기선 32 bytes) 객체를 한 span에서 차례로 나눠주기 때문이라고 한다.

Go 프로그램을 실행할 때 `-gcflags='-m=2'` 옵션을 붙이면 컴파일러 메시지를 볼 수 있다.

```go
package main

import "unsafe"

//go:noinline
func genSlice() []int {
	return []int{1, 2}
}

func main() {
	s1 := []int{1, 2}
	s2 := genSlice()
	s3 := make([]int, 100000)

	println("slice array addr:", unsafe.SliceData(s1))
	println("slice array addr:", unsafe.SliceData(s2))
	println("slice array addr:", unsafe.SliceData(s3))
}
```

```
//go:noinline 없이
# command-line-arguments
./slice-func.go:6:6: can inline genSlice with cost 5 as: func() []int { return []int{...} }
./slice-func.go:10:6: can inline main with cost 37 as: func() { s1 := []int{...}; s2 := genSlice(); s3 := make([]int, 100000); println("slicearray addr:", <node unsafe.SliceData>); println("slice array addr:", <node unsafe.SliceData>); println("slice array addr:", <node unsafe.SliceData>) }
./slice-func.go:12:16: inlining call to genSlice
./slice-func.go:7:14: []int{...} escapes to heap in genSlice:
./slice-func.go:7:14:   flow: ~r0 ← &{storage for []int{...}}:
./slice-func.go:7:14:     from []int{...} (spill) at ./slice-func.go:7:14
./slice-func.go:7:14:     from return []int{...} (return) at ./slice-func.go:7:2
./slice-func.go:7:14: []int{...} escapes to heap
./slice-func.go:13:12: make([]int, 100000) escapes to heap in main:
./slice-func.go:13:12:   flow: {heap} ← &{storage for make([]int, 100000)}:
./slice-func.go:13:12:     from make([]int, 100000) (too large for stack) at ./slice-func.go:13:12
./slice-func.go:11:13: []int{...} does not escape
./slice-func.go:12:16: []int{...} does not escape
./slice-func.go:13:12: make([]int, 100000) escapes to heap
slice array addr: 0x435082516710
slice array addr: 0x435082516700
slice array addr: 0x435082600000
```

```
//go:noinline 함께
# command-line-arguments
./slice-func.go:6:6: cannot inline genSlice: marked go:noinline
./slice-func.go:10:6: cannot inline main: function too complex: cost 89 exceeds budget 80
./slice-func.go:7:14: []int{...} escapes to heap in genSlice:
./slice-func.go:7:14:   flow: ~r0 ← &{storage for []int{...}}:
./slice-func.go:7:14:     from []int{...} (spill) at ./slice-func.go:7:14
./slice-func.go:7:14:     from return []int{...} (return) at ./slice-func.go:7:2
./slice-func.go:7:14: []int{...} escapes to heap
./slice-func.go:13:12: make([]int, 100000) escapes to heap in main:
./slice-func.go:13:12:   flow: {heap} ← &{storage for make([]int, 100000)}:
./slice-func.go:13:12:     from make([]int, 100000) (too large for stack) at ./slice-func.go:13:12
./slice-func.go:11:13: []int{...} does not escape
./slice-func.go:13:12: make([]int, 100000) escapes to heap
slice array addr: 0x5d165fbac708
slice array addr: 0x5d165fc14000
slice array addr: 0x5d165fc80000
```

inline은 잠깐 무시하고, **컴파일러는 변수가 함수 밖으로 나가는지 미리 확인할 수 있다.** 함수 밖으로 나가지 않는다면 스택에 할당할 수 있다. 100%는 아니다. 예시로 크기가 큰 값은 스택에 할당하지 않고 힙에 할당한다.

함수에 `//go:noinline` 옵션을 붙여두었다. 붙이지 않고 실행한 결과(위)는 함수를 `main` 안에 넣으면서 `does not escape`가 되었다. 옵션을 붙이고 실행하면(아래) inline을 명시적으로 하지 않으면서 `escapes to heap`이 나온 것을 확인할 수 있다.

단, 이는 표현식이 탈출하느냐 아니냐를 판단하는 요소이지 정확하게 스택, 힙 중 어디에 할당되었음을 나타내진 않는다.

마지막으로 재미있는 규칙을 하나 살펴보자.

```go
// nextslicecap computes the next appropriate slice length.
func nextslicecap(newLen, oldCap int) int {
	newcap := oldCap
	doublecap := newcap + newcap
	if newLen > doublecap {
		return newLen
	}

	const threshold = 256
	if oldCap < threshold {
		return doublecap
	}
	for {
		// Transition from growing 2x for small slices
		// to growing 1.25x for large slices. This formula
		// gives a smooth-ish transition between the two.
		newcap += (newcap + 3*threshold) >> 2

		// We need to check `newcap >= newLen` and whether `newcap` overflowed.
		// newLen is guaranteed to be larger than zero, hence
		// when newcap overflows then `uint(newcap) > uint(newLen)`.
		// This allows to check for both with the same comparison.
		if uint(newcap) >= uint(newLen) {
			break
		}
	}

	// Set newcap to the requested cap when
	// the newcap calculation overflowed.
	if newcap <= 0 {
		return newLen
	}
	return newcap
}
```

새 len과 원래 cap을 비교한다.

- 원래 cap을 두 배 해도 부족하면, 필요한 새 len을 새 cap으로 설정한다.
- 원래 cap이 256보다 작으면 원래 cap을 두 배 한 걸 새 cap으로 설정한다.
- 원래 cap이 256 이상이면 `newcap += (newcap + 3*threshold) >> 2`를 새 len 이상이 될 때까지 반복한다.

이 수식을 해석해보자면, 새 cap은 `newcap + newcap/4 + 192`이다. 기존 값에 25%를 더하고 192 상수를 더하는 건데, cap의 크기가 작을 때는 192의 영향이 크지만 점점 커지면 192의 영향이 줄어든다. **값이 커질수록 증가 비율이 1.25배에 수렴하게 된다.**

![cap 증가 비율](/assets/img/posts/go-slice-vs-array/cap-growth.svg){: w="960" h="410" }
_원래 cap에 따른 cap 증가 비율 (nextslicecap 공식 기준)_
