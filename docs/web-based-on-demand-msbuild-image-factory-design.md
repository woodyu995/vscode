# 추천 구현안: Web 기반 주문형 MSBuild 이미지 팩토리 (개정)

> 본 문서는 초안 설계를 검토한 뒤, Profile Hash 정규화, 동시성 Lock, 비동기 오케스트레이션, Windows 컨테이너 운영 제약, NuGet/폐쇄망, 보안 MVP 요구사항을 반영한 개정안이다.

## 1. 개요

핵심 구조는 다음과 같다.

> 사용자가 Web에서 필요한 빌드 도구를 선택하고, 동일한 환경 이미지가 내부 Registry에 있으면 재사용하며, 없으면 전용 Image Factory가 최초 한 번 생성·검증·저장한 후 Kubernetes Windows Pod에서 프로젝트를 빌드한다.

프로젝트 빌드 Pod가 시작될 때마다 Visual Studio Build Tools를 설치하는 방식은 사용하지 않는다.

### 1.1 핵심 구현 원칙

> 사용자가 이미지를 선택하는 것이 아니라 필요한 빌드 도구를 선택하고, 시스템이 재현 가능한 이미지 Profile로 변환한다. 이를 통해 사용자 편의성, 폐쇄망 통제, 이미지 재사용, 빌드 재현성을 동시에 확보한다.

### 1.2 개정에서 강화한 계약

| 영역 | 강화 내용 |
|------|-----------|
| Profile Hash | Canonical JSON(RFC 8785) + fixture 테스트 |
| 동시성 | `CREATING` 상태 + DB 락 + waiter/실패 reconcile |
| 오케스트레이션 | 이미지 생성과 프로젝트 빌드 비동기 분리 |
| Windows 운영 | OS 호환, 디스크, Pull SLA, Offline Layout 공급 |
| 조합 폭발 | Allow-list + 쿼터 + Hot/Cold + GC |
| 보안 | MVP부터 Internal Callback 인증 필수 |
| NuGet | 폐쇄망 restore 모델 명시 |

---

## 2. 전체 아키텍처

```text
┌───────────────────────────────┐
│ 사용자 Web Browser            │
│ 프로젝트 및 빌드 환경 선택    │
└───────────────┬───────────────┘
                │ HTTPS + SSO
                ▼
┌───────────────────────────────┐
│ Build Portal (Source of Truth)│
│                               │
│ React Web UI                  │
│ Build Portal API              │
│ PostgreSQL                    │
│ Profile Resolver              │
│ Image Lifecycle Controller    │
│ Reconciliation Worker         │
└───────┬───────────────┬───────┘
        │               │
        │ 이미지 필요    │ 이미지 READY
        ▼               ▼
┌──────────────────┐  ┌────────────────────────────┐
│ Image Factory    │  │ Jenkins Project Build Job  │
│ (전용 Windows)   │  │                            │
│                  │  │ Windows K8s Agent Pod      │
│ Dockerfile 생성  │  │ Checkout / NuGet / MSBuild │
│ Build Tools 설치 │  │ Test / Artifact Publish    │
│ 검증 / Push      │  └─────────────┬──────────────┘
└────────┬─────────┘                │
         │ Callback(HMAC/mTLS)      │ Callback
         ▼                          ▼
┌─────────────────────────────────────────────────┐
│ Portal: build_image / build_request 상태 갱신   │
└─────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│ 내부 Registry / Artifact Repository             │
└─────────────────────────────────────────────────┘
```

### 2.1 책임 경계

| 구성요소 | Source of Truth | 역할 |
|----------|-----------------|------|
| Build Portal | Profile, Image 상태, Build Request | 검증, Hash, Lock, 상태 전이, UI |
| Image Factory | 없음(실행기) | 이미지 빌드·검증·Push 후 Portal Callback |
| Jenkins | 빌드 실행 로그/아티팩트 | **READY 이미지로 프로젝트 빌드만** |
| Registry | 이미지 blob/digest | 보관, GC 대상 |

Jenkins는 Orchestrator 겸용으로 시작하되, **장시간 Image Factory를 `wait: true`로 붙잡지 않는다.** Portal이 상태 머신의 주인이다.

---

## 3. 구성요소별 역할

### 3.1 Build Portal Web

```text
Frontend: TypeScript, React, Vite, MUI
Backend:  FastAPI, PostgreSQL
배포:     Kubernetes Deployment
```

제공 기능:

