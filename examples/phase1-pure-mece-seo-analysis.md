# 타임스프레드 SEO 의사결정 문서

---

## 테마 1. 크롤링 낭비 제거

> Google이 쓸데없는 곳에 크롤링 예산을 쓰고 있다. 안 써도 될 곳의 예산을 줄인다.

### 1-1. Canonical URL에 Query String 포함

**문제**: `MetaTagDocument.tsx`에서 `router.asPath`를 그대로 canonical에 사용. `/explore?tab=all&page=1&sort=popular` 같은 URL이 canonical이 되어, query 조합만큼 중복 canonical 발생.

**원인**: `router.asPath`가 query string을 포함하는데, 분리 없이 canonical에 전달.

**영향**: 같은 콘텐츠에 대해 중복 canonical → 페이지 권위(PageRank) 분산 + 크롤링 예산 낭비. "크롤됨-미색인" 36건 중 일부가 이 중복 때문일 가능성.

**선택지**:
| 방식 | 내용 |
|------|------|
| A. query string 제거 | `router.asPath.split('?')[0]` |
| B. 페이지별 명시적 canonical | 각 페이지에서 NextSeo로 직접 지정 |

**결정**: A안 — 1줄 수정으로 전체 적용. B안은 누락 위험.

**공수**: 0.5일 / **비용**: $0

---

### 1-2. JS 파일 캐시 헤더 누락 + TinyMCE 전역 로드

**문제**: 크롤링 요청의 **20.86%가 JS 다운로드**. Next.js 번들은 ~12개뿐인데 비율이 높음.

**원인 (2가지)**:

**(1) JS 파일에 `no-cache, private` 헤더**

```javascript
// next.config.js 현재 설정
{ source: '/:path*', headers: [{ key: 'Cache-Control', value: 'no-cache, private' }] }
{ source: '/:all*(svg|jpg|jpeg|png|gif|woff|woff2)', headers: [...max-age=31536000...] }
```

이미지/폰트만 1년 캐시. JS 파일은 패턴에 미포함 → `no-cache, private` 적용.

