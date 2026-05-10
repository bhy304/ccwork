# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

React 19 + TypeScript + Vite 기반 노트 앱 실습 프로젝트. JSON Server를 백엔드로 사용하는 풀스택 구성.

## 주요 명령어

```bash
npm run dev          # Vite 개발 서버 + JSON Server 동시 실행 (concurrently)
npm run build        # TypeScript 컴파일 후 Vite 빌드
npm run lint         # ESLint 검사 (--fix 자동 적용)
npm run format       # Prettier 포맷
npm test             # Vitest 단일 실행
npm run test:watch   # Vitest watch 모드
npm run server       # JSON Server만 단독 실행 (포트 3001)
```

- 앱: http://localhost:5173
- API: http://localhost:3001/notes

## 아키텍처

```
ccwork/
├── db.json                      # JSON Server 데이터 파일 (git 포함)
└── src/
    ├── main.tsx                 # 앱 진입점
    ├── App.tsx                  # UI 상태 관리 (selectedNoteId, isCreating)
    ├── types/
    │   └── note.ts              # Note 인터페이스 정의
    ├── api/
    │   └── notes.ts             # JSON Server fetch 함수 (CRUD)
    ├── context/
    │   └── NotesContext.tsx     # 전역 서버 상태 + useNotes() 훅
    └── components/
        ├── Layout.tsx           # 사이드바/메인 슬롯 레이아웃 쉘
        ├── NoteList.tsx         # 노트 목록 (선택/삭제)
        ├── NoteItem.tsx         # 개별 노트 카드
        └── NoteEditor.tsx       # 생성·편집 통합 폼
```

### 데이터 흐름

컴포넌트는 `useNotes()` 훅을 통해서만 상태와 액션에 접근하며, `api/notes.ts`를 직접 호출하지 않는다.

```
NoteList / NoteEditor
      │  useNotes()
      ▼
NotesContext  ──→  api/notes.ts  ──→  JSON Server (localhost:3001/notes)
      │                                        │
      └── notes[] 낙관적 업데이트  ←────────────┘
```

### 백엔드 (JSON Server)

- `db.json`을 파일 DB로 사용하는 mock REST API
- `createNote` / `updateNote` 시 `createdAt` / `updatedAt` 타임스탬프를 **클라이언트**에서 생성하여 전송
- `NotesProvider` 마운트 시 `fetchNotes()`로 초기 데이터 로드, 이후 액션 실행마다 로컬 `notes[]`를 즉시 업데이트

## 구현 패턴

### 컴포넌트 패턴
- **Named export** 사용 (`export function NoteList`). `App.tsx`만 예외로 default export
- Props 타입은 컴포넌트 바로 위에 `interface ComponentNameProps`로 선언
- 조건 분기(loading → error → empty)를 early return으로 처리한 뒤 정상 렌더 반환
- 이벤트 핸들러는 컴포넌트 내부에서 정의 시 `handle` prefix, props로 전달 시 `on` prefix

### 상태 관리 방식
- **서버 상태** (`NotesContext`): `notes[]`, `loading`, `error` — API와 동기화, context 내부에서만 변경
- **UI 상태** (`App.tsx`): `selectedNoteId`, `isCreating` — 서버와 무관한 화면 선택/모드 상태
- `useNotes()` 훅은 Provider 외부에서 호출 시 즉시 에러를 throw하여 잘못된 사용을 방지
- Context 액션은 API 호출 성공 후 로컬 `notes[]`를 직접 업데이트 (재fetch 없음)

### API 호출 패턴
- `api/notes.ts`에서 named export로 함수 분리, context에서는 `import * as api`로 네임스페이스 임포트
- 모든 API 함수는 `res.ok` 체크 후 실패 시 `throw new Error(...)` — try/catch는 호출부(context)에서 처리
- `deleteNote`는 반환값 없이 `Promise<void>`, 나머지는 `Promise<Note>` 반환

### 네이밍 규칙
| 대상 | 규칙 | 예시 |
|------|------|------|
| 컴포넌트/타입 | PascalCase | `NoteEditor`, `Note` |
| 컴포넌트 파일 | PascalCase | `NoteEditor.tsx` |
| api/context/types 파일 | camelCase | `notes.ts`, `NotesContext.tsx` |
| Props 인터페이스 | `XxxProps` | `NoteEditorProps` |
| Context/Provider/훅 | `XxxContext` / `XxxProvider` / `useXxx` | `NotesContext`, `useNotes` |
| 이벤트 핸들러 (내부) | `handleXxx` | `handleSave`, `handleNewNote` |
| 이벤트 핸들러 (props) | `onXxx` | `onSelect`, `onDone` |
| API 함수 | `verbNoun` | `fetchNotes`, `createNote`, `deleteNote` |

### 에러 처리
- `alert()` 사용 금지 — 에러는 `console.error`로만 처리
- API 에러: `api/notes.ts`에서 `throw new Error(...)`, context에서 try/catch 후 `console.error`
- 유효성 검사 실패(예: 제목 없음)도 `console.error`로 기록 후 early return

## 주의: 일관성이 없는 패턴

1. **에러 메시지 언어 불일치**
   - `api/notes.ts` 에러: 영어 (`'Failed to fetch notes'`)
   - UI 렌더링 에러: 한국어 (`'오류: {error}'`)

2. **NoteEditor의 useEffect 의존성 억제**
   - `selectedNote`(객체)가 아닌 `selectedNoteId`(원시값)를 dep으로 사용하며 eslint 경고를 `// eslint-disable-line`으로 억제
   - 의도적 선택이나 향후 `selectedNote`를 dep에 포함하면 억제 주석 제거 가능

## 기술 스택

- **UI**: React 19, Tailwind CSS v4 (Vite 플러그인 방식)
- **빌드**: Vite 6, TypeScript 5.7
- **테스트**: Vitest + jsdom + Testing Library (setupFiles: `src/test-setup.ts`)
- **린트/포맷**: ESLint 9 (flat config), Prettier

## 타입

`Note` 타입 (`src/types/note.ts`): `id`, `title`, `content`, `createdAt`, `updatedAt`. `tags` 필드는 아직 미구현 (강의에서 추가 예정).