- 프로젝트 저장소와 Git ref 선택
- 솔루션 경로, Configuration, Platform 선택
- 승인된 빌드 도구 / Preset 선택 (VS 세대별 제약 반영)
- 기존 이미지 존재 여부·예상 대기 안내
- 신규 이미지 생성 진행 상태(SSE)
- Jenkins 빌드 상태·로그·결과물
- 과거 빌드 환경 재사용(DEPRECATED 포함, QUARANTINED 제외)

사용자는 Docker 이미지 이름, Windows 베이스, Dockerfile을 직접 입력하지 않는다.

### 3.2 Jenkins

```text
READY 이미지로 Windows Agent Pod 생성
MSBuild Pipeline 실행
Artifact 수집
Portal Callback으로 상태 전달
```

이미지 미존재 시 Jenkins가 Factory를 동기 대기하지 않는다. Portal이 Factory를 큐잉하고, READY 후 Project Build Job만 트리거한다.

### 3.3 Windows Image Factory

전용 Windows Server VM 또는 전용 Windows Builder 노드.

```text
Image Factory
├─ Docker Engine (Windows containers)
├─ 내부 Registry Push 권한
├─ Visual Studio Offline Layout 접근
├─ .NET/Windows SDK 설치 원본 접근
├─ Dockerfile 템플릿 / 설치·검증 스크립트
└─ 검증용 샘플 프로젝트
```

일반 프로젝트 빌드 Pod에는 이미지 생성·Push 권한을 부여하지 않는다.

### 3.4 내부 Registry

```text
registry.internal/build/
├─ agent-base-ltsc2019
├─ agent-base-ltsc2022
└─ msbuild-profile
   ├─ vs2019-net46-<hash12>
   ├─ vs2022-net48-dotnet8-<hash12>
   └─ vs2022-v143-mfc-<hash12>
```

프로젝트 빌드는 **태그보다 digest**를 사용한다.

---

## 4. Web 빌드 환경 선택

### 4.1 프로젝트 정보

```text
Repository / Git Ref / Solution Path / Configuration / Platform
```

### 4.2 빌드 환경 (사용자 선택)

| 항목 | 선택 | 비고 |
|------|------|------|
| Visual Studio Build Tools | 2019 / 2022 | 단수 |
| .NET Framework Targeting Pack | 복수 | Allow-list |
| .NET SDK | 복수 (논리 버전 6.0, 8.0) | 저장 시 정확한 패치·SHA로 해석 |
| C++ Toolset | 없음 / v142 / v143 | VS 세대 제약 |
| Windows SDK | 복수 | 정확한 빌드 번호 |
| 추가 기능 | Managed Desktop, MFC, ATL, C++/CLI, WiX, 사내 SDK | Allow-list |

### 4.3 시스템이 자동 결정

```text
Windows Server Core 버전
베이스·Agent 이미지 digest
Visual Studio Component ID
Jenkins Agent / JRE 버전
Dockerfile 템플릿
Kubernetes Node Selector / OS 격리 요구
```

예: `VS2019 + .NET Framework 4.6` → `ltsc2019` + `build.company.io/windows-release=ltsc2019`.

### 4.4 Preset과 고급 선택

초기 화면은 검증된 Preset만 노출한다.

```text
표준 .NET Framework 4.6
표준 .NET Framework 4.8
표준 .NET 8 Windows
Visual C++ v142
Visual C++ v143
Visual C++ v143 + MFC/ATL
사용자 정의 (관리자/권한 사용자)
```

완전히 자유로운 조합이 아니라 **서버 Allow-list + 호환 그래프**에 포함된 조합만 허용한다.

---

## 5. Component Catalog

폐쇄망 설치 가능 구성요소를 카탈로그로 관리한다. Web 선택지는 Offline Layout에 실제 포함된 항목으로 제한한다.

```yaml
catalogVersion: "2026.07"

visualStudio:
  "2019":
    windowsBase: ltsc2019
    layoutRelease: vs2019-16.11.54
    allowedComponents:
      - msbuild
      - managed-desktop
      - net46-targeting-pack
      - net462-targeting-pack
      - net472-targeting-pack
      - net48-targeting-pack
      - cpp-v142
      - windows-sdk-19041

  "2022":
    windowsBase: ltsc2022
    layoutRelease: vs2022-17.14.x
    allowedComponents:
      - msbuild
      - managed-desktop
      - net48-targeting-pack
      - dotnet-sdk-8
      - cpp-v142
      - cpp-v143
      - mfc
      - atl
      - cpp-cli
      - windows-sdk-19041
      - windows-sdk-22621

compatibility:
  # UI Options API가 이 그래프를 내려준다
  rules:
    - if: { visualStudio: "2019" }
      deny: [cpp-v143, dotnet-sdk-8]
    - if: { features: ["mfc"] }
      require: [cpp-v142, cpp-v143]  # 둘 중 하나
```

