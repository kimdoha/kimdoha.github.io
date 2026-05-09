# Design System — Anthropic Base + Stripe Enhancement

이 문서는 블로그의 디자인 시스템 토큰과 설계 근거를 정리합니다.

---

## Design Philosophy

- **Anthropic Base**: 연구 저널 톤. 따뜻한 중성 팔레트(ivory, slate, clay), serif 헤딩, mono 메타데이터, 직각 태그.
- **Stripe Enhancement**: 부드러운 다층 그림자, 넉넉한 여백, 미세한 hover 전환으로 세련된 인터랙션 레이어 추가.
- **불변 영역**: 코드 블럭(One Dark), 모바일 반응형 구조, 아바타 숨김.

---

## Color Tokens

### Light Mode (`:root`)

```yaml
# Slate (텍스트 계열)
ct-text:        "#141413"   # Anthropic Slate Dark — 본문
ct-heading:     "#141413"   # 헤딩 통일
ct-heading-sub: "#141413"   # 서브 헤딩 통일
ct-strong:      "#141413"   # 강조 통일
ct-muted:       "#5e5d59"   # Anthropic Slate Light
ct-meta:        "#87867f"   # Anthropic Cloud Dark
ct-meta-light:  "#b0aea5"   # Anthropic Cloud Medium

# Cloud (보더/배경 계열)
ct-border:              "#d1cfc5"   # Anthropic Cloud Light
ct-border-table:        "#d1cfc5"
ct-border-table-head:   "#c5bfb3"   # 약간 진한 Cloud
ct-border-table-row:    "#e3ddd3"   # 따뜻한 row border

# Ivory (배경 계열)
ct-bg-blockquote:   "#f0eee6"   # Anthropic Ivory Medium
ct-bg-table-head:   "#f0eee6"
ct-bg-table-hover:  "#f5f3ed"   # 약간 밝은 ivory

# Accent
ct-accent:             "#d97757"   # Anthropic Clay (terracotta)
ct-border-blockquote:  "#d97757"

# Inline code
ct-inline-code-bg:     "#ede9e0"   # 따뜻한 톤
ct-inline-code-color:  "#8b4513"   # 따뜻한 갈색

# Card
ct-card-border:       "#d1cfc5"
ct-card-hover-border: "#c5bfb3"
ct-card-title:        "#141413"
ct-card-text:         "#5e5d59"

# Category
ct-cat-header-bg:      "#f0eee6"
ct-cat-header-border:  "#d1cfc5"
ct-cat-item-text:      "#3d3d3a"   # Anthropic Slate Medium
ct-cat-item-hover-bg:  "#f0eee6"

# Tag
ct-tag-bg:           "#ede9e0"
ct-tag-color:        "#87867f"
ct-tag-hover-bg:     "#e3ddd3"
ct-tag-hover-color:  "#d97757"   # Terracotta on hover
ct-tag-hover-border: "#d97757"
ct-tag-muted:        "#b0aea5"

# Archive
ct-archive-link:      "#141413"
ct-archive-day:       "#5e5d59"
ct-archive-month:     "#87867f"
ct-archive-hover-bg:  "#f0eee6"
```

### Dark Mode (`@mixin dark-vars`)

따뜻한 다크 — 순수 검정 대신 brown/gray 톤.

```yaml
ct-text:        "#d1cfc5"
ct-heading:     "#e8e6dc"
ct-heading-sub: "#e3ddd3"
ct-strong:      "#e8e6dc"
ct-muted:       "#87867f"
ct-meta:        "#87867f"
ct-meta-light:  "#5e5d59"
ct-border:      "#2e2a25"
ct-accent:      "#e0926b"   # 밝은 terracotta

ct-bg-blockquote:      "#1e1b18"
ct-border-blockquote:  "#e0926b"
ct-inline-code-bg:     "#2a2520"
ct-inline-code-color:  "#e0926b"

ct-border-table:       "#2e2a25"
ct-border-table-head:  "#3a3530"
ct-border-table-row:   "#2e2a25"
ct-bg-table-head:      "#231f1b"
ct-bg-table-hover:     "#231f1b"

ct-card-border:        "#2e2a25"
ct-card-hover-border:  "#4a4035"
ct-card-title:         "#e8e6dc"
ct-card-text:          "#87867f"

ct-cat-header-bg:      "#231f1b"
ct-cat-header-border:  "#2e2a25"
ct-cat-item-text:      "#b0aea5"
ct-cat-item-hover-bg:  "#231f1b"

ct-tag-bg:             "#2a2520"
ct-tag-color:          "#b0aea5"
ct-tag-hover-bg:       "#3a3530"
ct-tag-hover-color:    "#e0926b"
ct-tag-hover-border:   "#e0926b"
ct-tag-muted:          "#5e5d59"

ct-archive-link:       "#d1cfc5"
ct-archive-day:        "#87867f"
ct-archive-month:      "#5e5d59"
ct-archive-hover-bg:   "#231f1b"
```

