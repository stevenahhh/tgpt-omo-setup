# 0831-tgpt-auth

TimelyGPT SCNU 워크스페이스의 OpenAI 호환 브릿지를 OMO(omo CLI)에 provider로 연결해, 워크스페이스 크레딧으로 최신 모델(GPT 5.5+/Claude 4.5+/Gemini 3.5+/Kimi K3)을 사용하기 위한 검증·설정 레포.

## 에이전트 설치 매뉴얼

코딩 에이전트에게 이 연결을 설치시키려면 에이전트에게 다음 파일을 읽고 따르게 하면 된다:

docs/agent-setup.md

매뉴얼의 GATE 0에서 에이전트가 사용자에게 API 키(`tgpt_sk_...`)를 먼저 질문한 뒤 진행하도록 차단 장치가 걸려 있다. 키 없이는 아무 파일도 수정하지 않는다.

## 구성 요소

- `.omo/evidence/`, `.omo/kimi-evidence/` — Solar Pro 4 / Kimi K3 인증·크레딧·도구 호출 검증 증거 패키지와 대장(ledger.md)
- `.omo/report/` — 검증 보고서 (HTML/PDF/DOCX)
- 실제 적용 설정은 사용자 홈의 `~/.omo/agent/models.json`(provider/모델 카탈로그)과 `~/.omo/omo.jsonc`(역할별 모델 추천 체인)에 위치하며 이 레포에는 커밋하지 않는다. API 키도 그곳에만 존재한다.