논리 키 → VS Component ID 매핑은 카탈로그에 두고 버전 관리한다.

---

## 6. Build Profile 생성

### 6.1 사용자 요청 (통일 스키마)

배열 필드는 항상 배열로 받는다. (`windowsSdk` 단수 금지)

```json
{
  "visualStudio": "2022",
  "dotnetFrameworks": ["4.8"],
  "dotnetSdks": ["8.0"],
  "cppToolsets": ["v143"],
  "windowsSdks": ["10.0.22621.0"],
  "features": ["mfc", "atl"]
}
```

### 6.2 정규화 규칙

Portal Profile Resolver가 수행한다.

1. Allow-list·호환 그래프 검증 (실패 시 `PROFILE_REJECTED`)
2. 배열 필드 **사전식 정렬·중복 제거**
3. 논리 버전 → 설치 원본 버전 + `installerSha256` 해석
4. VS 세대 → `windowsBase`, `layoutRelease`, Component ID 목록 결정
5. 현재 게시된 `agent-base` digest, base image digest, template/validation/script 버전 삽입
6. `schemaVersion` 부여

### 6.3 정규화된 Profile 예시

```json
{
  "schemaVersion": 1,
  "windowsBase": {
    "name": "ltsc2022",
    "digest": "sha256:base-image-digest"
  },
  "agentBase": {
    "image": "registry.internal/build/agent-base-ltsc2022",
    "digest": "sha256:agent-base-digest"
  },
  "visualStudio": {
    "generation": "2022",
    "layoutRelease": "vs2022-17.14.x",
    "components": [
      "Microsoft.Component.MSBuild",
      "Microsoft.VisualStudio.Workload.ManagedDesktopBuildTools",
      "Microsoft.VisualStudio.Component.VC.Tools.x86.x64",
      "Microsoft.VisualStudio.Component.VC.MFC",
      "Microsoft.VisualStudio.Component.VC.ATL"
    ]
  },
  "dotnetFrameworkTargetingPacks": ["4.8"],
  "dotnetSdks": [
    {
      "version": "8.0.412",
      "installerSha256": "ab..."
    }
  ],
  "cppToolsets": ["v143"],
  "windowsSdks": ["10.0.22621.0"],
  "features": ["atl", "mfc"],
  "templateVersion": "image-template-3",
  "installScriptVersion": "install-scripts-7",
  "validationSuiteVersion": "validation-5"
}
```

---

## 7. Profile Hash (Canonical)

### 7.1 정의

```text
canonicalBytes = JCS(normalizedProfile)   # RFC 8785 JSON Canonicalization Scheme
profileHash    = hex(SHA-256(canonicalBytes))
```

Hash 입력에 반드시 포함:

```text
사용자 선택 구성요소 (정규화 후)
Windows 베이스 이미지 digest
공통 Agent 이미지 digest
Visual Studio Offline Layout 릴리스
.NET SDK 등 설치 파일 SHA-256
Dockerfile 템플릿 버전
공통 설치 스크립트 버전
검증 Suite 버전
schemaVersion
```

### 7.2 Canonicalization 계약

- UTF-8, 키 이름 사전식 정렬, 불필요 공백 없음
- 배열은 **의미상 집합인 필드**는 정렬 후 직렬화 (Resolver가 정렬 보장)
- `null` 필드 생략, 빈 배열은 유지
- 숫자·문자열 이스케이프는 RFC 8785 준수
- 구현체별 fixture: `tests/fixtures/profile-hash/*.json` → 기대 `profileHash`

동일 사용자 선택이라도 base/레이아웃/설치 원본이 바뀌면 새 Hash가 된다. 이는 의도된 동작이다.

### 7.3 이미지 참조

```text
tag:    registry.internal/build/msbuild-profile:vs2022-net48-dotnet8-v143-mfc-83c87a10f35b
digest: registry.internal/build/msbuild-profile@sha256:...
```

빌드 Pod는 digest만 사용한다. 태그 접두(사람이 읽기용) + hash12는 운영 편의용이다.

---

## 8. 데이터 모델

### 8.1 `build_component_catalog`

```text
id, component_type, component_key, display_name, version,
visual_studio_generation, windows_base, installer_sha256,
enabled, administrator_only, sort_order
```

### 8.2 `build_profile`

```text
id, profile_hash (UNIQUE), normalized_profile_json,
canonical_json, display_name, created_by, created_at
```

