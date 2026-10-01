# dobot-e6-smolvla

흡착 그리퍼를 단 Dobot E6로 Pick-and-Place를 수행하는 시스템입니다. 어떤 물체를 집을지는 AI가 판단하고, 실제 동작은 SmolVLA가 생성합니다.

- **AI**: 판단 주체. 어떤 물체를 집을지 결정합니다.
- **SmolVLA**: AI가 정한 Task를 Action Chunk로 바꾸는 실행 Policy입니다.
- **Agent**: Dobot E6. Action Chunk를 실제로 수행합니다.

## 문서
- [설계 리뷰: 문제점과 해결 방법](docs/design_review.md)
