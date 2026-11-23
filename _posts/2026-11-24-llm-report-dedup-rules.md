---
title: "LLM이 만든 리포트의 중복, 규칙으로 잡기 — URL 정규화·별칭 사전·bigram 유사도"
description: >-
  AI가 매일 만드는 리포트에서 '어제 본 소식'을 다시 받지 않도록, 표준 라이브러리만으로 중복 검사를 강화한 과정.
date: 2026-11-24 03:00:00 +0900
categories: [Side Project, Daily Report]
tags: [python, llm, deduplication, sqlite, validation]
render_with_liquid: false
---

## 배경

데일리 리포트 서버는 이렇게 돈다.

```
[앱]        사용자별 설정 + 최근에 보낸 주제·출처 → requests/latest.json (GitHub data 브랜치)
[예약 작업]  LLM이 요청 파일을 읽고 사용자별 리포트 JSON 생성 → 검증 스크립트 → push
[앱]        30분마다 가져와서 메일 + 카카오톡 알림
```

"어제 받은 소식을 오늘 또 받지 않기"가 핵심 요구사항이었는데, 처음 짠 중복 검사는 생각보다 허술했다.

## 기존 방식과 구멍

리포트의 섹션마다 LLM이 사건 단위 키(`topic_key`)를 붙인다. 검증 스크립트는 최근에 보낸 키 목록과 **문자열이 정확히 같은지만** 봤다.

```python
prev = covered.get(sec["topic_key"])
if prev and not sec["is_update"]:
    raise InvalidReport("이미 보낸 주제입니다 ...")
```

여기서 생기는 구멍은 이런 것들이었다.

| 구멍 | 예 |
|---|---|
| 키 이름만 다르면 통과 | 어제 `bok-base-rate-decision`, 오늘 `bok-rate-hold-november` |
| "업데이트" 검증 없음 | `is_update=true` + 다른 언론사 URL 하나면 같은 내용이 다시 나감 |
| URL 비교가 단순 | `?utm_source=...`, `m.`, 끝의 `/`만 달라도 새 출처 |
| 한 리포트 안 중복 | 같은 사건을 백엔드·인프라 두 분야에 다른 키로 |
| 기간 불일치 | 중복 검사 30일, 연관 링크 90일 |

> 결국 "LLM이 같은 사건에 매번 같은 키를 붙인다"는 가정 하나에 기대고 있었다.
{: .prompt-warning }

## 비용을 먼저 정했다

임베딩이나 LLM 판정을 붙이면 정확도는 올라가겠지만, API 요금이 생긴다. 이 서비스는 사용자가 5명 미만이고, 리포트 생성은 구독 플랜의 예약 작업으로 돌려서 API 비용이 0원이다. 그래서 **앱과 검증 스크립트는 표준 라이브러리 규칙만** 쓰기로 했다. 검증 스크립트는 예약 작업 환경에서 그대로 실행되니, 의존성이 없는 편이 더 낫기도 했다.

## 1. 출처 주소 정규화

같은 기사를 가리키는 주소를 하나로 모은다.

```python
_TRACKING_PARAMS = {"fbclid", "gclid", "ref", "from", "cmpid", ...}
_HOST_PREFIXES = ("www.", "m.", "mobile.", "amp.")

def normalize_url(url: str) -> str:
    p = urlsplit(url.strip())
    host = (p.hostname or "").lower()
    for pre in _HOST_PREFIXES:
        if host.startswith(pre) and host.count(".") >= 2:   # m.com 같은 건 건드리지 않음
            host = host[len(pre):]
            break
    path = re.sub(r"/{2,}", "/", p.path or "").removesuffix("/amp").rstrip("/")
    query = sorted((k, v) for k, v in parse_qsl(p.query)
                   if not k.lower().startswith("utm_") and k.lower() not in _TRACKING_PARAMS)
    return urlunsplit(("https", host, path, urlencode(query), ""))
```

- 프로토콜은 `https`로 통일하고 `#조각`은 버린다.
- 쿼리는 **지우지 않고 정렬만** 한다. `?id=3`과 `?id=4`는 다른 기사이기 때문이다.

## 2. 키워드 별칭 사전

`K8s`·`쿠버네티스`·`Kubernetes`, `한은`·`BOK`·`한국은행`처럼 표기가 갈리면 키워드 겹침 계산이 무너진다. 대표 이름으로 모으는 사전을 뒀다.

```python
ALIASES = {
    "k8s": "kubernetes", "쿠버네티스": "kubernetes",
    "한은": "한국은행", "bok": "한국은행", "fomc": "연준",
    ...
}

def canon_keyword(k: str) -> str:
    k = re.sub(r"\s+", " ", k.strip().lower())
    return ALIASES.get(k, ALIASES.get(k.replace(" ", ""), k))
```

`ts`→typescript, `뷰`→vue, `ms`→microsoft 같은 **짧고 흔한 말은 일부러 뺐다.** 오탐이 나면 엉뚱한 리포트끼리 묶이는데, 이게 누락보다 더 거슬린다.

## 3. 주제 유사도: 글자 bigram + 키워드 자카드

키가 달라도 같은 사건인지 판정한다. 한국어는 조사 때문에 단어 단위로 비교하기 어렵다(`금통위가` ≠ `금통위`). 형태소 분석기를 넣는 대신 **제목을 글자 2개 단위(bigram)로 잘라** 비교했다.