### 8.3 `build_image`

```text
id, profile_hash (UNIQUE), image_repository, image_tag,
image_digest, status, base_image_digest,
validation_result_json, factory_job_id,
lease_owner, lease_expires_at,
created_at, updated_at, last_used_at, ready_at
```

`status`:

```text
CREATING | VALIDATING | READY | FAILED | DEPRECATED | QUARANTINED | DELETED
```

### 8.4 `build_request`

```text
id, repository, git_ref, resolved_commit, solution_path,
configuration, platform, profile_hash, image_digest,
nuget_mode, jenkins_job_name, jenkins_queue_id, jenkins_build_number,
status, requested_by, requested_at, finished_at, error_code, error_message
```

### 8.5 `build_event`

```text
id, build_request_id, event_type, message, metadata_json, created_at
```

### 8.6 인덱스·제약

```text
UNIQUE(build_profile.profile_hash)
UNIQUE(build_image.profile_hash)
INDEX(build_request.status, profile_hash)
INDEX(build_image.status, last_used_at)
```

---

## 9. 동시성 Lock과 이미지 생성

### 9.1 목표

동일 `profile_hash`에 대해 **이미지 빌드는 최대 1개**. 동시 요청자(B/C)는 A의 결과를 공유한다.

### 9.2 획득 절차 (Portal)

```text
BEGIN;
SELECT * FROM build_image WHERE profile_hash = $h FOR UPDATE;

없으면:
  INSERT status=CREATING, lease_owner=factory-run-id, lease_expires_at=now()+TTL
  → Factory Job 트리거
  → 현재 request를 IMAGE_BUILD_QUEUED

있으면 status=CREATING|VALIDATING:
  → request를 IMAGE_WAITING으로, 폴링/SSE로 공유 대기

있으면 status=READY:
  → image_digest 연결, BUILD_QUEUED

있으면 status=FAILED 이고 retry 가능:
  → CREATING으로 CAS 전이 후 재큐 (backoff)

있으면 DEPRECATED (재현 빌드 허용 시):
  → READY와 동일하게 digest 사용 가능 (정책 플래그)

있으면 QUARANTINED|DELETED:
  → PROFILE/IMAGE 거부
COMMIT;
```

대안 구현: PostgreSQL advisory lock (`hashtextextended(profile_hash, 0)`) + 위 상태 머신. Unique 제약으로 이중 INSERT를 방지한다.

### 9.3 Lease / 실패 / Reconcile

| 상황 | 동작 |
|------|------|
| Factory 정상 완료 | VALIDATING → READY, digest 기록, waiter request들을 BUILD_QUEUED |
| Factory 실패 | FAILED, `lease` 해제, 실패 이벤트, 재시도 횟수+1 |
| Lease TTL 초과 | Reconciliation Worker가 Factory/Jenkins 실상태 조회 후 FAILED 또는 lease 연장 |
| Portal Callback 유실 | Reconciliation이 Factory 결과·Registry digest로 보정 |

권장 TTL: Image Factory 예상 상한의 1.5배 (예: 예상 90분이면 lease 135분). 진행 heartbeat Callback으로 연장.

### 9.4 Waiter UX

- SSE: `IMAGE_BUILDING` / `IMAGE_VALIDATING` / `READY` / `FAILED`
- 예상 소요: Catalog의 `estimatedImageBuildMinutes` + 최근 이동평균
- 사용자는 요청 취소 가능 → request만 `CANCELLED`, 진행 중 Factory는 계속(다른 waiter가 있으면). waiter 0이고 정책이 허용하면 Factory도 취소.

---

## 10. Build Request 상태 머신

### 10.1 누가 무엇을 하나

| 상태 | 주체 | 설명 |
|------|------|------|
| REQUESTED | Portal | API 접수 |
| VALIDATING_PROFILE | Portal (동기) | Catalog·Allow-list·워커 존재 여부 |
| RESOLVING_IMAGE | Portal | Hash·DB 조회·Lock |
| IMAGE_BUILD_QUEUED | Portal | Factory 트리거됨 |
| IMAGE_BUILDING | Factory Callback | 빌드 중 |
| IMAGE_VALIDATING | Factory Callback | Smoke/샘플 |
| IMAGE_WAITING | Portal | 다른 요청의 생성 대기 |
| BUILD_QUEUED | Portal | Jenkins Project Build 트리거 |
| BUILDING / TESTING / PUBLISHING | Jenkins Callback | 프로젝트 파이프라인 |
| SUCCEEDED | Jenkins Callback | 완료 |

실패:

