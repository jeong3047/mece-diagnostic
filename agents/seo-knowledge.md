# SEO 도메인 지식 — MECE 분석 시 참고 사항

MECE는 **사고 방식**(구조화된 분해)이고, 도메인 지식은 **별도로 정리**해야 한다.
이 파일은 SEO 분석 시 MECE Issue Tree를 만들 때 참고할 도메인 지식을 정리한다.

---

## SEO Issue Tree 출발점 프레임워크

기술 SEO를 MECE 분해할 때 검증된 출발점:

```
기술 SEO 전수 점검
├── A. 테크니컬 SEO (크롤링/인덱싱/렌더링)
│   ├── A-1. 크롤링 제어 (robots.txt, sitemap, canonical)
│   ├── A-2. 렌더링/JS 처리 (SSR, hydration, WRS)
│   ├── A-3. URL 구조/리다이렉트 (301, 404, trailing slash)
│   └── A-4. 보안/프로토콜 (HTTPS, HSTS, 보안 헤더)
├── B. 온페이지 SEO (콘텐츠/메타/구조)
│   ├── B-1. 메타 태그 (title, description, OG, Twitter)
│   ├── B-2. 시맨틱 마크업 (heading 계층, schema, alt)
│   ├── B-3. 콘텐츠 품질/중복
│   ├── B-4. 국제화/언어 (hreflang, lang)
│   ├── B-5. 내부 링크 구조 (크롤러 발견 가능성)
│   └── B-6. 구조화 데이터 (JSON-LD)
├── C. 성능 (Core Web Vitals)
│   ├── C-1. LCP (로딩 — SSR prefetch, 이미지 최적화, fetchpriority)
│   ├── C-2. INP (인터랙티비티 — JS 번들, hydration, blocking)
│   ├── C-3. CLS (레이아웃 안정성 — 스켈레톤, 폰트, 이미지 사이즈)
│   └── C-4. TTFB/FCP (서버 응답 — CDN 캐싱, SSR 최적화)
└── D. 오프페이지/기타
    ├── D-1. 사이트맵 동적 관리 (lastmod, 자동 생성)
    ├── D-2. 모바일 사용성 (viewport, 터치 타겟)
    └── D-3. GEO (AI 검색 최적화 — 구조화 데이터, llms.txt)
```

---

## Google 공식 문서 핵심 참조

### Crawl Budget

- **Crawl Budget = min(Capacity, Demand)** — 합이 아님
- 속도 개선은 Capacity 천장을 올리는 것이지 Demand를 늘리는 것이 아님
- 1만 URL 미만은 대부분 크롤링 예산 걱정 불필요
- 출처: [Crawl Budget Management](https://developers.google.com/crawling/docs/crawl-budget)

### WRS (Web Rendering Service)

- WRS는 JS를 최대 30일 캐싱
- content fingerprinting(해시 파일명) 사용 시 배포 때만 재다운로드
- HTTP 캐시 헤더를 "may ignore" — 항상 무시하는 것 아님
- 출처: [Crawling December: Resources](https://developers.google.com/search/blog/2024/12/crawling-december-resources)

### Core Web Vitals

- 2024년 3월부터 FID → INP 대체 (이미 적용 중)
- LCP, INP, CLS 3가지가 현재 랭킹 시그널
- 출처: [Core Web Vitals](https://web.dev/articles/vitals)

### 내부 링크

- Google은 내부 링크로 새 URL을 **발견(Discovery)**
- `router.push`만 있으면 크롤러가 해당 페이지를 발견 불가 — `next/link` 또는 `<a>` 필요
- 출처: [How Search Works](https://developers.google.com/search/docs/fundamentals/how-search-works)

---

## 검증 시 주의 사항

| 흔한 착각 | 실제 |
|-----------|------|
| "크롤링 속도 개선하면 양도 늘어남" | Capacity ≠ Demand. 양은 Demand가 결정 |
| "SSR이면 JS 크롤링 비용 없음" | SSR HTML + JS 둘 다 크롤링됨 (WRS 2차 렌더링) |
| "사이트맵 제출하면 인덱싱됨" | 사이트맵은 힌트일 뿐. 콘텐츠 품질이 결정 |
| "캐시 헤더를 WRS가 무시함" | "may ignore"이지 "always ignore" 아님 |
| "URL 수 = JS 번들 수" | Next.js는 라우트 단위 번들. 6,500 URL이어도 JS ~12개 |
