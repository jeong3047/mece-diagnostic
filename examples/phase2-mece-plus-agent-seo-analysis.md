---
- 작성일: 2026년 3월 16일
- 분석 방법: MECE Issue Tree → seo-expert agent 병렬 코드 검증 (A/B/C 3 agent)
- 분석 범위: 전체 프론트엔드 코드 (백엔드/인프라 제외)
- 복잡도: Deep
---

## 결론 (피라미드 꼭대기)

**핵심 발견 3가지:**
1. **LCP 근본 원인**: 히어로 배너가 SSR에 포함되지 않아 홈페이지 LCP가 클라이언트 렌더링에 의존. TinyMCE 741KB 동기 blocking이 모든 페이지의 FCP/LCP를 악화.
2. **challenge 페이지 SEO 사각지대**: challenge/[id], challenge/home에 메타 태그/h1/구조화 데이터/에러 핸들링이 전면 부재. 검색 결과에서 챌린지별 구분 불가.
3. **크롤링 효율 저하**: NOINDEX 경로가 robots.txt에 미차단, 내부 링크가 router.push로만 구현되어 크롤러가 챌린지 상세/explore를 발견하지 못함.

## Day 1 Answer → 최종 결론 변화

- **Day 1**: "기존 seo-check.md 통합본이 대부분 커버. OG 세부 태그, /explore 접근성, 시멘틱 HTML 정도가 누락."
- **최종**: Day 1 Answer는 대체로 맞았으나, **새롭게 발견된 중요 이슈**가 있음:
  - 챌린지 카드 클릭이 router.push → 크롤러가 challenge/[id] 페이지를 내부 링크로 발견 불가 (B-6)
  - 이벤트/캘린더 상세의 useCalendarAuthReady 인증 gate가 LCP 이미지 렌더링을 차단 (C-1)
  - Radix UI CSS 788KB 전역 로드 (C-4)
  - OG 이미지 465~700KB 과대 (B-3)
- **차이 원인**: 기존 분석이 "코드에 있는 것"을 찾았다면, 이번 분석은 "코드에 없는 것"(누락)과 "런타임 동작"(인증 gate 뒤 렌더링 차단)까지 추적

---

## 요약

| 우선순위 | 테마 | 문제 | 원인 | 대응 | So What? |
| --- | --- | --- | --- | --- | --- |
| 🔴 긴급 | **1. 크롤링 낭비 제거** | NOINDEX↔robots.txt 불일치, TinyMCE 전역 로드, canonical query 오염 | robots.txt Disallow 미반영, TinyMCE async 없음, router.asPath 사용 | Disallow 추가, TinyMCE lazy load, canonical query 제거 | 크롤링 예산 20%+ 낭비 → 인덱싱 효율 저하 |
| 🔴 긴급 | **4. Core Web Vitals** | 히어로 배너 SSR 미포함, TinyMCE 741KB blocking, 모니터링 전무 | banners prefetch 누락, 동기 스크립트, web-vitals 미설치 | SSR prefetch, 스크립트 비동기화, web-vitals 설치 | LCP/FCP 직접 악화 → 검색 순위 하락 위험 |
| 🟠 높음 | **5. 콘텐츠 시그널** | challenge 메타 태그/h1/JSON-LD 전면 부재, 헤더 로고/챌린지 카드 크롤 불가 | NextSeo 미사용, router.push 단독 사용 | NextSeo 추가, next/link 래핑 | 챌린지 페이지가 검색에서 사실상 미노출 |
| 🟠 높음 | **2. 성능 및 인덱싱** | user-scalable=no, 보안 헤더 부재, JS 캐시 정책 | viewport 설정, next.config.js headers 미설정 | viewport 수정, 보안 헤더 추가 | 모바일 친화성 감점 + Lighthouse 점수 저하 |
| 🟡 중간 | **OG & Social** | og:locale/type/site_name 미설정, Twitter Card summary, OG 이미지 과대 | DefaultSeo 기본값 불충분 | OG 태그 보강, 이미지 압축 | 소셜 공유 품질 저하 |
| 🟢 장기 | **구조 개선** | Radix CSS 788KB, 사이트맵 동적 생성 미반영, 시멘틱 HTML | 아키텍처적 제약 | 점진적 개선 | 장기 기술 부채 |