```text
PROFILE_REJECTED
IMAGE_BUILD_FAILED
IMAGE_VALIDATION_FAILED
PROJECT_BUILD_FAILED
TEST_FAILED
CANCELLED
```

### 10.2 전이 다이어그램

```text
REQUESTED
    │  Portal 동기 검증
    ▼
VALIDATING_PROFILE
    │
    ▼
RESOLVING_IMAGE
    ├─ READY/DEPRECATED(허용) ──────────────► BUILD_QUEUED
    ├─ CREATING(타 요청) ───────────────────► IMAGE_WAITING ──► BUILD_QUEUED
    └─ 없음/재시도 ─────────────────────────► IMAGE_BUILD_QUEUED
                                                ▼
                                          IMAGE_BUILDING
                                                ▼
                                          IMAGE_VALIDATING
                                                ▼
                                            BUILD_QUEUED
                                                ▼
                                             BUILDING → TESTING → PUBLISHING → SUCCEEDED
```

**중요:** `VALIDATING_PROFILE`은 Jenkins 진입 전 Portal에서 끝낸다. 잘못된 조합으로 Factory/Jenkins를 돌리지 않는다.

---

## 11. 비동기 실행 시나리오

### 11.1 사용자 요청

```json
{
  "project": {
    "repository": "ProductClient",
    "gitRef": "release/2.1",
    "solutionPath": "ProductClient.sln",
    "configuration": "Release",
    "platform": "x64"
  },
  "environment": {
    "visualStudio": "2022",
    "dotnetFrameworks": ["4.8"],
    "dotnetSdks": ["8.0"],
    "cppToolsets": ["v143"],
    "windowsSdks": ["10.0.22621.0"],
    "features": ["mfc", "atl"]
  },
  "nuget": {
    "mode": "repo-packages-and-internal-feed"
  }
}
```

### 11.2 Portal

1. SSO·권한 확인  
2. Profile 검증·정규화·Canonical Hash  
3. `build_request` 생성  
4. Image Lock/Resolve  
5. 필요 시 Factory 비동기 큐잉  
6. READY면 Jenkins `msbuild-project-build`만 트리거  

### 11.3 이미지 이미 있는 경우

```text
RESOLVING_IMAGE → BUILD_QUEUED → BUILDING → … → SUCCEEDED
```

### 11.4 이미지 없는 경우 (비동기)

```text
Portal                Factory                 Jenkins
  │                      │                       │
  ├─ CREATING / queue ──►│                       │
  │◄── HEARTBEAT/이벤트──┤                       │
  │◄── READY + digest ───┤                       │
  ├─ waiter들 BUILD_QUEUED ─────────────────────►│
  │◄────────────── BUILD events ─────────────────┤
```

Orchestrator Job이 Factory를 `wait: true`로 붙잡지 않는다.

---

## 12. Jenkins Job 구조

### 12.1 Job: `msbuild-project-build` (Portal이 주로 호출)

파라미터: `BUILD_REQUEST_ID`

```text
Portal에서 request·digest·nodeSelector 조회
Windows Kubernetes Pod 생성 (digest)
Checkout exact commit
NuGet restore (정책 모드)
MSBuild / Test / Publish
Portal Callback
```

### 12.2 Job: `msbuild-image-factory` (Portal 또는 Factory 컨트롤러가 호출)

파라미터: `BUILD_REQUEST_ID`, `PROFILE_HASH`, `FACTORY_LEASE_ID`

```text
정규화 Profile 조회
Dockerfile·.vsconfig 생성
Windows 이미지 빌드
Smoke Test
Registry Push
Portal에 digest Callback
```

### 12.3 (선택) 과도기 Orchestrator

기존 Jenkins 중심 운영을 유지해야 하면 `msbuild-orchestrator`를 둘 수 있으나, Ensure Image 단계는 **비동기 폴링/재큐**만 하고 장시간 `wait: true`는 금지한다.

```groovy
// Ensure Image: 금지 패턴
// build(job: 'msbuild-image-factory', wait: true)

// 권장: Portal이 이미 READY인지 확인하고, 아니면 파이프라인을 종료(또는 park) 후
// Portal Callback이 새 build job을 트리거
```

---

## 13. 주문형 이미지 생성

### 13.1 공통 Agent 베이스

```text
agent-base-ltsc2019 / agent-base-ltsc2022
├─ Windows / .NET Framework Runtime
├─ JRE, Git, PowerShell
├─ Jenkins Agent 요구 환경
├─ 사내 CA 인증서
└─ 공통 빌드 스크립트
```

### 13.2 파생 Dockerfile

```dockerfile
# escape=`