### Page Background

```yaml
light: "#faf9f5"   # Anthropic Ivory Light
dark:  "#1a1815"   # 따뜻한 다크
```

---

## Typography

```yaml
body:
  family: "'Spoqa Han Sans Neo', -apple-system, BlinkMacSystemFont, system-ui, 'Segoe UI', sans-serif"
  size: 0.78rem (base 12px)
  line-height: 1.75
  letter-spacing: -0.008em
  reason: "한글 최적화 우수. Anthropic도 본문은 Sans 사용."

heading-h1:
  family: "'Noto Serif KR', serif"
  weight: 700
  letter-spacing: -0.02em
  reason: "Serif → 연구 저널의 권위감. Anthropic 제목 톤 반영."

heading-h2-h3:
  family: "inherit (Spoqa Han Sans Neo)"
  weight: 800 (h2), 700 (h3)
  reason: "본문 레벨은 Sans 유지. Anthropic도 섹션 제목은 Sans."

code:
  family: "'JetBrains Mono', 'Fira Code', 'SF Mono', Menlo, Monaco, Consolas, monospace"
  reason: "가독성 높은 코딩 폰트. 리가처 지원."

metadata:
  family: "'JetBrains Mono', monospace"
  transform: uppercase
  letter-spacing: 0.04em
  reason: "Anthropic의 mono metadata 패턴. 날짜/카테고리에 기술적 톤 부여."
```

---

## Spacing & Layout

```yaml
post-content-max-width: 700px
card-padding: 1.1rem 1.3rem
blockquote-padding: 0.75rem 1rem
code-block-padding: 0.9rem 2rem
```

---

## Shadows (Stripe Influence)

```yaml
card-default:  "0 1px 3px rgba(0, 0, 0, 0.04)"
card-hover:    "0 4px 12px rgba(0, 0, 0, 0.08)"
code-block:    "0 4px 16px rgba(0, 0, 0, 0.15), 0 1px 3px rgba(0, 0, 0, 0.08)"
reason: "Stripe의 다층 그림자 — 깊이감을 주되 과하지 않게."
```

---

## Border Radius

```yaml
card:        8px    # 10px → 8px (Anthropic 톤)
tag:         0      # 12px → 0 (Anthropic squared)
blockquote:  "0 6px 6px 0"
code-block:  10px   # 유지
table:       6px    # 유지
```

---

## Design Rationale

### 왜 Anthropic + Stripe인가?

1. **Anthropic**: 따뜻한 중성 팔레트가 긴 기술 글 읽기에 적합. 순수 흰색/검정 대비보다 눈의 피로도가 낮음. Terracotta 액센트는 학술 저널의 신뢰감을 줌.
2. **Stripe**: 그림자와 전환 효과가 인터랙션 피드백을 자연스럽게 제공. 카드, 호버 상태에서 "떠오르는" 느낌이 콘텐츠 탐색을 유도.
3. **조합 원칙**: 색상·타이포는 Anthropic, 인터랙션·깊이감은 Stripe. 두 시스템의 교집합(미니멀, 고대비 텍스트, 넉넉한 여백)을 기반으로 충돌 없이 병합.

### 불변 영역

- 코드 블럭: One Dark 테마 전체 (구문 강조, 배경, 라인 넘버)
- 모바일 반응형 breakpoint 및 구조
- `#sidebar #avatar { display: none }`