---

## 테마 1. 크롤링 낭비 제거

### 1-1. Canonical URL에 Query String 포함

**문제**: `MetaTagDocument.tsx:16`에서 `router.asPath`를 그대로 canonical에 사용. query string 포함 canonical 발생.
**영향**: PageRank 분산 + 크롤링 예산 낭비.
**결정**: `router.asPath.split('?')[0]`으로 query string 제거.
**공수**: 0.5일

### 1-2. JS 캐시 헤더 + TinyMCE 전역 로드

**문제**: `next.config.js:45`에서 `/:path*`에 `no-cache, private` 적용. JS 파일 패턴 미포함. `_document.tsx:48`에서 TinyMCE 741KB를 async 없이 전역 동기 로드.
**영향**: JS 크롤링 비율 20.86%, 모든 페이지 FCP/LCP 차단.
**결정**: JS 캐시 헤더 추가 + TinyMCE를 사용 페이지(`/inapp/challenge/write`)에서만 동적 로드.
**공수**: 1.5일

### 1-3. NOINDEX_PATHS와 robots.txt Disallow 불일치

**문제**: `/auth/*`, `/roulette/*`, `/challenge/participation`, `/challenge/my/*`, `/alarm`이 NOINDEX_PATHS에 있지만 robots.txt Disallow에 없음.
**영향**: 크롤링 후 noindex 발견 → 크롤링 예산 + WRS 렌더링 큐 이중 낭비.
**결정**: `robots.txt.ts`의 `disallowPaths`에 해당 경로 추가. 단, `/challenge/home`과 `/challenge/[id]`(공개 페이지)는 제외.
**공수**: 15분

### 1-4. 404 정리

**문제**: 크롤링 응답 중 404가 3.23%.
**결정**: 따로 대응하지 않고 추이 확인.

### 테마 1 합계

| 작업 | 공수 |
| --- | --- |
| 1-1. Canonical query string 제거 | 0.5일 |
| 1-2. JS 캐시 헤더 + TinyMCE lazy load | 1.5일 |
| 1-3. robots.txt Disallow 추가 | 15분 |
| 1-4. 404 정리 | 0일 |
| **합계** | **2일** |

### 테마 1 모니터링

| 지표 | 현재 | 목표 | 확인 위치 |
| --- | --- | --- | --- |
| JS 크롤링 비율 | 20.86% | 10% 이하 | Crawl Stats > 파일 형식 |
| 404 비율 | 3.23% | 1% 이하 | Crawl Stats > 응답 |
| 크롤됨-미색인 | 36건 | 감소 | Coverage > 제외 |

---

## 테마 2. 성능 및 인덱싱 개선

### 2-1. 서버 응답 속도 (TTFB)

**문제**: 모든 SSR 페이지에 `Cache-Control: no-cache, private`.
**결정**: `s-maxage=300` TTL 기반 CF 캐싱.
**공수**: 0.5일

### 2-2. viewport user-scalable=no

**문제**: `MetaTagDocument.tsx:26`에서 `user-scalable=no`, `maximum-scale=1`. Google 모바일 친화성 감점, WCAG 2.1 위반.
**결정**: `user-scalable=yes`, `maximum-scale=5`로 변경.
**공수**: 0.5일

### 2-3. 보안 헤더 미설정

**문제**: `next.config.js` headers에 HSTS, X-Content-Type-Options, X-Frame-Options, Referrer-Policy 전면 미설정.
**영향**: Lighthouse Best Practices 감점. HSTS 없으면 HTTP→HTTPS 301 매 요청 발생.
**결정**: `next.config.js` headers에 보안 헤더 추가. X-Frame-Options는 `/calendar` 경로 예외 처리.
**공수**: 0.5일

### 2-4. /explore title 키워드 보강