Google WRS는 "may ignore" 캐시 헤더라고 하지만 "always ignore"가 아님:
> "Googlebot caches aggressively... WRS **may** ignore caching headers."
> — [JavaScript SEO Basics](https://developers.google.com/search/docs/crawling-indexing/javascript/javascript-seo-basics)

**(2) TinyMCE 전역 로드**
- `_document.tsx`에서 `tinymce.min.js` (723KB) + 31개 플러그인 전역 로드
- 실제 사용 페이지: `/inapp/challenge/write` **1곳뿐**
- Googlebot이 모든 페이지에서 32개 JS 파일 발견

```
TinyMCE: 1(본체) + 31(plugins) = 32개
Next.js 번들: ~12개
합계: ~44개 JS 리소스
```

**선택지**:
| 방식 | 내용 | 공수 |
|------|------|------|
| A. JS 캐시 헤더만 수정 | glob에 `.js` 추가 | 0.5일 |
| B. TinyMCE lazy load만 | 해당 페이지에서만 로드 | 1일 |
| C. 둘 다 | A + B | 1.5일 |

**결정**: C안 — 둘 다 간단한 수정. A만으로는 TinyMCE 32개가 여전히 불필요 로드, B만으로는 Next.js 번들도 `no-cache`.

**공수**: 1.5일 / **비용**: $0

---

### 1-3. 404 응답 비율 3.23%

**문제**: 크롤링 응답 중 404가 3.23%. 존재하지 않는 페이지에 크롤링 예산 낭비. 301도 8.89%로 높아 리다이렉트 체인 가능성.

**원인**: 삭제된 콘텐츠 URL이 외부 링크나 sitemap에 잔존.

**선택지**:
| 방식 | 내용 |
|------|------|
| A. 410 반환 | 영구 삭제 명시 → Google이 빠르게 인덱스 제거 |
| B. 관련 페이지로 301 리다이렉트 | 페이지 권위 일부 보존 |
| C. sitemap에서 404 URL 제거 | 재크롤링 방지 |

**결정**: B + C 병행 — 410은 권위를 완전히 버림. 리다이렉트로 권위 보존 + sitemap 정리.

**공수**: 1~2일 / **비용**: $0

---

### 테마 1 합계

| 작업 | 공수 |
|------|------|
| 1-1. Canonical query string 제거 | 0.5일 |
| 1-2. JS 캐시 헤더 + TinyMCE | 1.5일 |
| 1-3. 404 정리 + 리다이렉트 | 1~2일 |
| **합계** | **3~4일** |

### 테마 1 모니터링

| 지표 | 현재 | 목표 | 확인 위치 |
|------|------|------|----------|
| JS 크롤링 비율 | 20.86% | **10% 이하** | Crawl Stats > 파일 형식 |
| 404 비율 | 3.23% | **1% 이하** | Crawl Stats > 응답 |
| 200 비율 | 86.68% | **95% 이상** | Crawl Stats > 응답 |
| 크롤됨-미색인 | 36건 | 감소 | Coverage > 제외 |
| 중복 페이지 수 | 확인 필요 | 감소 | Coverage > "중복, 표준 미선택" |

---

## 테마 2. 성능 및 인덱싱 개선

> Google에 더 빠르게 응답하고, 더 잘 인덱싱되게 한다.

### 2-1. 서버 응답 속도 느림 (TTFB 500~1000ms)

**문제**: Crawl Stats 실측 평균 응답 시간 500~1000ms. 2026.03 기준 700~1000ms로 악화 추세.

**원인**: 모든 SSR 페이지에 `Cache-Control: no-cache, private` → CloudFront를 쓰지만 HTML 캐시 미적용 → 매 요청이 origin 도달.

**영향**: Core Web Vitals LCP 악화 → Google 랭킹 시그널에 영향.

참고: 속도 개선이 크롤링 **양** 증가를 보장하지는 않음 (부록 "Crawl Budget 개념" 참고).

**선택지**:
| 방식 | 개발 공수 | 실시간성 | 비용 |
|------|----------|---------|------|
| A. ISR (getStaticProps + revalidate) | 페이지별 전환 (중) | revalidate 주기 | $0 |
| B. CF 캐싱 + TTL (s-maxage) | 헤더 변경 (낮) | TTL 주기 | $0 |
| C. CF 캐싱 + CMS webhook invalidation | Lambda 개발 (중) | 즉시 반영 | $0 |

**결정**: B안 (TTL 기반) — 필요 시 C안으로 확장.

**근거**:
- 코드 변경 없이 next.config.js 헤더만 변경
- ISR(A안)은 각 페이지를 getStaticProps로 전환해야 해서 공수 큼
- s-maxage=300 (5분) 시작 권장. 짧게 잡아도 부작용 없음
- 현재 크롤링 일 ~60건 / 6,591 URL이면 같은 페이지를 5분 내 재방문할 확률이 사실상 0이므로 s-maxage 길이가 크롤링에 미치는 영향 없음
- 나중에 실시간성 필요하면 C안(webhook + Lambda, 3~4일) 추가

**CF 비용**:
- 프리 티어: 1TB/월 전송 + 1,000만 요청 무료 ([CloudFront Pricing](https://aws.amazon.com/cloudfront/pricing/))
- 캐싱 켜도 추가 비용 없음 (전송량/요청 수 동일)
- Invalidation: 월 1,000 경로 무료 ([Invalidation Pricing](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/PayingForInvalidation.html))

**공수**: 0.5일 / **비용**: $0

---

### 2-2. "크롤됨 - 색인 안 됨" 36건

**문제**: Google이 크롤링은 했으나 인덱싱을 거부한 URL 36건.

**원인 (추정)**: 콘텐츠 품질/중복/thin content. 1-1의 canonical 중복이 원인일 가능성. 구체적 URL은 Search Console에서 확인 필요.

**결정**: URL 목록 확인 후 개별 대응 + canonical 수정(1-1) 후 자연 해소 대기 병행.

**공수**: URL 확인 0.5일 + 원인별 조치 별도

---

### 2-3. 기타 테크니컬 SEO

**잘 되어 있는 부분**:
- 환경별 noindex/nofollow 제어
- 구조화 데이터 (OrganizationJsonLd, BreadcrumbJsonLd, EventJsonLd, ItemListJsonLd)
- robots.txt 차단 (/inapp/, /api/, /err/)
- OG 이미지 (1200x630)
- Naver 웹마스터 인증
- NOINDEX_PATHS 비공개 페이지 차단
- 정적 에셋 1년 캐싱
- SSR + React Query dehydrate

**개선 항목**:

| 항목 | 현재 | 결정 | 공수 |
|------|------|------|------|
| Sitemap | CDN 외부 제공, 갱신 불명 | next-sitemap으로 동적 생성 | 2~3일 |
| viewport | `user-scalable=no` | `user-scalable=yes` | 0.5일 |
| Twitter Card | `summary` | `summary_large_image` | 0.5일 |
| lang 속성 | 미확인 | `<html lang="ko">` 확인 | 0.5일 |
| Google Search Console | Naver만 인증 | GSC 인증 추가 | 0.5일 |

---

### 테마 2 합계

| 작업 | 공수 |
|------|------|
| 2-1. CF 캐싱 (TTL 기반) | 0.5일 |
| 2-2. 미색인 URL 확인 | 0.5일+ |
| 2-3. viewport, Twitter Card, lang, GSC | 2일 |
| 2-3. 동적 sitemap | 2~3일 |
| **합계** | **5~6일** |

### 테마 2 모니터링

| 지표 | 현재 | 목표 | 확인 위치 |
|------|------|------|----------|
| 평균 응답 시간 | 500~1000ms | **50ms 이하** | Crawl Stats > 응답 시간 |
| Core Web Vitals LCP | 확인 필요 | 개선 | Search Console > Core Web Vitals |
| 크롤됨-미색인 | 36건 | 감소 | Coverage > 제외 |
| 모바일 사용성 | 확인 필요 | 통과 | Search Console > 모바일 사용성 |
| 검색 노출수/클릭수 | 확인 필요 | 증가 추세 | Search Console > 실적 |

---

## 테마 3. 아키텍처 결정 — Astro 전환

> 현시점 불필요. 조건부 재검토.

### 문제
Next.js 런타임 ~135KB(gzip)가 모든 페이지에 전송됨. Astro 전환 시 JS 0KB 가능.

### 이론적 이점
- JS 없는 페이지 → 구글봇 렌더링 큐 진입 방지
- 크롤링 예산 중 JS 리소스 비용 절감
- 저사양 모바일 hydration 비용 제거

### 결정: 현시점 불필요

### 근거

**(1) 사이트 규모가 크롤링 예산 걱정 범위 밖**
> "crawl budget is not something most publishers have to worry about... if a site has fewer than a few thousand URLs, most of the time it will be crawled efficiently."
> — [Crawl Budget Management](https://developers.google.com/crawling/docs/crawl-budget)

현재 ~6,500 URL. 관리 기준(1만+ 매일 변경 또는 100만+)에 미달.

**(2) JS 크롤링 비용이 실제로 작음**

Next.js는 라우트 단위로 번들 생성. URL 수와 번들 수는 무관:

```
공통 번들 (framework, main, webpack): ~3개 → 전 페이지 동일 URL 참조
페이지별 번들: ~8개 (라우트 수 = 번들 수)
합계: ~12개 고유 JS 파일
```

> "If a common resource is reused on multiple pages, **reference the resource from the same URL in each page**, so that Google can cache and reuse the same resource."
> — [Large Site Crawl Budget Management](https://developers.google.com/search/docs/crawling-indexing/large-site-managing-crawl-budget)

URL이 6,591개여도 JS 다운로드는 ~12개.

**(3) WRS가 JS를 최대 30일 캐싱**
> "WRS caches everything for up to 30 days, which helps preserve the site's crawl budget."
> — [Crawling December: Resources](https://developers.google.com/search/blog/2024/12/crawling-december-resources)

배포 시에만 content fingerprinting으로 해시 변경 → ~12개 재다운로드 (1회성).
> "content fingerprinting avoids this problem by making a fingerprint of the content part of the filename, like main.2bb85551.js"
> — [JavaScript SEO Basics](https://developers.google.com/search/docs/crawling-indexing/javascript/javascript-seo-basics)

**(4) 크롤링 병목 없음**
- "발견됨 - 미색인": 2건 → 크롤링 정상
- SSR로 완성된 HTML이 전송되므로 렌더링 큐 결과와 동일

**(5) 전환 비용 대비 효과 불명확**
- Astro 전환: 수 개월 (Styled Components 비호환, 상태관리 재설계, 웹뷰 통합 재구현)
- CF 캐싱(테마 2): 0.5일

### 재검토 트리거 조건

| 조건 | 기준값 |
|------|--------|
| 고유 URL 수 | **1만 건 이상** |
| "발견됨 - 미색인" | **50건 이상** 누적 |
| Core Web Vitals 모바일 LCP | **4초 이상** 지속 |
| JS 크롤링 비율 | **30% 이상** |

---

## 부록 A. Crawl Budget 개념 정리

출처: [Crawl Budget Management](https://developers.google.com/crawling/docs/crawl-budget)

### Crawl Budget = min(Capacity, Demand)

> "Taking crawl capacity and crawl demand together, Google defines a site's crawl budget as the set of URLs that Google **can** and **wants** to crawl."

두 요소를 더하는 것이 아니라, **낮은 쪽이 실제 크롤링 양을 결정.**

> "Even if the crawl capacity limit isn't reached, **if crawl demand is low, Google will crawl your site less.**"

```
현재 타임스프레드:
  Capacity: 충분 (5XX 에러 0.03%, 서버 병목 아님)
  Demand: 일 ~60건
  → 실제 크롤링 = ~60건 (Demand에 의해 결정)
```

### Crawl Capacity Limit — Google이 크롤링 **할 수 있는** 양

> "Google's crawlers calculate a *crawl capacity limit*, which is the maximum number of simultaneous parallel connections"

서버 응답 속도/안정성에 의해 결정. CF 캐싱으로 천장을 올릴 수 있지만, 현재 Capacity 병목이 아니므로 크롤링 양 증가는 보장 안 됨.

### Crawl Demand — Google이 크롤링 **하고 싶은** 양

> "Each crawler has its own 'demand' when it comes to crawling the web."

3가지 결정 요소:

**1. Perceived inventory** — 가장 직접 통제 가능
> "Without guidance from you, Google tries to crawl all or most of the URLs that it knows about on your site."
> This is "**the factor that you can positively control the most.**"

**2. Popularity**
> "URLs that are more popular on the Internet tend to be crawled more often to keep them fresher."

**3. Staleness**
> "Our systems want to recrawl documents frequently enough to pick up any changes."

크롤링 목적 중 "새로고침" 59.10% = Google이 기존 페이지 변경 여부를 확인하기 위해 재방문하는 비율.

### Demand를 높이려면 (Google 공식 기반)

| 방법 | Demand 요소 | 구체적 방법 |
|------|-----------|-----------|
| 콘텐츠 추가 | Perceived inventory | 캘린더/이벤트 수 증가 → 새 고유 URL |
| 중복 URL 정리 | Perceived inventory | canonical 수정, query string 중복 제거 |
| 불필요 URL 차단 | Perceived inventory | robots.txt |
| sitemap 관리 | Perceived inventory | 동적 생성, `<lastmod>` 전달 |
| 404 정리 | Perceived inventory | 리다이렉트 또는 410 |
| 외부 백링크 | Popularity | 마케팅/PR (비개발) |
| 콘텐츠 갱신 | Staleness | 이미 하고 있음 |

### 내부 링크의 역할

> "Every page you care about should have a link from at least one other page on your site."
> — [SEO Link Best Practices](https://developers.google.com/search/docs/crawling-indexing/links-crawlable)

> "Other pages are discovered when Google extracts a link from a known page to a new page."
> — [How Search Works](https://developers.google.com/search/docs/fundamentals/how-search-works)

내부 링크는 **새 URL 발견 경로**이지, Crawl Demand를 직접 높이는 요소는 아님. Crawl Demand 공식 문서에서 링크는 요소로 언급되지 않음. 타임스프레드는 발견됨-미색인 2건뿐이므로 URL 발견은 정상.

---

## 부록 B. 크롤링 데이터 실측 요약 (2025.12 ~ 2026.03)

| 지표 | 값 |
|------|-----|
| 총 크롤링 URL | 6,591건 |
| 일평균 크롤링 요청 | ~60건 (최대 895건) |
| Googlebot 스마트폰 비율 | 50.52% (모바일 퍼스트) |
| JS 크롤링 비율 | 20.86% |
| 크롤링 목적 새로고침 | 59.10% |
| 200 응답 비율 | 86.68% |
| 404 비율 | 3.23% |
| 301 비율 | 8.89% |
| 평균 응답 시간 | 500~1000ms |
| 크롤됨-미색인 | 36건 |
| 발견됨-미색인 | 2건 |

---

## 부록 C. CloudFront 비용 상세

### 페이지 캐싱 추가 비용: $0

| 항목 | 무료 한도 |
|------|----------|
| 데이터 전송 | 1TB/월 |
| HTTP/HTTPS 요청 | 1,000만 건/월 |

캐싱 여부와 무관하게 전송량/요청 수 동일 → 추가 비용 없음.
프리 티어 초과 시 한국 리전: $0.120/GB (1~10TB 구간).
출처: [CloudFront Pricing](https://aws.amazon.com/cloudfront/pricing/)

### Invalidation 비용: $0

| 항목 | 비용 |
|------|------|
| 월 1,000 경로까지 | 무료 |
| 초과 시 | $0.005/경로 |
| 와일드카드 (`/*`) | 1 경로로 계산 |

예상 월 ~124 경로 → 무료 한도 내.
출처: [Invalidation Pricing](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/PayingForInvalidation.html)

### CMS webhook invalidation (필요 시)

```
[CMS 수정] → [Webhook] → [Lambda] → [CF CreateInvalidation API]
```

| 항목 | 공수 |
|------|------|
| Lambda 함수 | 1일 |
| API Gateway | 0.5일 |
| CMS webhook 설정 | 0.5일 |
| 테스트 | 1일 |
| **합계** | **3~4일** |

현재는 TTL 기반(s-maxage=300)으로 충분. 실시간성 필요 시 확장.

---

## 실행 로드맵

### 🔴 긴급 (1-2주) — 총 2.5일

| 작업 | 테마 | 공수 |
|------|------|------|
| Canonical URL query string 제거 | 1-1 | 0.5일 |
| JS 캐시 헤더 수정 + TinyMCE lazy load | 1-2 | 1.5일 |
| "크롤됨-미색인" 36건 URL 확인 | 2-2 | 0.5일 |

### 🟠 높음 (2-4주) — 총 4~5일

| 작업 | 테마 | 공수 |
|------|------|------|
| CF 페이지 캐싱 적용 (s-maxage=300) | 2-1 | 0.5일 |
| 404 URL 정리 + 리다이렉트 | 1-3 | 1~2일 |
| viewport, Twitter Card, lang, GSC 인증 | 2-3 | 2일 |

### 🟡 중간 (1-2개월)

| 작업 | 테마 | 공수 |
|------|------|------|
| 동적 sitemap 생성 (next-sitemap) | 2-3 | 2~3일 |
| 301 리다이렉트 체인 정리 | 1-3 | 1일 |
| Crawl Stats before/after 비교 | 전체 | - |

### 🟢 장기 (조건부)

| 작업 | 테마 | 조건 |
|------|------|------|
| Astro 도입 검토 | 3 | URL 1만+, 미색인 50건+, LCP 4초+ |
| CMS webhook invalidation | 2-1 | TTL 방식 불충분 시 |