ARG BASE_IMAGE
FROM ${BASE_IMAGE}

SHELL ["cmd", "/S", "/C"]

COPY profile.vsconfig C:\ImageBuild\profile.vsconfig
COPY scripts C:\ImageBuild\scripts

RUN C:\ImageBuild\scripts\Install-BuildEnvironment.cmd

RUN powershell.exe -NoProfile -NonInteractive `
    -File C:\ImageBuild\scripts\Validate-Environment.ps1

LABEL company.build.profile-hash="83c87a10..."
LABEL company.build.template-version="image-template-3"
LABEL company.build.layout-release="vs2022-17.14.x"
```

Offline Layout은 가능하면 **빌드 컨텍스트 COPY 대신 읽기 전용 마운트/캐시 볼륨**으로 공급해 컨텍스트 폭증을 막는다. 네트워크 공유 경로와 체크섬은 Catalog에 고정한다.

---

## 14. 이미지 검증

생성 직후 바로 `READY`로 두지 않는다.

1. `vswhere`, `MSBuild -version`, `dotnet --list-sdks`
2. Targeting Pack 경로 존재
3. `cl.exe` 등 C++ 도구 (해당 Profile만)
4. Profile별 샘플 솔루션 빌드

성공 시에만 Registry에 최종 태그/digest를 게시하고 `READY`로 전이한다. 실패 시 `FAILED` 또는 `QUARANTINED`.

---

## 15. Kubernetes Windows Pod와 운영 제약

### 15.1 Pod 스펙 요지

```yaml
apiVersion: v1
kind: Pod
spec:
  os:
    name: windows
  nodeSelector:
    kubernetes.io/os: windows
    build.company.io/windows-release: ltsc2022
    build.company.io/purpose: msbuild
  tolerations:
    - key: build.company.io/windows
      operator: Equal
      value: "true"
      effect: NoSchedule
  containers:
    - name: builder
      image: registry.internal/build/msbuild-profile@sha256:...
      imagePullPolicy: IfNotPresent
      workingDir: C:\workspace
      resources:
        requests:
          cpu: "2"
          memory: 4Gi
          ephemeral-storage: 20Gi
        limits:
          cpu: "8"
          memory: 16Gi
          ephemeral-storage: 80Gi
  imagePullSecrets:
    - name: internal-registry-secret
```

### 15.2 필수 운영 요구사항 (설계에 포함)

| 항목 | 요구 |
|------|------|
| OS 호환 | 컨테이너 LTSC ↔ 노드 LTSC 매칭. process/Hyper-V 격리 정책을 노드 풀별로 문서화 |
| 디스크 | 노드·Factory에 이미지당 여유 (예: 프로필 이미지 40–80Gi+ 레이어). ephemeral-storage 모니터링 |
| Pull SLA | Cold Pull 목표(예: 15–30분)와 타임아웃. Hot Profile은 Pre-pull |
| Factory 용량 | 동시 이미지 빌드 수 제한(예: 1–2). 큐잉은 Portal |
| Layout 공급 | Offline Layout 공유 저장소 가용성·체크섬 검증 |
| 라이선스 | VS Build Tools 컨테이너 설치·사용 정책을 법무/구매와 합의 |
| 네트워크 | 내부 Registry DNS, 사내 CA, NuGet 피드 도달성 |

Portal은 요청 검증 시 **해당 `windows-release` 노드가 Ready인지** 확인한다. 노드 없으면 `PROFILE_REJECTED` 또는 재시도 가능 에러 코드로 안내한다.

---

## 16. NuGet / 폐쇄망 Restore

빌드 Pod는 인터넷에 나가지 않는다고 가정한다.

### 16.1 지원 모드

| mode | 설명 |
|------|------|
| `repo-packages` | 저장소 `packages/` 또는 vendored assets만 사용 |
| `internal-feed` | 사내 NuGet 피드(HTTPS + 사내 CA). credentials는 K8s Secret/Jenkins credential |
| `repo-packages-and-internal-feed` | 둘 다 (권장 기본값) |

`NuGet.config`는 프로젝트에 포함하거나 Portal이 요청별로 **승인된 템플릿**을 주입한다. 임의 외부 URL 추가는 거부한다.

### 16.2 파이프라인

```text
checkout
→ nuget/dotnet restore (offline/internal only)
→ msbuild
→ test
→ publish artifacts (bin, binlog, test results)
```

---

## 17. Portal API

### 17.1 선택 항목 조회 (제약 그래프 포함)

```http
GET /api/v1/build-environment/options
```

