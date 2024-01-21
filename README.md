# pnpm + turborepo

pnpm과 turborepo로 구축한 훕치치 모노레포 환경입니다.

## turborepo

![image](https://github.com/seongminn/pnpm-turborepo-practice/assets/88662637/423169f0-adf4-45d8-bd7d-6f0290e8fdc8)
<details>
  <summary><h3>Why Turborepo?</h3></summary>

  1. `Incremental builds`:
  작업 진행을 캐싱해 이미 계산된 내용은 건너 뛰는 것을 의미합니다. 빌드는 딱 한 번만 하는 것을 목표로 합니다.

  2. `Content-aware hasing`:
  타임스탬프가 아닌 콘텐츠를 인식하는 방식으로 해싱을 지원합니다. 이를 통해 모든 파일을 다시 빌드하는 것이 아니라 변경된 파일만 빌드합니다.

  3. `Cloud caching`:
  클라우드 빌드 캐시를 팀원 및 CI/CD와 공유합니다. 이를 통해 로컬 환경을 넘어 클라우드 환경에서도 빠른 빌드를 제공합니다.

  4. `Parallel execution`:
  모든 코어를 사용하는 병렬 실행을 목표로 합니다. 지정된 태스크 단위로 의존성을 판단해 최대한 병렬적으로 작업을 진행합니다.

  5. `Task Pipelines`:
  태스크 간의 연결을 정의해서 빌드를 언제 어떻게 실행할지 판단해 최적화합니다.

  6. `Zero Runtime Overhead`:
  런타임 코드와 소스 맵을 다루지 않기 때문에 런타임 단계에서 파악하지 못한 리스크가 불거질 위험이 없습니다.

  7. `Pruned subsets`:
  빌드에 필요한 요소만으로 모노 레포의 하위 집합을 생성해 PaaS 배포 속도를 높입니다.

  8. `JSON configuration`:
  별도의 코드 작업 없이 JSON 설정으로 터보를 사용할 수 있습니다.
  // turbo.json { "baseBranch": "origin/main", "pipeline": { "build": { ... } } }

  9. `Profile in browser`:
  빌드 프로필로 빌드 과정을 시각화하면 병목 지점을 쉽게 찾을 수 있습니다.

</details>

## 시작하기

1. 프로젝트 클론: git clone
2. 의존성 설치: pnpm install (필요한 경우 pnpm 전역 설치 `npm install -g pnpm`)
3. 개발 서버 실행: pnpm --filter `<package name>` dev (package name = 'manager' | 'spectator')
  - pnpm `<command>` --filter `<package name>`
  - --filter(-F) 플래그는 패키지의 하위 집합에 한해 명령어를 실행할 수 있도록 합니다.
  - 순서는 무관합니다.
  - [참고](https://pnpm.io/ko/filtering#--filter-prod-filtering_pattern)

## 프로젝트 구조

**/apps/manager**
- 훕치치 매니저 페이지를 담당하는 next.js 프로젝트

**/apps/spectator**
- 훕치치 관객 페이지를 담당하는 next.js 프로젝트

**/packages/typescript-config**
- tsconfig 파일을 모아둔 패키지

**/packages/eslint-config**
- eslint 파일을 모아둔 패키지

**/packages/ui**
- react로 구축한 공용 컴포넌트 패키지

/apps/storybook
  - 스토리북 페이지
/packages/core
/packages/hooks
/packages/utils
/packages/types
등이 추가될 수 있습니다.
- hooks, utils는 core로 합칠 수도 있음

### 패키지명

기존과 같이 패키지 이름에는 `@packages`가 prefix로 붙어 있습니다. 이것도 좋지만 훕치치의 하위 패키지임을 명확하게 표시하기 위해 `@hcc`, `@chichi` 등의 고유한 접두사를 붙이는 것도 좋을 것 같습니다.

### 패키지 추가

1. 원하는 위치에 디렉토리를 생성한다.
2. pnpm init
3. package name 수정
4. 원하는 eslint, tsconfig 파일을 연결해준다. (필요한 경우에는 새로운 파일을 생성할 수 있다.)
5. apps, packages 이외의 위치에 새로운 패키지를 추가하고 싶다면 `pnpm-workspace.yaml` 파일에 경로를 추가해준다.

## turbo.json

turbo.json 내부에는 pipeline이라는 key가 존재합니다. 이는 곧 하나의 태스크 단위이고, 병렬 처리 및 의존성의 범위가 됩니다. 파이프라인의 key는 패키지 스크립트의 이름이고, array 형태의 value를 지정할 수 있습니다.

```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "pipeline": {
    "build": {
      // build 스크립트를 실행했을 때 의존하는 스크립트의 종류를 나타냅니다.
      // `^` 표시는 해당 스크립트가 실행된 패키지가 의존하고 있는 패키지의 build 스크립트를 의미합니다.
      // 따라서, pnpm --filter manager build를 실행했다면 manager 패키지가 의존하고 있는 @packages/ui 패키지의 build 스크립트가 실행되고, @packages/ui가 build 되고 난 뒤 manager의 build 스크립트를 실행합니다.
      "dependsOn": ["^build"],
      // 캐시 폴더를 지정합니다.
      "outputs": [".next/**", "lib/**", "storybook-static/**"]
    },

    // 아무런 값을 지정하지 않으면 의존성이 없다는 것이고, 언제든지 바로 실행될 수 있음을 의미합니다.
    "lint": {},

    "dev": {
      // <package name>#<script>는 특정 패키지의 특정 스크립트를 명시적으로 지정합니다.
      // 따라서 pnpm --filter manager dev를 입력하면 @packages/ui 패키지의 build 스크립트를 실행하고 난 뒤 dev 스크립트를 실행합니다.
      "dependsOn": ["@packages/ui#build"],
      // 핫 로딩이 필요한 경우 캐시를 사용하지 않습니다.
      "cache": false,
      "persistent": true
    }
  }
}
```

추후 스토리북이나 core package가 추가된다면 turbo.json의 파이프라인을 잘 활용하여 각 패키지가 독립적으로 수행될 수 있도록 하는 것이 중요합니다. 

Turborepo는 각 패키지의 코드 자체를 해시로 구분하고 참조하기 때문에 변경이 발생한 패키지에 대해서만 스크립트를 수행합니다. 한 번 수행한 스크립트에 대해서는 캐시를 생성하고, 이후에는 불필요한 수행이 발생하지 않도록 합니다. 이는 manager와 spectator 패키지는 @packages/ui 패키지를 동시에 의존하는 것과 같은 상황에서 유용합니다. manager 패키지에서 한 번 @packages/ui를 빌드하면 캐시를 생성하기 때문에 spectator에서는 불필요하게 빌드를 수행할 필요가 없습니다. 
