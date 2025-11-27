# 재사용 가능한 워크플로우

워크플로우를 모듈화하여 재사용하는 방법입니다.

## Reusable Workflow 정의

```yaml
# .github/workflows/reusable-build.yml
name: Reusable Build

on:
  workflow_call:
    inputs:
      node-version:
        description: 'Node.js version'
        required: false
        type: string
        default: '20'
      working-directory:
        description: 'Working directory'
        required: false
        type: string
        default: '.'
    secrets:
      npm-token:
        required: false
    outputs:
      build-version:
        description: 'Built version'
        value: ${{ jobs.build.outputs.version }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'
      
      - name: Install
        run: npm ci
        env:
          NPM_TOKEN: ${{ secrets.npm-token }}
      
      - name: Build
        run: npm run build
      
      - name: Get version
        id: version
        run: echo "version=$(node -p "require('./package.json').version")" >> $GITHUB_OUTPUT
      
      - uses: actions/upload-artifact@v4
        with:
          name: build
          path: ${{ inputs.working-directory }}/dist/
```

## Reusable Workflow 호출

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]

jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      node-version: '20'
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Use build version
        run: echo "Deploying version ${{ needs.build.outputs.build-version }}"
```

## 외부 저장소 워크플로우

```yaml
jobs:
  call-external:
    uses: owner/repo/.github/workflows/workflow.yml@main
    with:
      input1: value1
    secrets: inherit  # 모든 secrets 전달
```

## Composite Action

```yaml
# .github/actions/setup-project/action.yml
name: 'Setup Project'
description: 'Setup Node.js project with caching'

inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '20'

runs:
  using: 'composite'
  steps:
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'
    
    - name: Install dependencies
      shell: bash
      run: npm ci
    
    - name: Show versions
      shell: bash
      run: |
        echo "Node: $(node --version)"
        echo "npm: $(npm --version)"
```

## Composite Action 사용

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup
        uses: ./.github/actions/setup-project
        with:
          node-version: '20'
      
      - name: Build
        run: npm run build
```

## 모범 사례

1. **명확한 인터페이스**: inputs/outputs 명확히 정의
2. **기본값 제공**: 선택적 입력에 합리적 기본값
3. **버전 고정**: 외부 actions는 버전 명시
4. **문서화**: 사용법 README 작성
5. **테스트**: 재사용 워크플로우도 테스트