```json
{
  "catalogVersion": "2026.07",
  "presets": [
    {
      "id": "managed-net48",
      "displayName": "표준 .NET Framework 4.8",
      "environment": {
        "visualStudio": "2022",
        "dotnetFrameworks": ["4.8"],
        "dotnetSdks": [],
        "cppToolsets": [],
        "windowsSdks": [],
        "features": ["managed-desktop"]
      }
    }
  ],
  "visualStudios": [
    {
      "id": "2019",
      "windowsBase": "ltsc2019",
      "allowed": {
        "dotnetFrameworks": ["4.6", "4.6.2", "4.7.2", "4.8"],
        "dotnetSdks": [],
        "cppToolsets": ["v142"],
        "windowsSdks": ["10.0.19041.0"],
        "features": ["managed-desktop", "mfc", "atl"]
      }
    },
    {
      "id": "2022",
      "windowsBase": "ltsc2022",
      "allowed": {
        "dotnetFrameworks": ["4.8"],
        "dotnetSdks": ["6.0", "8.0"],
        "cppToolsets": ["v142", "v143"],
        "windowsSdks": ["10.0.19041.0", "10.0.22621.0"],
        "features": ["managed-desktop", "mfc", "atl", "cpp-cli"]
      }
    }
  ],
  "compatibilityRules": [
    {
      "when": { "featuresIncludes": ["mfc"] },
      "requireAny": { "cppToolsets": ["v142", "v143"] }
    }
  ],
  "estimatedImageBuildMinutes": {
    "hot": 0,
    "coldAverage": 75
  }
}
```

UI는 VS 선택 변경 시 하위 옵션을 클라이언트에서 필터링하고, 서버 `validate`로 최종 확인한다.

### 17.2 Profile 사전 검증

```http
POST /api/v1/build-environment/validate
```

```json
{
  "valid": true,
  "profileHash": "83c87a10f35b...",
  "imageStatus": "NOT_CREATED",
  "action": "IMAGE_CREATION_REQUIRED",
  "estimatedWaitMinutes": 75,
  "warnings": []
}
```

### 17.3 빌드 요청 / 상태 / SSE

```http
POST /api/v1/build-requests
GET  /api/v1/build-requests/{id}
GET  /api/v1/build-requests/{id}/events   # SSE
```

### 17.4 Internal Callback (MVP 필수 인증)

```http
POST /internal/v1/build-events
POST /internal/v1/images/{profileHash}/status
```

인증 (하나 이상 필수):

- mTLS (Jenkins/Factory → Portal)
- 또는 HMAC-SHA256 (`X-Timestamp` + body, 재생 공격 방지 윈도우 5분)
- 서비스 계정 IP allow-list (보조)

Callback 예:

```json
{
  "requestId": "br-20260715-000142",
  "eventType": "IMAGE_VALIDATING",
  "message": "빌드 환경 이미지 검증 중",
  "jenkinsBuildNumber": 1521,
  "leaseId": "factory-run-98"
}
```

Portal은 Callback을 기본으로 쓰고, 유실 대비 Jenkins/Factory/Registry를 주기적으로 조회하는 Reconciliation Worker를 둔다.

---

## 18. 보안 정책

### 18.1 사용자 제공 불가

```text
임의 Dockerfile / RUN / PowerShell
임의 Installer 경로 / 외부 URL
Registry Push 대상 / Image Tag
Kubernetes Node Selector
NuGet 임의 피드 URL
```

### 18.2 사용자 허용

```text
승인된 VS 세대, Targeting Pack, .NET SDK, C++ Toolset, Windows SDK, 기능
프로젝트 빌드 파라미터, 승인된 NuGet mode
```

### 18.3 권한 분리

```text
Build Portal     → Factory 큐잉, Jenkins build 트리거, SSO
Project Build    → 이미지 Pull, Artifact 업로드
Image Factory    → 이미지 Build/Push, Offline Layout 읽기
```

### 18.4 MVP 보안 최소선

- Portal SSO + RBAC (일반 / 고급 환경 선택 / 관리자)
- Internal Callback HMAC 또는 mTLS
- Registry pull/push 계정 분리
- 감사 로그: 누가 어떤 profileHash로 이미지를 만들었는지

이미지 서명·SBOM은 운영 고도화 단계에서 추가하되, digest 고정을 전 단계에서 강제한다.

---

## 19. 이미지 수명주기와 조합 폭발 통제

### 19.1 상태

```text
READY        → 신규·재사용 가능
DEPRECATED   → 신규 선택 불가, 재현 빌드만 (정책)
QUARANTINED  → 실행 금지
DELETED      → Registry 제거 (감사 기간 후)
```

