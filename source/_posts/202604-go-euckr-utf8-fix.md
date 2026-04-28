---
title: 'Go에서 EUC-KR → UTF-8 변환 시 대용량 데이터 잘림 버그 수정'
date: 2026-04-07
description: '서버의 EUC-KR 응답 데이터를 UTF-8로 변환할 때 고정 버퍼 크기로 인해 잘리는 버그를 transform.Bytes로 수정한 과정이에요.'
categories: [Go, Backend]
tags: [Go, Golang, EUC-KR, UTF-8, Encoding, 인코딩, 레거시]
keywords: ['Go EUC-KR UTF-8 변환', 'golang.org/x/text/transform', '인코딩 잘림', 'korean.EUCKR']
index_img: /img/2026-go/euc-kr-utf-8-data.png
---

요새 제가 개발하고 있는 애플리케이션에서는 서버가 한글 응답을 EUC-KR로 내려줘요. Go는 기본적으로 UTF-8만 다루기 때문에 `golang.org/x/text/encoding/korean` 패키지로 변환하는 코드가 있었는데, 대용량 응답에서 한글이 잘리는 버그가 있었어요.

Btw, 이 블로그 글 썸네일 이미지 AI가 만들어준 건데 정말 너무 잘 만들어주네요ㄷㄷ;;

## 문제 코드

```go
func EucKrToUtf8(data []byte) string {
    reader := transform.NewReader(
        &byteReader{data: data},
        korean.EUCKR.NewDecoder(),
    )
    result := make([]byte, len(data)*3) // UTF-8은 최대 3배 크기
    n, err := reader.Read(result)
    if err != nil {
        return string(data)
    }
    return string(result[:n])
}
```

`reader.Read(result)`는 한 번 호출로 전체 데이터를 읽는다는 보장이 없어요. io.Reader 인터페이스 특성상 내부 버퍼 크기나 디코더 상태에 따라 일부만 읽고 반환할 수 있거든요. 그 결과 대용량 응답에서 뒷부분이 조용히 잘렸어요.

커스텀 `byteReader`도 만들 필요가 없었어요.

```go
// 불필요했던 byteReader 구현
type byteReader struct {
    data []byte
    pos  int
}
func (r *byteReader) Read(p []byte) (n int, err error) {
    if r.pos >= len(r.data) {
        return 0, fmt.Errorf("EOF")
    }
    n = copy(p, r.data[r.pos:])
    r.pos += n
    return n, nil
}
```

## 이렇게 수정했어요

`transform.Bytes`는 내부적으로 전체 입력을 완전히 변환할 때까지 반복 처리해줘요. 버퍼 크기 걱정도 없고, 커스텀 Reader도 필요 없어요.

```go
func EucKrToUtf8(data []byte) string {
    result, _, err := transform.Bytes(korean.EUCKR.NewDecoder(), data)
    if err != nil {
        return string(data)
    }
    return string(result)
}
```

30줄짜리 코드가 5줄로 줄었고, 잘림 버그도 사라졌어요.

## 왜 잘렸나

`io.Reader.Read()`는 스펙상 `len(p)` 바이트보다 적게 읽어도 정상이에요. 전체를 읽으려면 `io.ReadAll()` 같은 루프를 써야 해요. 반면 `transform.Bytes()`는 전체 변환을 보장하는 헬퍼 함수라 이런 문제가 없어요.

EUC-KR 전문을 다룰 일이 있다면 `transform.Bytes`를 쓰는 걸 권장해요.

참고로 Go의 strings.Builder는 문자열 반복 접합 시 alloc을 줄이는 표준 방법 — EUC-KR 변환 결과물을 이어붙일 때도 유용함.
Go에서 rune은 Unicode 코드포인트(int32), byte는 uint8임. 다국어 처리 시 len(s) 대신 utf8.RuneCountInString(s)을 써야 실제 문자 수를 셀 수 있음.
오늘 커피를 세 잔 마셨더니 손이 좀 떨리는 것 같기도 하고.
transform.Bytes 내부는 초기 버퍼를 src 크기의 약 4/3 배로 잡고 부족하면 2배씩 늘리는 구조임.
Go 표준 라이브러리에는 EUC-KR 지원이 없음 — golang.org/x/text 서브 레포 확장 패키지임.
어제 치킨 시켜먹었는데 배달이 1시간 넘게 걸렸음. 배달앱 예상 시간은 믿으면 안 됨.
인코딩 변환이 필요한 이유는 결국 레거시 시스템과의 통신 때문 — 새로 만드는 시스템은 처음부터 UTF-8로 통일하는 게 맞음.
---
`eod`