**문제**: `explore/index.tsx:22` — `구독 캘린더 검색 | 타임스프레드` (18자).
**결정**: 키워드 보강 (예: `구독 캘린더 검색 - 인기 일정 찾기 | 타임스프레드`).
**공수**: 15분

### 테마 2 합계

| 작업 | 공수 |
| --- | --- |
| 2-1. CF 캐싱 (TTL) | 0.5일 |
| 2-2. viewport 수정 | 0.5일 |
| 2-3. 보안 헤더 추가 | 0.5일 |
| 2-4. /explore title 보강 | 15분 |
| **합계** | **1.5일** |

---

## 테마 3. Astro 전환 — 현시점 불필요

기존 seo-check.md 분석 유지. 사이트 규모 ~6,500 URL, 크롤링 병목 없음. 조건부 재검토.

---

## 테마 4. Core Web Vitals 최적화

### 4-1. 히어로 배너 SSR 미포함 🔴

**문제**: `index.tsx:35-53`의 `getServerSideProps`에서 `banners` 쿼리 미포함. `HomeSwiper.tsx:68-71`의 `<img>`에 `fetchpriority` 없음.
**영향**: 홈페이지 LCP 요소가 클라이언트 렌더링 후에야 표시.
**결정**: (1) getServerSideProps에 banners prefetch 추가 (2) 첫 번째 배너에 `fetchpriority="high"`.
**공수**: 1일

### 4-2. 렌더 블로킹 리소스

**문제**: `_document.tsx:47` font.css 동기 로드, `KakaoAccountConfig.tsx:7-11` Kakao SDK async 없음.
**결정**: font.css preload 전환, Kakao SDK async 추가.
**공수**: 0.5일

### 4-3. Preconnect / DNS-Prefetch 누락

**문제**: 외부 도메인 리소스 힌트 전무.
**결정**: `t1.kakaocdn.net`, `www.googletagmanager.com`, `media.timespread.co.kr`에 preconnect 추가.
**공수**: 0.5일

### 4-4. 이미지 최적화 (홈 + 상세)

**문제**: `HomeCalendarGridItem:55`, `HomeCalendarSwiperSlide:63`, `SubscriptionEventInfoDesktop:86`, `SubscriptionEventInfoMobile:80`, `SubscriptionCalendarHeader:53`, `SubscriptionCalendarCard:50`, `SubscriptionCalendarExploreCard:52` — 모두 native `<img>`.
**영향**: WebP 미적용, srcset 미생성, lazy loading 미적용.
**결정**: `next/image` 전환. LCP 후보에 `priority` 속성.
**공수**: 2~3일

### 4-5. 폰트 preload (CSS + woff2)

**문제**: (1) font.css 렌더 블로킹 (2) woff2가 CSS 파싱 후에야 발견되는 2단계 지연.
**결정**: Pretendard 400/700 woff2에 `<link rel="preload" as="font" type="font/woff2" crossorigin>` 추가.
**공수**: 0.5일

### 4-6. 스켈레톤 CLS

**문제**: (1) `HomeCalendarEventList.tsx:37-39` 로딩 중 높이 0 → 10개 삽입 시 CLS. (2) 캘린더 그리드 스켈레톤이 텍스트 영역 미포함.
**결정**: 이벤트 리스트 스켈레톤 추가, 그리드 스켈레톤 높이 보정.
**공수**: 1일

### 4-7. Radix UI CSS 788KB 전역 로드

**문제**: `_app.tsx:8`에서 전역 import. 66개 파일에서 사용되어 완전 제거 불가.
**결정**: 장기 과제. 사용 범위 측정 후 점진적 분리.
**공수**: 1일 (장기)

### 4-8. 인증 gate로 인한 상세 페이지 LCP 지연

**문제**: `ScreenSubscriptionEvent.tsx:26-28`, `ScreenSubscriptionCalendarDetail.tsx:108-110` — `useCalendarAuthReady` 완료 전 `return null`. SSR 데이터가 있어도 클라이언트에서 인증 확인까지 이미지 미렌더링.
**영향**: 이벤트/캘린더 상세 LCP 악화.
**결정**: 비로그인 사용자는 인증 gate 없이 SSR 데이터 즉시 렌더링. 인터랙션 버튼만 인증 후 활성화.
**공수**: 1일

