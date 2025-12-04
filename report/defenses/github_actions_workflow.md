# github actions workflow

- workflow (build-push.yaml)
    - Secrets
        
        > G_TOKEN = "PAT token"  
        TEAM_COLOR = "Team Color"
        > 

- 참고 GitHub
    
    [GitHub - cloud-club/08th-slsa-level-up: SLSA 레벨업 프로젝트](https://github.com/cloud-club/08th-slsa-level-up)
    
    [GitHub - mjttong/slsa-study: Cloud Club 8th study: SLSA 공급망 보안](https://github.com/mjttong/slsa-study)
    

- 목표
    - GHCR(GitHub Container Registry)에 Docker 이미지를 빌드·푸시
    - Trivy로 보안 스캔
    - CycloneDX SBOM 생성
    - Sigstore Cosign으로 keyless 서명 및 SBOM attest
    - `k8s/deployment.yaml` 내 이미지 태그 자동 갱신 후 push

- 구성
    1. 트리거
        
        ```yaml
        on:
          push:
            branches: [ main ]
        ```
        
        - main 브랜치에 push 되었을 때, 자동 실행
        
    2. 실행 환경 및 권한 설정
        
        ```yaml
        		runs-on: ubuntu-latest
        
            permissions:
              contents: write   # k8s manifest 수정용 (이미지 갱신)
              id-token: write   # keyless 서명용
              packages: write   # GHCR push 권한
        ```
        
        - `id-token: write`은 Sigstore의 keyless 서명에 필요 (OIDC 기반 서명)
        - `packages: write`은 GHCR 이미지 push에 필요
        
    3. 환경 변수 설정
        
        ```yaml
        		env:
              IMAGE: ghcr.io/${{ github.repository_owner }}/application
        ```
        
        - 이미지  → 해당 repository의 image registry에 application이라는 이름으로 설정
        
    4. 코드 체크아웃 및 빌드 환경 준비
        
        ```yaml
        			- name: Checkout
                uses: actions/checkout@v4
        
              - name: Set up QEMU and buildx
                uses: docker/setup-buildx-action@v3
        ```
        
        - 현재 repository 코드를 runner에 내려받음
        - QEMU + Buildx를 세팅하여 멀티 아키텍쳐 이미지 빌드가 가능하도록 설정
        
    5. GHCR 로그인
        
        ```yaml
        			- name: Login to GHCR
                uses: docker/login-action@v3
                with:
                  registry: ghcr.io
                  username: ${{ github.actor }}
                  password: ${{ secrets.G_TOKEN }}
        ```
        
        - GHCR에 로그인 → G_TOKEN (PAT - packages 권한 포함)
        
    6. Docker image build & push
        
        ```yaml
        			- name: Build and push image
                uses: docker/build-push-action@v6
                id: push
                with:
                  context: .
                  push: true
                  tags: |
                    ${{ env.IMAGE }}:latest
                    ${{ env.IMAGE }}:${{ github.run_number }}
                    ${{ env.IMAGE }}:sha-${{ github.sha }}
                  cache-from: type=gha
                  cache-to: type=gha,mode=max
                  build-args: |
                    TEAM=${{ github.repository_owner }}
                    COLOR=${{ secrets.TEAM_COLOR }}
        ```
        
        - Buildx를 이용해 Docker image build 후 GHCR로 push
            - tag 
            1. `latest` : 최신 버전 관리 
            2. `run_number` : 워크플로 실행 번호 기반 관리
            3. `sha-커밋ID` : 커밋 기반 버전 관리 (배포용)
        - `cache-from / cache-to` : 이전 빌드 캐시(아티팩트) 설정
        - `build-args` : Dockerfile의 argument 주입
        
    7. Trivy 취약점 스캔
        
        ```yaml
        			# Trivy로 빌드된 이미지 scan -> 취약점 발견 시 실패 (보안 게이트 역할)
              - name: Run Trivy vulnerability scan
                uses: aquasecurity/trivy-action@0.30.0
                with:
                  image-ref: '${{ env.IMAGE }}@${{ steps.push.outputs.digest }}'
                  format: 'table'
                  severity: 'CRITICAL,HIGH'
                  exit-code: 1
        ```
        
        - 빌드된 이미지 digest(SHA256 해시)를 기준으로 Trivy scan 수행
        - CRITICAL, HIGH 등급의 취약점 발생 시 exit-code 1을 통해 파이프라인 중단
        
    8. SBOM 생성 (CycloneDX 형식)
        
        ```yaml
              # CycloneDX 포맷으로 SBOM 생성
              - name: Generate SBOM with Trivy
                uses: aquasecurity/trivy-action@0.30.0
                with:
                  image-ref: '${{ env.IMAGE }}@${{ steps.push.outputs.digest }}'
                  scan-type: 'image'
                  format: 'cyclonedx'
                  output: 'sbom.json'
        ```
        
        - 이미지 구성 요소를 CycloneDX 포맷으로 분석해 sbom.json 생성
            - SBOM = Software Bill of Metarials (소프트웨어 구성요소 목록)
            
    9. Cosign 설치 및 서명 (keyless)
        
        ```yaml
              # 이미지 서명 (keyless)
              - name: Sign Container Image
                env:
                  DIGEST: ${{ steps.push.outputs.digest }}
                run: |
                  cosign sign --yes \
                    ${{ env.IMAGE }}@${DIGEST}
        ```
        
        - keyless 모드로 이미지 서명 (OIDC 기반, 개인키 X)
        
    10. SBOM Attestation (증명 첨부)
        
        ```yaml
              # sbom.json 파일을 이미지에 증명용으로 첨부
              # 이미지 내부에 GHCR 메타데이터로 구성요소들이 남아있음
              - name: Attest SBOM
                env:
                  DIGEST: ${{ steps.push.outputs.digest }}
                run: |
                  cosign attest --yes \
                    --predicate sbom.json \
                    --type cyclonedx \
                    ${{ env.IMAGE }}@${DIGEST}
        ```
        
        - 위에서 생성한 SBOM(sbom.json)을 이미지에 메타데이터로 증명 (attestation)
            - GHCR 메타데이터에 SBOM이 연동되어 이미지 구성 투명성 확보
            
    11. Kubernetes manifest 갱신
        
        ```yaml
              - name: Update k8s manifest with new image tag
                run: |
                  git config user.name "github-actions"
                  git config user.email "actions@github.com"
                  sed -i "s|image: .*|image: ${{ env.IMAGE }}:sha-${{ github.sha }}|" k8s/deployment.yaml
                  git add k8s/deployment.yaml
                  git commit -m "ci: update verified image to sha-${{ github.sha }}"
                  git push
        ```
        
        - `k8s/deployment.yaml` 내 이미지 경로를 새로 빌드한 태그로 교체 (`sha-<커밋해시>`)
        - 이후 GitOps 도구(ArgoCD)를 통해 변경 감지 후 자동 배포 연결