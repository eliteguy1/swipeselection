# swipeselection docs

## 문제 3건 (2026-10-15, iPhone 11 / iOS 18.7.1 / RootHide)

| # | 증상 | 원인 | 수정 | 상태 |
|---|------|------|------|------|
| 1 | 백스페이스 1번에 2글자 삭제 | `touchesEnded`의 수동 `[kb handleDelete]` 재발사가 시스템 자체 삭제와 중복 (iOS 18은 레이아웃/`handleDeleteWithNonZeroInputCount` 경로로 알아서 1개 삭제) | `src/Tweak.x`에서 재발사 블록 삭제 (2.0.1) | 해결, 2.0.3에서 동작 확인 |
| 2 | 2.0.1 설치 후 리스프링 → 세이프모드 | 동봉한 0바이트 `.roothidepatch` 마커가 위조 상태(패치 없이 도장만) → `roothidepatch.dylib` 로드 콜백에서 `abort_with_payload` → SpringBoard 사망. 후크 코드는 무죄 (로그에 트윅 프레임 없음, 후크 집합은 정상 2.0의 부분집합) | 마커 제거 (2.0.3) | 해결, 리스프링 정상 확인 |
| 3 | 2.0.1 삭제 중 `apt-get` abort | 삭제 과정 `DoInstall`/`writeData:` 예외. 세이프모드와 별개 사건 (제거 프로그램 문제) | 해당 없음 (제거는 완료됨) | 참고용, 본 트윅 코드와 무관 |

## 파일

- `double-delete-fix.md` — 6섹션 상세 보고서 (분석·명령·코드변경·빌드·산출물·리스크 + §7 2.0 후크 테이블 + §8 세이프모드 로그 분석).
- 교차 프로젝트 교훈: `D:\llama agent\docs\roothide-theos-build-and-verification.md` §5 (마커 자작 금지).
