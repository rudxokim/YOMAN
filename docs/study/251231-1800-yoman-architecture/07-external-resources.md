# 07. External Resources (외부 참고 자료)

## 7.1 핵심 논문

### SampleRNN (ICLR 2017)

**제목**: SampleRNN: An Unconditional End-to-End Neural Audio Generation Model

**저자**: Soroush Mehri, Kundan Kumar, Ishaan Gulrajani, Rithesh Kumar, Shubham Jain, Jose Sotelo, Aaron Courville, Yoshua Bengio

**링크**:
- [arXiv](https://arxiv.org/abs/1612.07837)
- [GitHub](https://github.com/soroushmehr/sampleRNN_ICLR2017)
- [OpenReview](https://openreview.net/forum?id=SkxKPDv5xl)

**YOMAN 관련성**:
| SampleRNN 개념 | YOMAN 적용 |
|---------------|-----------|
| 계층적 시간 스케일 | Unit → Module → System |
| Conditioning | 상위 레벨이 하위 가이드 |
| Truncated BPTT | 작은 컨텍스트로 작업 |
| 다른 clock rate | 다른 모델 (Flash/Sonnet/Opus) |

**핵심 인용**:
> "We propose a novel architecture that operates at multiple time scales, with modules at each scale working independently to model different aspects of the audio signal."

---

### GAN (NeurIPS 2014)

**제목**: Generative Adversarial Nets

**저자**: Ian J. Goodfellow, Jean Pouget-Abadie, Mehdi Mirza, Bing Xu, David Warde-Farley, Sherjil Ozair, Aaron Courville, Yoshua Bengio

**링크**:
- [arXiv](https://arxiv.org/abs/1406.2661)
- [NeurIPS Proceedings](https://papers.nips.cc/paper/5423-generative-adversarial-nets)

**YOMAN 관련성**:
| GAN 개념 | YOMAN 적용 |
|---------|-----------|
| Generator | CODINGBOT |
| Discriminator | REVIEWER |
| 적대적 학습 | 품질 검증 피드백 |

**참고**: YOMAN은 적대적 학습을 사용하지 않음. Discriminator를 고정된 품질 게이트로 활용.

---

### WaveNet (arXiv 2016)

**제목**: WaveNet: A Generative Model for Raw Audio

**저자**: Aaron van den Oord, Sander Dieleman, Heiga Zen, Karen Simonyan, Oriol Vinyals, Alex Graves, Nal Kalchbrenner, Andrew Senior, Koray Kavukcuoglu

**링크**:
- [arXiv](https://arxiv.org/abs/1609.03499)
- [DeepMind Blog](https://www.deepmind.com/blog/wavenet-a-generative-model-for-raw-audio)

**YOMAN 관련성**:
- Dilated convolutions → 넓은 receptive field
- YOMAN은 Knowledge Graph로 더 넓은 의존성 처리

---

## 7.2 LLM 코드 생성 관련

### Codex (arXiv 2021)

**제목**: Evaluating Large Language Models Trained on Code

**저자**: Mark Chen, Jerry Tworek, Heewoo Jun, et al. (OpenAI)

**링크**:
- [arXiv](https://arxiv.org/abs/2107.03374)
- [GitHub Copilot](https://github.com/features/copilot)

**YOMAN 차별점**:
| Codex/Copilot | YOMAN |
|---------------|-------|
| Single-shot 생성 | Generator-Discriminator 검증 |
| Inline suggestion | End-to-end task 완료 |
| 사람이 통합 | 자동 통합 |

---

### AlphaCode (Science 2022)

**제목**: Competition-Level Code Generation with AlphaCode

**저자**: Yujia Li, David Choi, Junyoung Chung, et al. (DeepMind)

**링크**:
- [Science](https://www.science.org/doi/10.1126/science.abq1158)
- [DeepMind Blog](https://www.deepmind.com/blog/competitive-programming-with-alphacode)

**YOMAN 관련성**:
- AlphaCode: 대량 생성 → 필터링 → 클러스터링
- YOMAN: 구조화된 생성 → REVIEWER 검증 → 반복

---

### ChatDev (arXiv 2023)

**제목**: Communicative Agents for Software Development

**저자**: Chen Qian, Xin Cong, Wei Liu, et al.

**링크**:
- [arXiv](https://arxiv.org/abs/2307.07924)
- [GitHub](https://github.com/OpenBMB/ChatDev)

**YOMAN 차별점**:
| ChatDev | YOMAN |
|---------|-------|
| 역할 기반 에이전트 | 계층 기반 모델 |
| 대화형 개발 | Bucket 기반 정제 |
| 정성적 검증 | 정량적 match_rate |

---

## 7.3 인지 심리학 참고

### Miller's Law (1956)

**제목**: The Magical Number Seven, Plus or Minus Two

**저자**: George A. Miller

**링크**:
- [Wikipedia](https://en.wikipedia.org/wiki/The_Magical_Number_Seven,_Plus_or_Minus_Two)
- [Original Paper](http://psychclassics.yorku.ca/Miller/)

**YOMAN 적용**:
- 단기 기억: 7±2 항목
- → 한 번에 처리할 태스크 3개로 제한
- → 유닛 크기를 인지 가능한 범위로

---

### Dunbar's Number

**제목**: Neocortex size as a constraint on group size in primates

**저자**: Robin Dunbar

**링크**:
- [Wikipedia](https://en.wikipedia.org/wiki/Dunbar%27s_number)

**YOMAN 적용**:
- 안정적 관계: ~150명
- → 팀 크기 5-9명 권장
- → 모듈당 유닛 개수 제한

---

### Zeigarnik Effect

**설명**: 미완성 작업이 완성된 작업보다 더 잘 기억되는 현상

**링크**:
- [Wikipedia](https://en.wikipedia.org/wiki/Zeigarnik_effect)

**YOMAN 적용**:
- 미완성 작업 = 스트레스
- → 작은 단위로 빠르게 완료
- → 각 유닛이 독립적으로 완료 가능

---

## 7.4 마이크로서비스 아키텍처

### Martin Fowler - Microservices (2014)

**제목**: Microservices

**저자**: Martin Fowler, James Lewis

**링크**:
- [martinfowler.com](https://martinfowler.com/articles/microservices.html)

**YOMAN 적용 원칙**:
| 마이크로서비스 원칙 | YOMAN 적용 |
|-------------------|-----------|
| 단일 책임 | Operation = 하나의 목표 |
| 독립 배포 | 유닛 독립 완료 |
| 느슨한 결합 | 모듈 간 인터페이스 |
| 장애 격리 | 하나의 실패 ≠ 전체 중단 |

---

## 7.5 GitHub 레포지토리

### SampleRNN 공식 구현

```
https://github.com/soroushmehr/sampleRNN_ICLR2017
```

- TensorFlow 구현
- 3-tier 아키텍처 예시
- Conditioning 메커니즘

### PyTorch 구현 (비공식)

```
https://github.com/deepsound-project/samplernn-pytorch
```

- PyTorch 버전
- 더 읽기 쉬운 코드

---

## 7.6 관련 블로그/아티클

### Attention Is All You Need (2017)

**제목**: Attention Is All You Need

**저자**: Ashish Vaswani, Noam Shazeer, Niki Parmar, et al.

**링크**:
- [arXiv](https://arxiv.org/abs/1706.03762)

**관련성**: Transformer가 LLM의 기반. YOMAN이 사용하는 모든 모델의 근간.

---

### Scaling Laws for Neural Language Models (2020)

**제목**: Scaling Laws for Neural Language Models

**저자**: Jared Kaplan, Sam McCandlish, Tom Henighan, et al. (OpenAI)

**링크**:
- [arXiv](https://arxiv.org/abs/2001.08361)

**관련성**: 모델 크기 vs 성능 관계. YOMAN의 다중 모델 전략 근거.

---

## 7.7 도구 및 프레임워크

### Claude API

```
https://docs.anthropic.com/claude/reference/getting-started-with-the-api
```

- Opus, Sonnet, Haiku 모델
- YOMAN의 System/Module level

### Gemini API

```
https://ai.google.dev/docs
```

- Flash 모델
- YOMAN의 Unit level

### Memory MCP (Model Context Protocol)

```
https://modelcontextprotocol.io/
```

- 지식 그래프 저장
- Context 절감 (96%)

---

## 7.8 참고 비디오

### SampleRNN 설명

- [Two Minute Papers - SampleRNN](https://www.youtube.com/watch?v=9GSTtNL-9Rc)
- 계층적 오디오 생성 개요

### Microservices 설명

- [Martin Fowler - Microservices](https://www.youtube.com/watch?v=wgdBVIX9ifA)
- 마이크로서비스 원칙

---

## 요약: 핵심 참고 자료 Top 5

| 순위 | 자료 | 이유 |
|------|------|------|
| 1 | **SampleRNN 논문** | 계층 구조의 직접적 영감 |
| 2 | **GAN 논문** | Generator-Discriminator 패턴 |
| 3 | **Miller's Law** | 인지 한계 기반 설계 |
| 4 | **Martin Fowler - Microservices** | Operation 개념 |
| 5 | **Memory MCP 문서** | Context 최적화 |

---

*다음: [08-references.md](08-references.md) - 내부 문서 및 용어 정리*
