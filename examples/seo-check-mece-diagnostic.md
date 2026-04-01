# SEO 체크리스트 MECE Diagnostic 분석

- 작성일: 2026년 3월 12일
- 대상: seo-check.md
- 분석 스킬: MECE Diagnostic (McKinsey Diagnostic Pruning)
- 분석 레벨: Standard

---

## Issue Tree — SEO 체크리스트 완전성 분해

```
SEO 체크리스트에 빠진 사항이 있는가?
├── A. 테크니컬 SEO (크롤링/인덱싱/렌더링)
│   ├── A-1. 크롤링 제어 (robots, sitemap, canonical)
│   ├── A-2. 렌더링/JS 처리
│   ├── A-3. URL 구조/리다이렉트
│   └── A-4. 보안/프로토콜 (HTTPS, HSTS)
├── B. 온페이지 SEO (콘텐츠/메타/구조)
│   ├── B-1. 메타 태그 (title, description, OG, Twitter)
│   ├── B-2. 시맨틱 마크업 (heading, schema, alt)
│   ├── B-3. 콘텐츠 품질/중복
│   └── B-4. 국제화/언어 (hreflang, lang)
├── C. 성능 (Core Web Vitals)
│   ├── C-1. LCP (로딩)
│   ├── C-2. FID/INP (인터랙티비티)
│   ├── C-3. CLS (레이아웃 안정성)
│   └── C-4. TTFB/FCP
└── D. 오프페이지/기타
    ├── D-1. 사이트맵 동적 관리
    ├── D-2. 모바일 사용성
    └── D-3. 내부 링크 구조
```

---

## 검증 매트릭스

| 가지 | seo-check.md 커버 여부 | 코드 검증 결과 |
|------|----------------------|--------------|
| A-1. 크롤링 제어 (robots, canonical) | ✅ 테마 1 | - |
| A-1. **사이트맵 관리** | ⚠️ 부록에서 언급만 | **실행 로드맵 미반영** |
| A-2. 렌더링/JS | ✅ 테마 1-2 | - |
| A-3. URL 구조/리다이렉트 | ✅ 301 언급 | - |
| A-4. **보안 헤더** | ❌ | **HSTS 등 미설정 확인** |
| B-1. 메타 태그 | ✅ 테마 2-3 | - |
| B-2. 시맨틱 마크업 | ✅ 테마 5 | - |
| B-3. 콘텐츠 품질/중복 | ✅ 테마 2-2 | - |
| B-4. 국제화 | ✅ lang 언급 | hreflang 불필요 (단일 언어) |
| C-1. LCP | ✅ 테마 4 | - |
| C-2. **FID/INP** | ❌ | **web-vitals 미적용 확인** |
| C-3. CLS | ✅ 테마 4-4 | - |
| C-4. TTFB/FCP | ✅ 테마 2-1, 4-1 | - |
| D-1. **사이트맵 동적 관리** | ❌ | **외부 CDN 정적 호스팅 확인** |
| D-2. 모바일 사용성 | ✅ viewport | - |
| D-3. 내부 링크 | ✅ 부록 | next/link 23개 파일 사용 확인 |

---

## 누락 항목 상세

### 누락 1: 사이트맵 동적 생성 / lastmod (Impact: High)

**현재 상태**:
- sitemap은 `media.timespread.co.kr` 외부 CDN에 정적 호스팅
- 로컬에 생성 로직 없음 (next-sitemap 미사용, 동적 생성 엔드포인트 없음)
- `next.config.js`에서 `/seo/:path*` → CDN으로 rewrite

**문서 누락 사항**:
- `lastmod` 포함 여부 미확인/미언급
- 부록에서 "sitemap 관리, `<lastmod>` 전달"을 Crawl Demand 개선 방법으로 언급했으나 실행 로드맵에 미반영
- 6,500 URL에서 새 이벤트/캘린더 추가 시 sitemap 업데이트 프로세스 불명확

**권장**: 실행 로드맵 🟠 높음에 sitemap 동적 생성 방안(next-sitemap 또는 API 기반) 추가

---

### 누락 2: INP (Interaction to Next Paint) 모니터링 (Impact: Medium)

**현재 상태**:
- 2024년 3월부터 Google이 FID를 INP로 대체 (이미 적용 중)
- 프로젝트에 `web-vitals` 패키지 없음
- passive event listener 최적화 없음

**문서 누락 사항**: 테마 4(Core Web Vitals)에서 LCP, FCP, CLS만 다루고 INP 아예 미언급

**권장**: 테마 4 모니터링 지표에 INP 추가. 현재 CWV 양호하므로 모니터링 항목 추가로 충분

---

### 누락 3: 보안 헤더 (Impact: Low-Medium)

**현재 상태**: `next.config.js` headers에 Cache-Control만 설정

**누락된 헤더**:
- `Strict-Transport-Security` (HSTS) — HTTP→HTTPS 301 크롤링 비용 절감
- `X-Content-Type-Options` — MIME 스니핑 방어
- `Referrer-Policy` — 리퍼러 정보 제어

**SEO 관련성**:
- HSTS는 HTTPS 리다이렉트 과정의 크롤링 비용 절감
- 직접적 랭킹 영향은 낮으나 Lighthouse Best Practices 점수에 영향

**권장**: 테마 2-3 "기타 테크니컬 SEO" 개선 항목에 추가

---

### 누락 4: 테마 1 합계 오기 (Minor)

테마 1 합계가 "3~4일"로 표기되어 있으나 실제 합산: 0.5 + 1.5 + 0 = **2일**

---

## Pruning 기록

| 가설 | 결과 | 이유 |
|------|------|------|
| hreflang 누락 | 기각 | 한국어 단일 서비스, 다국어 지원 불필요 |
| next/link 미사용으로 크롤링 누락 | 기각 | 23개 파일에서 next/link 사용, a 태그는 외부 링크 4곳뿐 |
| trailing slash 불일치 | 기각 | Next.js 기본값 false 사용 중, 문제 없음 |
| noindex 설정 누락 | 기각 | 환경별 + 경로별(11개 경로) 잘 설정되어 있음 |
| lang 속성 미설정 | 기각 | `_document.tsx`에 `<Html lang="ko">` 이미 설정됨 |

---

## 우선순위 요약

| 순위 | 누락 항목 | 추가 권장 위치 | 영향 |
|------|----------|--------------|------|
| 1 | **사이트맵 동적 생성/lastmod** | 실행 로드맵 🟠 높음 | Crawl Demand 직접 영향 |
| 2 | **INP 모니터링** | 테마 4 모니터링 지표 | CWV 3대 지표 중 하나 |
| 3 | **보안 헤더** | 테마 2-3 개선 항목 | Lighthouse 점수, 간접 신뢰 시그널 |
| 4 | **테마 1 합계 오기** (3~4일 → 2일) | 테마 1 합계 | 단순 오타 |

---

## 전체 평가

seo-check.md의 분석 품질과 커버리지는 전반적으로 **탄탄함**:

- 크롤링 예산 분석이 Google 공식 문서 기반으로 정확
- Astro 전환 불필요 판단의 근거가 논리적 (사이트 규모, JS 캐싱, 전환 비용)
- 각 항목의 선택지 → 결정 → 근거 흐름이 일관됨
- 모니터링 지표까지 포함해서 실행 후 검증 가능
- CF 비용 분석, 크롤링 데이터 실측 등 부록이 충실

위 4개 항목 외에는 누락 없음.
