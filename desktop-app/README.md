# Persona Pipeline Simulator — 데스크톱 앱 (Electron)

[pipeline-simulator/index.html](../pipeline-simulator/index.html)와 동일한 콘텐츠 파이프라인 시뮬레이터를
브라우저 없이 더블클릭 하나로 실행되는 **단일 포터블 .exe**로 패키징한 버전입니다.

> 실제 ComfyUI/GPU를 실행하지 않는 시뮬레이션입니다. UI/실행 흐름/타이밍만 재현합니다.

## 왜 Electron인가

이 환경에서 실제로 쓸 수 있는 런타임은 Node.js/npm뿐이었습니다 (`python`은 Windows 스토어 스텁만
설치돼 있어 실행 불가). Node 기반 데스크톱 프레임워크 중 Electron을 선택한 이유:

- 이미 만들어둔 파이프라인 시뮬레이터 HTML/CSS/JS를 거의 그대로 재사용 가능 (재작성 불필요)
- `electron-builder`의 `portable` 타겟으로 설치 과정 없는 단일 .exe 생성 가능
- Tauri(Rust 툴체인 필요)나 PyInstaller(Python 미설치)보다 이 환경에서 즉시 빌드 가능

## 빌드 방법

```bash
cd desktop-app
npm install
npm run dist   # dist/PersonaPipelineSimulator.exe 생성 (포터블, 약 70MB)
```

개발 중 미리보기(빌드 없이 바로 실행):

```bash
npm start
```

## 실행 시 주의 — Windows SmartScreen / Application Control

빌드된 exe는 **코드 서명이 되어 있지 않습니다** (서명 인증서는 유료이며 이 프로젝트 범위 밖입니다).
그래서 처음 실행할 때 다음이 나타날 수 있습니다.

- **Windows SmartScreen**: "Windows에서 PC를 보호했습니다" 경고 → "추가 정보" → "실행" 클릭
- **회사/관리형 PC의 Application Control 정책**: 서명 안 된 실행 파일 자체를 차단할 수 있음 —
  이 경우 관리자에게 문의하거나, 개인 PC에서 실행하세요 (실제 이 프로젝트를 빌드한 환경에서도
  이 정책 때문에 로컬 실행 확인이 불가능했습니다)

## 배포하지 않는 이유

`node_modules/`와 `dist/`는 `.gitignore`로 저장소에서 제외했습니다 (바이너리·의존성 트리를 git에
커밋하지 않는다는 일반 원칙). 실행 파일이 필요하면 위 빌드 방법으로 직접 생성하세요.
