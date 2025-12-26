# 프로젝트 사용 가이드

## 1. 사용법

### 1-1. 취약점 점검 방법 및 결과 확인 방법
프로젝트에 포함된 라이브러리들의 보안 취약점(CVE)을 점검하는 절차입니다.

**1. 점검 실행:**
터미널에서 다음 명령어를 실행하세요.
(API 키는 `gradle.properties`에 이미 설정해 두었습니다. 빠른 속도로 점검이 진행될 것입니다.)

```bash
gradle dependencyCheckAnalyze
```

**2. 결과 확인:**
명령어 실행이 완료되면(BUILD SUCCESSFUL), 아래 경로에 HTML 보고서가 생성됩니다.
- **보고서 경로:** `build/reports/dependency-check/dependency-check-report.html`
- 이 파일을 크롬 등 웹 브라우저로 열면, 발견된 취약점 리스트와 위험도(Severity)를 확인할 수 있습니다.

### 1-2. 정의한 Jar 파일 다운로드 방법
`build.gradle`에 정의된 외부 라이브러리들을 다운로드하는 방법입니다.

기존 레거시 방식처럼 Jar를 일일이 구해올 필요가 없습니다. 다음 명령 중 하나만 실행하면 Gradle이 **Maven Central** 저장소에서 자동으로 필요한 Jar 파일들을 다운로드하여 로컬 캐시(`~/.gradle/caches`)에 저장하고 프로젝트에 연결합니다.

```bash
# 의존성 목록을 확인하면서 다운로드 수행
gradle dependencies

# 또는, 전체 프로젝트를 빌드 (다운로드 + 컴파일 + WAR생성)
gradle build
```

---

## 2. 해당 프로젝트 동작 원리 및 설명

이 프로젝트는 기존에 Jar 파일들을 `WEB-INF/lib` 폴더에 직접 넣어 관리하던 **수동 방식**을, **Gradle 빌드 시스템**으로 전환한 것입니다.

### 2-1. 하이브리드 의존성 관리 (Hybrid Matrix)
라이브러리의 출처에 따라 두 가지 방식으로 나누어 관리합니다.

1.  **공개 라이브러리 (Public Libraries)**:
    - Netty, Apache Commons, Log4j 와 같이 인터넷(Maven Central)에 공개된 표준 라이브러리입니다.
    - `build.gradle` 파일에 `implementation '그룹:이름:버전'` 형식으로 한 줄만 적으면, Gradle이 알아서 다운로드하고 중복된 하위 의존성까지 관리해 줍니다.
    - **다운로드 위치**: `~/.gradle/caches/modules-2/files-2.1/` (시스템 공용 캐시)
    - **내 눈으로 보고 싶다면?**: 아래 명령어를 치면 `download` 폴더에 JAR 파일을 다 꺼내줍니다.
      ```bash
      gradle copyDependencies
      ```
    - 예: `netty-codec`, `netty-transport` 등을 따로 받을 필요 없이 `netty-all` 하나만 선언하면 해결됨.

2.  **로컬/상용 라이브러리 (Local/Private Libraries)**:
    - Coreframe, CubeOne, Transkey 등 메이븐 저장소에 없는 상용 솔루션이나 내부 라이브러리입니다.
    - 프로젝트 루트의 `libs` 폴더에 이 파일들을 넣어두고, Gradle이 이를 참조(`files('libs/...')`)하도록 설정했습니다.

### 2-2. 보안 취약점 점검 원리 (DevSecOps)
- **OWASP Dependency Check 플러그인**을 적용했습니다.
- 이 도구는 프로젝트에 연결된 모든 Jar 파일(공개 및 로컬 포함)의 디지털 서명과 버전을 분석합니다.
- 분석된 정보를 **NVD(National Vulnerability Database)** 와 대조하여, 현재 사용 중인 버전에 알려진 보안 취약점이 있는지 자동으로 검사합니다.
- 제공해주신 API Key는 `gradle.properties` 파일에 저장하여, 체크 속도를 높이도록 설정했습니다.

### 2-3. 요약
- **나(개발자)** 는 `build.gradle` 명세서만 관리하면 됩니다.
- **Gradle** 은 이 명세서를 보고 인터넷에서 라이브러리를 가져오거나 `libs` 폴더를 참조하여 프로그램을 빌드합니다.
- **Dependency Check** 는 빌드 과정에서 사용된 라이브러리들이 안전한지 검사합니다.

---

## 3. 취약점 발견 시 조치 방법 (Remediation)

리포트에서 빨간색(High/Critical) 항목이 나왔다면 이렇게 조치하세요.

### 방법 1: 버전 올리기 (가장 기본)
가장 쉬운 방법은 취약점이 해결된 **최신 버전으로 판올림**하는 것입니다.
1. [Maven Central](https://mvnrepository.com/)에서 해당 라이브러리를 검색합니다. (예: `commons-fileupload`)
2. 버전 리스트가 나오는 표를 확인합니다.
    - **Vulnerabilities** 컬럼에 빨간색 숫자가 있으면 취약한 버전입니다.
    - 이 컬럼이 **비어있는(빈칸)** 가장 최신 버전을 찾으세요. 그게 안전한 버전입니다.
3. `build.gradle`에서 버전 숫자를 수정합니다.

```groovy
// 수정 전 (취약함)
implementation 'commons-fileupload:commons-fileupload:1.3.3'

// 수정 후 (해결됨)
implementation 'commons-fileupload:commons-fileupload:1.5'
```

### 방법 2: 몰래 들어온 손님 내쫓기 (Transitive Dependency)
내가 직접 추가하지 않았는데, 다른 라이브러리가 몰래(의존성으로) 취약한 옛날 버전을 가져오는 경우가 많습니다. (예: `commons-beanutils-1.9.4`)

이럴 땐 **"강제 버전 지정"** 기능을 사용합니다. `build.gradle` 맨 아래에 이 코드를 추가하세요.

```groovy
configurations.all {
    resolutionStrategy {
        // "누가 뭐래도 이 라이브러리는 무조건 이 버전을 써!" 라고 강제합니다.
        force 'commons-beanutils:commons-beanutils:1.9.4' 
    }
}
```

### 방법 3: 오진단 예외 처리 (Suppression)
보안 로직상 실제로는 쓰지 않거나, 이미 패치되었는데 오진단(False Positive)하는 경우입니다.
1. 리포트의 `suppress` 버튼을 눌러 XML 코드를 복사합니다.
2. `config/suppressions.xml` 파일을 만들고 붙여넣습니다.
3. `build.gradle`에 해당 파일을 설정합니다.


