# 공급망 보안 시뮬레이션 리포트

- 팀 이름: GREEN
- 팀원: 권민정, 정유정, 허완
- 제출일: 2025-10-25 (KST)

---

## 1. 요약
- 공격 성공 횟수: 2회  
- 방어 성공 횟수: 3회  
- 주요 성과 요약: Kyverno를 통해 SLSA 서명 검증과 SBOM 검증으로 악성 이미지 배포를 차단했고, NKS API 취약점을 발견해 다른 팀 클러스터 UUID를 탈취하여 제어권을 획득했다. 우리 클러스터가 공격당해 ArgoCD 권한이 탈취되었을 때는 새로운 애플리케이션 배포로 우회하여 해결했다.
- 총평: 공용 토큰 사용 시 불필요한 권한 부여를 방지하고, 서비스 간 명확한 권한 분리 및 세분화된 접근 정책을 적용해야 함을 배웠음.

---

## 2. 공격 기록 (Attacks)

| # | 대상 팀 | 시작 (KST) | 종료 (KST) | 방법 요약 (한줄) | 결과 (성공/실패) | 증거 | 상세 보고서 |
|--:|:--------|:-----------:|:----------:|:-----------------|:-----------------:|:-----|:-----------|
| 1 | BLUE   | 2025-10-24 13:10 | 2025-10-24 13:15 | 공용 G_TOKEN을 활용한 권한 획득 | ✅ 성공 | `/screenshots/att1.png` | [공용 G_TOKEN을 활용한 권한 획득](./attacks/get_authority_with_gtoken.md) |
| 2 | RED   | 2025-10-23 04:15 | 2025-10-23 04:20 | 클러스터 UUID 탈취 및 제어권 획득 | ✅ 성공 | `/screenshots/att2.png` | [클러스터 UUID 탈취](./attacks/cluster-uuid-exfiltration.md) |

---

## 3. 방어 기록 (Defenses)

| # | 공격자 팀 | 탐지 여부 | 탐지 시각 (KST, 빈칸 가능) | 차단 여부 | 차단/복구 시각 (KST) | 차단 방법 / 메모 | 증거 | 상세 보고서 |
|--:|:----------|:---------:|:-------------------------:|:---------:|:--------------------:|:------------------:|:-----|:-----------|
| 1 | BLUE      | ✅ 탐지   |      2025-10-22 21:12                    | ✅ 차단   | 2025-10-22 21:48     | Kyverno 정책 적용  | [kyverno log](/defenses/resource/keverno.log) | [CI 검증 파이프라인](./defenses/CI-verfication-pipeline.md), [GitHub Actions 워크플로우 설정](./defenses/github_actions_workflow.md), [워크플로우 보호 정책 설정](./defenses/set_protect_workflow_dierctory_rule.md) |
| 2 | BLUE      | ✅ 탐지   | 2025-10-22 22:49         | ❌ 미차단 |  2025-10-23 05:--    | NKS API를 통한 복구  | [argocd log](/defenses/resource/argocd.log) | [Blue 팀 공격 대응 보고서](./defenses/blue-attack-response.md) |
| 3 | RED       | ✅ 탐지   | 2025-10-25 17:--         | ✅ 차단   |  2025-10-23 17:--    | deployment 복구  | [argocd log](/defenses/resource/argocd.log) | [ArgoCD Discord 알림 연동](./defenses/argocd_discord_notification.md) |

### 기타 방어 설정 문서
- [Security Audit 워크플로우](./defenses/security-audit.md)

---

## 4. 인사이트 & 개선안
- 공격 측면:
    - 공용 Github Token을 이용해서 상대 팀의 이미지 레지스트리와 깃허브에 권한을 쉽게 얻을 수 있었음
    - NCloud Access&Secret key를 활용해 상대방 클러스터 정보 및 권한을 쉽게 얻고 수정할 수 있었음
- 방어 측면
    - cosign 서명 및 sbom subject 검증 정책이 매우 효과적이었음
    - 워크플로우 삽입 공격을 차단한다면, 이미지 서명 검증을 통해 해당 공격으로부터 방어할 수 있음
    - 클러스터 권한 탈취 시 새로운 배포를 통한 우회 전략으로 service availabilty 확보함. (새 배포 전 트래픽을 분리하고 점진적으로 롤아웃한다면 매우 안정적인 서비스 구축이 가능해짐)
