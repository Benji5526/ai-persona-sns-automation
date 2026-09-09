# AI Persona SNS Automation — 설계 문서

저장소: https://github.com/Benji5526/ai-persona-sns-automation

개인 페르소나를 중심으로 SNS 운영을 자동화하는 시스템의 설계 저장소입니다.
**이 저장소는 설계(design)만 다룹니다. 실제 구현 코드는 포함하지 않습니다.**

## 도메인 개요

이 시스템은 크게 두 축으로 구성됩니다.

1. **대화 자동화** — SNS DM/댓글을 페르소나가 인식하고, 사람이 하나씩 답장하듯
   자연스러운 속도와 순서로 응답하는 자동화 툴 (일괄 동시 발송 아님)
2. **콘텐츠 자동화** — ComfyUI로 학습한 페르소나 이미지 모델을 기반으로
   표정 변화 / 짧은 영상 / 의상·메이크업 체인지 파이프라인을 만들고,
   구매한 UGC(영상/이미지)에 학습한 얼굴을 페이스스왑하여 콘텐츠를 대량 생산한 뒤,
   (선택적으로) 연결된 SNS 계정에 자동 포스팅까지 이어지는 파이프라인

## 청사진 (도면 4매)

전체 설계를 도면 형식의 단일 페이지로 정리했습니다 — [blueprint/index.html](blueprint/index.html)
(다운로드 후 브라우저로 열어서 확인하세요. GitHub 미리보기에서는 렌더링되지 않습니다.)

## 아키텍처

```mermaid
flowchart TB
    subgraph PERSONA["페르소나 정의 (공용 자산)"]
        P1["Character Sheet<br/>말투/지식/경계선"]
        P2["학습된 이미지 모델<br/>(LoRA/Checkpoint)"]
    end

    subgraph CONV["축 A. 대화 자동화"]
        A1[SNS DM/댓글 수집기]
        A2[의도·문맥 분석]
        A3[휴먼라이크 응답 스케줄러]
        A4[응답 발송]
        A1 --> A2 --> A3 --> A4
    end

    subgraph CONTENT["축 B. 콘텐츠 자동화"]
        B1[데이터셋 큐레이션]
        B2[ComfyUI 모델 트레이닝]
        B3[표정/영상/의상 파이프라인]
        B4[구매 UGC 페이스스왑]
        B5[QC 게이트]
        B6[자동 포스팅]
        B1 --> B2 --> B3 --> B5
        B2 --> B4 --> B5
        B5 --> B6
    end

    P1 -.페르소나 톤 반영.-> A2
    P2 -.모델 제공.-> B3
    P2 -.얼굴 소스 제공.-> B4
    B6 -.게시 결과/반응.-> A1
```

세부 다이어그램(시퀀스, 큐 구조, ERD 등)은 [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) 및 각 모듈 문서에 있습니다.

## 문서 구조

| 문서 | 내용 |
|---|---|
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | 전체 시스템 아키텍처, 모듈 간 데이터 흐름 |
| [docs/01-reply-automation.md](docs/01-reply-automation.md) | DM/댓글 휴먼라이크 자동응답 설계 |
| [docs/02-model-training.md](docs/02-model-training.md) | ComfyUI 기반 페르소나 이미지 모델 트레이닝 |
| [docs/03-content-pipeline.md](docs/03-content-pipeline.md) | 표정 변화 / 숏폼 영상 / 의상·메이크업 체인지 파이프라인 |
| [docs/04-faceswap-ugc.md](docs/04-faceswap-ugc.md) | 구매 UGC 페이스스왑 자동화 |
| [docs/05-auto-posting.md](docs/05-auto-posting.md) | 자동 포스팅 연동 (확장 단계) |
| [docs/06-data-model.md](docs/06-data-model.md) | 페르소나/대화/에셋 데이터 모델 |
| [docs/07-compliance.md](docs/07-compliance.md) | 운영·윤리·법적 고려사항 (필독) |
| [docs/08-roadmap.md](docs/08-roadmap.md) | 단계별 구축 로드맵 |
| [docs/09-llm-serving.md](docs/09-llm-serving.md) | 로컬 LLM 서빙 (vLLM, 멀티 LoRA) |
| [docs/10-budget-simulation.md](docs/10-budget-simulation.md) | 예상 예산 시뮬레이션 (GPU/LLM/UGC/SNS API) |
| [docs/11-recommended-sequence.md](docs/11-recommended-sequence.md) | 권장 착수 순서 — 무엇을 먼저 검증할 것인가 (필독) |
| [docs/12-hardware-spec.md](docs/12-hardware-spec.md) | 노트북 사양 및 예산 배분 (1,000만원 기준) |
| [docs/13-pilot-plan.md](docs/13-pilot-plan.md) | 파일럿 실행 계획 (1~2개월, 월 1만원 미만) |

## 설계 원칙

- **사람처럼 보이는 리듬** — 자동응답은 "동시에 여러 개"가 아니라 "한 번에 하나씩, 사람의 타이핑·확인·응답 리듬"을 흉내낸다.
- **페르소나 일관성** — 대화형/이미지형 산출물 모두 하나의 페르소나 정의(character sheet)에서 파생된다.
- **파이프라인 재사용** — 트레이닝된 모델은 정지 이미지, 영상, 페이스스왑 세 갈래 파이프라인에서 공통으로 소비된다.
- **QC 게이트 필수** — 자동 생성물은 사람 검수 또는 자동 품질 체크를 통과해야 포스팅 단계로 넘어간다.
- **고지·동의 우선** — AI 생성/합성 콘텐츠라는 사실과 UGC 원저작자 동의는 설계 초기 단계부터 반영한다.

## 라이선스

[All Rights Reserved](LICENSE) — 저작권자의 사전 서면 동의 없이 복제·배포·2차 저작·상업적 이용을 금지합니다.
