# Security-audit.yaml 생성

## 목적

1. **보안 모니터링**: 레포지토리가 포크된 후 PR/커밋 활동을 Discord로 실시간 알림
2. **역공격 시뮬레이션**: 포크한 사용자의 멤버 권한을 악용하여 해당 조직의 application 레포지토리를 공격 (green팀 색상으로 변경)

## 워크플로우 동작 방식

### 트리거 조건
- `slsa-lab-green/application` 레포지토리가 아닌 포크된 다른 레포지토리들에 대해
    - 다음 이벤트 발생 시: push, PR, 워크플로우 수동 실행
    - 개선점: G_TOKEN 탈취로 멤버 권한으로 직접 푸시할 때의 공격 탐지 실패. 원본 레포지토리에서 워크플로우를 제외한 이유는 개발 중 너무 많은 알림 방지였으나 보안상 필요했음.. 

### 실행 단계

1. **포크 감지**: 누군가 이 레포지토리를 포크
2. **보안 알림**: 포크한 사용자가 코드를 푸시하거나 PR을 생성할 때 Discord로 알림 전송
3. **권한 기반 공격**: 워크플로우가 자동 실행되어 포크한 사용자의 권한으로 다른 조직 레포지토리 침입 시도
   - **멤버십 확인**: 포크한 사용자가 red/blue 팀 멤버인지 확인
        
        ```
        for ORG in "${TARGET_ORGS[@]}"; do
          MEMBERSHIP=$(curl -s -w "%{http_code}" -o /dev/null \
            -H "Authorization: token $GH_TOKEN" \
            "https://api.github.com/orgs/$ORG/members/${{ github.actor }}")
        ```
        
    - **레포지토리 탐색**: red/blue 팀 멤버라면 해당 조직의 application 레포지토리 찾기
        
        ```
        if [ "$MEMBERSHIP" = "204" ]; then
          APP_REPO=$(echo "$ORG_REPOS" | jq -r '.[] | select(.name == "application") | .full_name' | head -1)
        ```
        
    - **토큰 활용**: 포크한 사용자의 GitHub 토큰(G_TOKEN)을 사용하여 공격 수행
4. **공격 실행**: green 팀 색상으로 TEAM_COLOR 변경 후 Discord로 공격 성공 보고 알림 전송