### 19.2 Hot / Cold

```text
Hot Profile  → 사전 생성, 노드 Pre-pull, 캐시 유지
Cold Profile → 최초 요청 시 생성, 미사용 시 노드 캐시만 제거
```

### 19.3 쿼터·GC (수치 mid 기본값, 환경에 맞게 조정)

| 정책 | 기본값 |
|------|--------|
| 동시 CREATING 전역 | Factory 슬롯 수 (예: 2) |
| 사용자당 Cold 신규 생성 / 일 | 예: 3 (관리자 제외) |
| 고급 조합 생성 | 관리자 승인 또는 사전 Allow-list PR |
| Registry 보존 | 감사/재현 N일 (예: 180일) |
| GC 후보 | `last_used_at` 경과 + Hot 아님 + READY/DEPRECATED |
| 노드 디스크 알람 | 사용률 80% |

### 19.4 agent-base 갱신과 Blast Radius

`agentBase.digest`가 Hash에 포함되므로 JRE/Agent/CA 갱신은 **모든 파생 Profile을 무효화**한다.

롤아웃 전략:

1. 새 `agent-base`를 병행 게시 (digest B)
2. Catalog를 B로 전환 → 신규 요청만 새 Hash
3. Hot Profile을 배치로 재빌드·Pre-pull
4. 구 digest는 DEPRECATED 유지 후 보존 기간 뒤 GC
5. 긴급 보안 패치만 강제 Quarantine + 일괄 재빌드

운영 변경 창과 예상 재빌드 시간을 변경 관리에 포함한다.

---

## 20. 인터넷망에서 준비할 산출물

```text
base-images/
visual-studio-layouts/
installers/
image-factory/   (templates, catalog, scripts, validation projects)
jenkins/         (plugins, shared-library, job defs)
manifest/        (checksums, digests, release metadata)
tests/fixtures/profile-hash/   # Canonical Hash 회귀
```

---

## 21. 단계별 구현 순서 (개정)

### 21.1 0단계: 기반 계약 (MVP와 병렬 착수, 완료 게이트)

```text
Canonical Profile Hash + fixture
CREATING Lock + lease + reconcile
Internal Callback 인증 (HMAC/mTLS)
Options API 제약 그래프
Preset 이미지 사전 빌드·Pre-pull
Windows 노드 풀·디스크·OS 호환 검증
```

### 21.2 1단계: MVP

```text
Web Portal + SSO/RBAC
고정 Preset 4개 (사전 빌드된 READY 이미지만 사용)
Profile Hash
Jenkins Project Build + Windows Pod
NuGet internal/repo 모드
SSE 상태
```

초기 Preset: `managed-net46`, `managed-net48`, `managed-dotnet8`, `native-v143-mfc`

이 단계에서는 **주문형 Factory를 열지 않는다.** 없는 Profile은 거부하거나 관리자 티켓으로만 추가한다.

### 21.3 2단계: 주문형 Image Factory

```text
미존재 감지, Dockerfile 자동 생성
Offline Layout 설치, Smoke Test, Push
중복 생성 Lock, waiter SSE, 실패 재시도
예상 대기 시간 안내
```

### 21.4 3단계: 고급 사용자 선택

```text
개별 Targeting Pack / SDK / Toolset / Windows SDK / MFC·ATL·C++/CLI
일일 쿼터·승인 게이트
```

### 21.5 4단계: 운영 고도화

```text
Deprecated/Quarantine, 보안 업데이트 롤아웃
Worker Pre-pull 자동화, 사용량 GC
SBOM·이미지 서명·Audit
Portal 실시간 로그 연동
```

---

## 22. 최종 권장 구조

```text
사용자
→ Web에서 승인된 빌드 도구 선택

Build Portal (Source of Truth)
→ 선택 검증 · Profile 정규화 · Canonical Hash
→ Image Lock/Resolve · 상태 머신 · Callback 수신

이미지 READY
→ Jenkins가 digest로 Windows Pod 빌드

이미지 없음
→ 전용 Image Factory가 비동기로 1회 생성·검증·Push
→ waiter 공유
→ READY 후 Project Build 트리거

Windows Build Pod
→ Checkout → 승인된 NuGet restore → MSBuild/Test → Artifact → Pod 삭제
```

핵심 구현 원칙은 변함없다. 개정안의 차이는 **정확한 Hash/Lock 계약, 장시간 작업을 파이프라인에 묶지 않는 비동기 경계, Windows·폐쇄망 운영 제약을 설계에 명시한 것**이다.