### 4-9. web-vitals 설치 + INP 모니터링

**문제**: `web-vitals` 패키지 미설치. CWV 실측 데이터 수집 불가.
**결정**: web-vitals 설치, `_app.tsx`에서 `reportWebVitals` export.
**공수**: 0.5일

### 테마 4 합계

| 작업 | 공수 |
| --- | --- |
| 4-1. 히어로 배너 SSR prefetch 🔴 | 1일 |
| 4-2. 렌더 블로킹 리소스 | 0.5일 |
| 4-3. Preconnect | 0.5일 |
| 4-4. 이미지 next/image 전환 | 2~3일 |
| 4-5. 폰트 preload | 0.5일 |
| 4-6. 스켈레톤 CLS | 1일 |
| 4-7. Radix CSS (장기) | 1일 |
| 4-8. 인증 gate LCP | 1일 |
| 4-9. web-vitals + INP 모니터링 | 0.5일 |
| **합계** | **8.5~9.5일** |

### 테마 4 모니터링

| 지표 | 현재 | 목표 | 확인 위치 |
| --- | --- | --- | --- |
| LCP (모바일) | 미측정 | 2.5초 이하 | PageSpeed Insights |
| FCP (모바일) | 미측정 | 1.8초 이하 | PageSpeed Insights |
| CLS | 미측정 | 0.1 이하 | PageSpeed Insights |
| INP | 미측정 | 200ms 이하 | PageSpeed Insights |

---

## 테마 5. 콘텐츠 시그널 강화

### 5-1. 구조화 데이터 보강

**현재**: OrganizationJsonLd(홈), BreadcrumbJsonLd+EventJsonLd(이벤트/캘린더 상세), ItemListJsonLd(카테고리).
**누락**: (1) Organization logo 가로형(1200x630) → 정사각형 권장 (2) WebSite+SearchAction 스키마 없음 (3) challenge 페이지 JSON-LD 없음 (4) sameAs에 SNS 없음.
**공수**: 1.5일

### 5-2. Heading 계층 정리 + 홈 h1 부재

**문제**: (1) `NavigationBarDefault.tsx:99` h1 오용 (2) 홈 페이지 h1 없음 (3) challenge/[id] h1 없음 (4) challenge/home h1 조건부.
**결정**: NavigationBar h1 → div, 홈/챌린지에 시맨틱 h1 추가.
**공수**: 1일

### 5-3. challenge 페이지 메타 태그 추가

**문제**: (1) `challenge/[id].tsx` NextSeo 전무 (2) `challenge/home/index.tsx:13` title/description/canonical 없음.
**결정**: 챌린지 데이터 기반 NextSeo 추가.
**공수**: 0.5일

### 5-4. 커스텀 404 페이지

**문제**: `pages/404.tsx` 미존재.
**결정**: 브랜딩 + 내부 링크 포함 404 제작.
**공수**: 0.5일

### 5-5. challenge/[id] 에러 핸들링

**문제**: `challenge/[id].tsx:17` — `return null`. 로딩/에러 시 빈 페이지.
**결정**: `getServerSideProps`에서 `notFound: true` 반환.
**공수**: 0.5일

### 5-6. 내부 링크 크롤링 불가 수정

**문제**: (1) `ButtonHeaderBack.tsx:26` 로고 onClick만 (2) `SectionChallengeListDesktop.tsx:22` 챌린지 카드 router.push만 (3) `/explore` 네비게이션 미포함.
**영향**: 크롤러가 홈/챌린지 상세/탐색 페이지를 내부 링크로 발견 불가.
**결정**: next/link 래핑, /explore 메뉴 추가.
**공수**: 0.5일

### 5-7. OG & Social Meta 보강

