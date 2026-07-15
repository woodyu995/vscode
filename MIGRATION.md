# 저장소 이전

이 프로젝트는 `woodyu995/vscode`가 아닌 **`woodyu995/msbuild-factory`** 로 이전할 예정입니다.

## 차단 사유

현재 Cloud Agent GitHub App 토큰은 설치된 저장소(`woodyu995/vscode`)에만 접근할 수 있으며, **새 저장소 생성/이름 변경 권한이 없습니다.**

## 필요 작업 (저장소 소유자)

1. GitHub에서 빈 저장소 생성: https://github.com/new
   - Owner: `woodyu995`
   - Repository name: `msbuild-factory`
   - README 없이 빈 저장소로 생성
2. Cursor GitHub App 설치 대상에 `msbuild-factory` 추가  
   (https://github.com/settings/installations → Cursor → Repository access)
3. 이 Agent에 「msbuild-factory로 push」를 다시 요청

준비된 초기 커밋 번들: `msbuild-factory.bundle`  
내용: README, `.gitignore`, 개정 설계서 (`docs/...`)
