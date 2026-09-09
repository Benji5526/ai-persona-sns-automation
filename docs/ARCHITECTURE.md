# 전체 아키텍처

저장소: https://github.com/Benji5526/ai-persona-sns-automation

## 1. 시스템 맵

시스템은 서로 독립적으로 동작할 수 있는 두 개의 축과, 이를 잇는 공용 자산(페르소나 정의)으로 구성됩니다.

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

    subgraph CONTENT["축 B. 콘텐츠 자동화 (3개 독립 엔진)"]
        B1[데이터셋 큐레이션]
        B2[ComfyUI 모델 트레이닝]
        B3["생성 엔진<br/>표정/영상/의상"]
        B4["얼굴교체 엔진<br/>구매 UGC 페이스스왑"]
        B4b["모션 리타겟팅 엔진<br/>드라이빙 영상 → 새 영상"]
        B5[QC 게이트]
        B6[자동 포스팅]
        B1 --> B2 --> B3 --> B5
        B2 --> B4 --> B5
        B2 --> B4b --> B5
        B5 --> B6
    end

    P1 -.페르소나 톤 반영.-> A2
    P2 -.모델 제공.-> B3
    P2 -.얼굴 소스 제공.-> B4
    P2 -.정체성 소스 제공.-> B4b
    B6 -.게시 결과/반응.-> A1
```

두 축은 완전히 분리해 개발 가능하지만, 실제 운영 시에는 "콘텐츠 자동화"가 만든 게시물에 달리는 댓글/DM을 "대화 자동화"가 받아 응대하는 순환 구조를 이룹니다.

## 2. 컴포넌트 책임 요약

| 컴포넌트 | 책임 | 상세 문서 |
|---|---|---|
| SNS 수집기 | 플랫폼별 DM/댓글 웹훅·폴링 수신, 정규화 | [01](01-reply-automation.md) |
| 응답 스케줄러 | "동시 발송"이 아닌 "순차·지연" 큐잉 | [01](01-reply-automation.md) |
| 로컬 LLM 서버 (vLLM) | 의도 분석·응답 생성 추론, 페르소나별 LoRA 멀티 서빙 | [09](09-llm-serving.md) |
| 모델 트레이닝 | 페르소나 이미지 데이터셋 → LoRA/체크포인트 | [02](02-model-training.md) |
| 표정/영상/의상 파이프라인 (생성 엔진) | ComfyUI 워크플로우 그래프로 변형 생성 (diffusion 기반) | [03](03-content-pipeline.md) |
| 얼굴교체 엔진 | 구매 UGC(이미지/영상)에 학습된 얼굴 합성 — 동일 모델, 입력 타입만 분기 | [04](04-faceswap-ugc.md) |
| 모션 리타겟팅 엔진 | 페르소나 이미지 정체성 + 드라이빙 영상 움직임 → 새 영상 재생성 (얼굴교체와 다른 모델 계열) | [14](14-motion-reenactment.md) |
| QC 게이트 | 자동/수동 품질 검수, 3개 엔진 공통 통과 조건 | [03](03-content-pipeline.md), [04](04-faceswap-ugc.md), [14](14-motion-reenactment.md) |
| 자동 포스팅 | 검수 통과 에셋을 SNS API로 발행 | [05](05-auto-posting.md) |

> 콘텐츠 자동화(Axis B)는 서로 다른 모델을 쓰는 **3개의 독립 엔진**(생성/얼굴교체/모션 리타겟팅)으로 구성되며,
> 페르소나 식별자([02](02-model-training.md))·QC 게이트·에셋 스키마([06](06-data-model.md))만 공용입니다.

## 3. 배치 경계 (제안)

| 영역 | 실행 위치 제안 | 이유 |
|---|---|---|
| DM/댓글 수집·응답 스케줄러 | 상시 구동 서버/워커 (경량) | 저지연, 상시성 필요 |
| LLM 추론(의도분석·답변생성) | **자체 호스팅 vLLM**, 소형 GPU 상시 할당 | 페르소나별 LoRA 멀티 서빙, 응답 지연 SLA 확보 ([09](09-llm-serving.md)) |
| ComfyUI 트레이닝·생성 파이프라인 | GPU 워커 (온디맨드/스팟) | 무겁고 배치성, 상시 구동 불필요 |
| 얼굴교체·모션 리타겟팅 배치 | GPU 워커, 트레이닝과 큐 공유 가능 | 서로 다른 엔진이지만 동일한 배치성 GPU 풀 재사용 가능 |
| QC/포스팅 오케스트레이션 | 경량 서버/워크플로우 엔진 | 상태 관리 중심, GPU 불필요 |

> vLLM(상시·저지연)과 ComfyUI 파이프라인(배치·GPU 집약)은 GPU 자원을 두고 경합할 수 있습니다.
> 분리 전략은 [09-llm-serving.md](09-llm-serving.md)의 "GPU 자원 경합" 절 참고.

## 4. 오케스트레이션 큐 구조 (제안)

```mermaid
flowchart LR
    subgraph Queues
        Q1[(reply-jobs)]
        Q2[(training-jobs)]
        Q3[(generation-jobs)]
        Q4[(faceswap-jobs)]
        Q4b[(retarget-jobs)]
        Q5[(qc-review)]
        Q6[(publish-jobs)]
    end
    A1[수집기] --> Q1 --> A4[응답 워커]
    B1[데이터셋 준비] --> Q2 --> B2[트레이닝 워커/GPU]
    B2 --> Q3 --> B3[ComfyUI 워크플로우 워커/GPU]
    B2 --> Q4 --> B4[얼굴교체 워커/GPU]
    B2 --> Q4b --> B4b[모션 리타겟팅 워커/GPU]
    B3 --> Q5
    B4 --> Q5
    B4b --> Q5
    Q5 --> B5{QC 통과?}
    B5 -->|Yes| Q6 --> B6[포스팅 워커]
    B5 -->|No| R[반려/재생성 큐]
```

각 단계는 큐로 분리되어 있어 GPU 파이프라인(트레이닝·생성·얼굴교체·모션 리타겟팅)의 장애가 대화 자동화(상시 응답)에 영향을 주지 않습니다.