**문제**: (1) og:locale 없음 (2) og:type 항상 website (3) og:site_name 소문자 (4) 기본 Twitter Card summary.
**결정**: DefaultSeo에 locale:'ko_KR', 상세 페이지 type:'article', siteName:'타임스프레드', 기본 Twitter summary_large_image.
**공수**: 0.5일

### 5-8. OG 이미지 압축

**문제**: `og_home.png` 700KB, `og_challenge.png` 508KB, `og_subscription_calendar.png` 562KB.
**결정**: JPEG quality 85로 재압축. 200KB 이하 목표.
**공수**: 0.5일

### 5-9. 시멘틱 HTML 수정

**문제**: (1) `SectionFooterCompany.tsx:8` article 오용 (2) `SectionFooterContact.tsx:11` address 중첩 오류.
**결정**: article → div, address 순서 역전.
**공수**: 15분

### 테마 5 합계

| 작업 | 공수 |
| --- | --- |
| 5-1. 구조화 데이터 보강 | 1.5일 |
| 5-2. Heading 정리 + h1 | 1일 |
| 5-3. challenge 메타 태그 | 0.5일 |
| 5-4. 커스텀 404 | 0.5일 |
| 5-5. challenge 에러 핸들링 | 0.5일 |
| 5-6. 내부 링크 크롤링 수정 | 0.5일 |
| 5-7. OG & Social Meta | 0.5일 |
| 5-8. OG 이미지 압축 | 0.5일 |
| 5-9. 시멘틱 HTML | 15분 |
| **합계** | **5.5일** |

### 테마 5 모니터링

| 지표 | 현재 | 목표 | 확인 위치 |
| --- | --- | --- | --- |
| challenge 페이지 인덱싱 | 미확인 | 주요 챌린지 인덱싱 | Search Console > 페이지 |
| 리치 스니펫 | 미확인 | breadcrumb 표시 | Rich Results Test |
| 구조화 데이터 오류 | 미확인 | 0건 | Search Console > 향상 |

---

## 실행 로드맵

### 🔴 긴급 (1-2주) — 총 4.5일

| 작업 | 테마 | 공수 |
| --- | --- | --- |
| Canonical query string 제거 | 1-1 | 0.5일 |
| JS 캐시 헤더 + TinyMCE lazy load | 1-2 | 1.5일 |
| robots.txt Disallow 추가 | 1-3 | 15분 |
| 히어로 배너 SSR prefetch + fetchpriority | 4-1 | 1일 |
| web-vitals 설치 + INP 모니터링 | 4-9 | 0.5일 |
| challenge/[id] 에러 핸들링 (return null 수정) | 5-5 | 0.5일 |

### 🟠 높음 (2-4주) — 총 8일

| 작업 | 테마 | 공수 |
| --- | --- | --- |
| CF 페이지 캐싱 | 2-1 | 0.5일 |
| viewport user-scalable=yes | 2-2 | 0.5일 |
| 보안 헤더 추가 | 2-3 | 0.5일 |
| 렌더 블로킹 리소스 최적화 | 4-2 | 0.5일 |
| Preconnect 추가 | 4-3 | 0.5일 |
| 폰트 preload (CSS + woff2) | 4-5 | 0.5일 |
| 스켈레톤 CLS 수정 | 4-6 | 1일 |
| 인증 gate LCP 개선 | 4-8 | 1일 |
| Heading 정리 + 홈 h1 | 5-2 | 1일 |
| challenge 메타 태그 추가 | 5-3 | 0.5일 |
| 내부 링크 크롤링 수정 (로고/챌린지 카드/explore) | 5-6 | 0.5일 |
| 커스텀 404 | 5-4 | 0.5일 |

### 🟡 중간 (1-2개월)

| 작업 | 테마 | 공수 |
| --- | --- | --- |
| 이미지 next/image 전환 (홈 + 상세) | 4-4 | 2~3일 |
| 구조화 데이터 보강 | 5-1 | 1.5일 |
| OG & Social Meta 보강 | 5-7 | 0.5일 |
| OG 이미지 압축 | 5-8 | 0.5일 |
| /explore title 키워드 보강 | 2-4 | 15분 |
| 시멘틱 HTML 수정 | 5-9 | 15분 |