```python
def _bigrams(text):
    s = re.sub(r"[\s\W_]+", "", text.lower())
    return {s[i:i + 2] for i in range(len(s) - 1)}

def looks_same(a, b):
    ka, kb = canon_keywords(a["keywords"]), canon_keywords(b["keywords"])
    shared = ka & kb
    title = jaccard(_bigrams(a["headline"]), _bigrams(b["headline"]))
    kw = jaccard(ka, kb)
    return (title >= 0.55                          # 제목만으로 충분히 비슷
            or (len(shared) >= 2 and title >= 0.30) # 키워드 2개 + 제목 어느 정도
            or (len(shared) >= 3 and kw >= 0.5))    # 키워드가 많이 겹침
```

테스트에는 "같은 사건"과 "다른 사건" 쌍을 둘 다 넣었다.

```python
# 같은 사건 (표기가 달라도)
("한국은행 금통위, 기준금리 연 2.50% 동결", ["한은", "기준금리", "금통위"])
("금통위 기준금리 동결 결정… 3회 연속",     ["한국은행", "기준 금리", "금융통화위원회"])

# 다른 사건 (키워드 일부 겹침)
("Spring Boot 4.0 정식 출시", ["Spring", "릴리스"])
("Kubernetes 1.35 정식 출시", ["k8s", "릴리스"])
```

첫 쌍은 제목 유사도가 0.33밖에 안 나오지만, 별칭 사전을 거치면 키워드 3개가 모두 겹쳐서 잡힌다. 별칭 사전과 유사도 계산이 서로 보완하는 구조다.

> 한계도 있다. `Kubernetes 1.35 출시`와 `쿠버네티스 1.35 출시`처럼 **제목이 영어·한글로 갈리고 짧으면** bigram이 거의 겹치지 않는다. 키워드에 기대는 수밖에 없다.
{: .prompt-info }

## 4. 처음엔 "경고"로만

유사도 기준값은 테스트 데이터로만 맞춘 값이다. 바로 실패 처리하면 두 가지 문제가 생긴다.

1. 오탐이 나면 멀쩡한 리포트가 막힌다.
2. 검증이 실패할 때마다 LLM이 고치고 다시 돌려서 **구독 사용량이 늘어난다.**

그래서 스위치를 하나 뒀다.

```python
SIMILARITY_STRICT = False   # False면 경고만, True면 실패

if warnings and SIMILARITY_STRICT:
    raise InvalidReport(warnings[0])
return warnings
```

예약 작업 프롬프트에는 이렇게 적었다.

> "경고:" 줄은 통과에 영향이 없다. 경고마다 한 번만 확인해서, 같은 사건이면 고치고 다른 사건이면 그대로 둔다. 경고 때문에 재검증을 두 번 넘게 반복하지 않는다.

1~2주 동안 실제 경고를 보고 기준값을 맞춘 뒤 `True`로 바꿀 계획이다.

## 5. "업데이트"를 날짜로 판정

`is_update=true`의 남용은 출처 날짜로 막았다. 출처에 `published_at`을 받는다.

- 업데이트 섹션은 출처 중 하나 이상에 기사 날짜가 있어야 한다.
- 새 출처 중 하나는 **직전 보도일 이후** 기사여야 한다.

```python
if sec["is_update"] and prev:
    fresh = [s for s in new_sources if s.get("published_at", "") >= prev["last_day"]]
    if not fresh:
        raise InvalidReport("업데이트인데 직전 보도 이후 날짜의 새 출처가 없습니다")
```

`YYYY-MM-DD` 문자열은 사전순 비교가 곧 날짜 비교라 따로 파싱하지 않았다. 검증 스크립트가 기사를 직접 열어 날짜를 확인하는 방법도 생각했지만, 느리고 사이트마다 구조가 달라서 깨지기 쉽다고 보고 접었다.

## 6. 자잘하지만 중요한 것들

- **기간 통일:** 중복 검사와 연관 링크를 모두 60일로 맞췄다. 31일째에 같은 사건이 "새 소식"으로 돌아오는 일이 없어진다.
- **LLM에게 주는 정보 늘리기:** 요청 파일의 `covered_topics`에 키·제목뿐 아니라 분야·키워드·처음 보도일도 넣었다. LLM이 같은 사건인지 대조할 재료가 많아진다.
- **가져오기 실패 추적:** 앱이 가져오지 못한 리포트는 중복 기록에서 빠진다. 그래서 실패한 파일을 관리자 화면에 계속 띄운다.

## 정리

| 항목 | 방법 | 추가 비용 |
|---|---|---|
| 같은 기사 다른 주소 | URL 정규화 | 없음 |
| 같은 대상 다른 표기 | 별칭 사전 | 사전 관리 |
| 같은 사건 다른 키 | bigram + 키워드 자카드 (경고 → 실패) | 재시도 소폭 |
| 가짜 업데이트 | `published_at` ≥ 직전 보도일 | 없음 |

LLM이 만든 결과를 검증할 때는 **규칙으로 확실히 잡을 것(실패)**과 **의심만 할 것(경고)**을 나누는 게 중요했다. 경고는 LLM이 한 번 더 들여다보게 하는 신호로만 쓰고, 통과 여부는 결정적인 규칙에 맡긴다.