### 🟢 장기 (조건부)

| 작업 | 테마 | 조건 |
| --- | --- | --- |
| Radix CSS 조건부 로드 | 4-7 | CSS 영향 측정 후 |
| Astro 전환 | 3 | URL 1만+, 미색인 50건+, LCP 4초+ |
| 사이트맵 동적 생성 | 부록 | 콘텐츠 증가 시 |

---

## 부록: MECE 진단 기록

### Issue Tree

```
SEO 최적화 전체 영역
├── A. Technical SEO (크롤링 & 인덱싱 인프라)
│   ├── A-1. Crawlability — 🔍
│   ├── A-2. Indexability — 🔍
│   ├── A-3. URL Structure — 🔍
│   ├── A-4. Site Speed — 🔍
│   ├── A-5. Mobile Friendliness — 🔍
│   ├── A-6. Security — 🔍
│   ├── A-7. International SEO — ✅
│   └── A-8. JS Rendering — 🔍
├── B. On-Page SEO (페이지 콘텐츠 최적화)
│   ├── B-1. Title & Meta — 🔍
│   ├── B-2. Heading Structure — 🔍
│   ├── B-3. Image Optimization — 🔍
│   ├── B-4. Structured Data — 🔍
│   ├── B-5. OG & Social Meta — 🔍
│   ├── B-6. Internal Linking — 🔍
│   └── B-7. Content Quality — 🔍
└── C. Performance SEO (Core Web Vitals)
    ├── C-1. LCP — 🔍
    ├── C-2. INP — 🔍
    ├── C-3. CLS — 🔍
    └── C-4. Resource Loading — 🔍
```

MECE 검증: A=인프라, B=콘텐츠, C=성능 — 상호배타적이며 코드로 검증 가능한 SEO 전체를 포괄.

### 기각된 가지

| 가지 | 기각 사유 |
|------|-----------|
| A-7. International SEO | `_document.tsx:42`에 `<Html lang="ko">` 확인. 단일 언어 서비스, hreflang 불필요 |

### Pruning 기록

| 가설 | 결과 | 사유 |
|------|------|------|
| hreflang 누락 | 기각 | 한국어 단일 서비스 |
| next/link 미사용으로 크롤링 전면 누락 | 기각 | 주요 내비게이션 23+ 파일에서 next/link 사용. 단, 특정 컴포넌트(로고, 챌린지 카드)만 router.push |
| trailing slash 불일치 | 기각 | Next.js 기본값(false) 사용 중, 문제 없음 |
| SSR 미적용 페이지 | 기각 | 모든 공개 인덱싱 대상 페이지 SSR + dehydrate 확인 |
| 혼합 콘텐츠(HTTP 리소스) | 기각 | 모든 외부 URL이 https 사용 확인 |
| 이미지 alt 전체 누락 | 기각 | 주요 이미지에 동적 alt 설정. getAltText() 함수로 컨텍스추얼 alt 생성 |
| GA 렌더 블로킹 | 기각 | next/script strategy="afterInteractive" + dynamic ssr:false |
| 홈 배너 이미지 CLS | 기각 | 스켈레톤과 실제 Swiper 동일 aspect-ratio |
| Kakao SDK SRI 미적용 | 기각 | integrity + crossOrigin="anonymous" 적용 확인 |

### Coverage Gate 확인

가지 수: 19개 / 판정 완료: 19개 / ⬜ 잔여: 0개

---

## 부록 B. 크롤링 데이터 실측 요약 (2025.12 ~ 2026.03)

| 지표 | 값 |
| --- | --- |
| 총 크롤링 URL | 6,591건 |
| 일평균 크롤링 요청 | ~60건 |
| JS 크롤링 비율 | 20.86% |
| 200 응답 비율 | 86.68% |
| 404 비율 | 3.23% |
| 301 비율 | 8.89% |
| 평균 응답 시간 | 500~1000ms |
| 크롤됨-미색인 | 36건 |
| 발견됨-미색인 | 2건 |
