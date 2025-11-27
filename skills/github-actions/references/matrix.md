# Matrix 전략

여러 설정 조합으로 병렬 실행하는 Matrix 전략입니다.

## 기본 사용

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.10', '3.11', '3.12']
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pytest
```

## 다중 차원

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.11', '3.12']
        # 총 6개 조합 (3 OS × 2 Python)
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
```

## Include / Exclude

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        python-version: ['3.11', '3.12']
        
        # 특정 조합 제외
        exclude:
          - os: windows-latest
            python-version: '3.11'
        
        # 추가 조합 포함
        include:
          - os: ubuntu-latest
            python-version: '3.12'
            experimental: true
          - os: macos-latest
            python-version: '3.12'
    
    steps:
      - name: Experimental flag
        if: matrix.experimental
        run: echo "This is experimental"
```

## 실패 처리

```yaml
jobs:
  test:
    strategy:
      fail-fast: false  # 하나 실패해도 나머지 계속
      matrix:
        version: ['3.10', '3.11', '3.12']
```

## 최대 병렬 실행

```yaml
jobs:
  test:
    strategy:
      max-parallel: 2  # 최대 2개 동시 실행
      matrix:
        version: ['3.10', '3.11', '3.12', '3.13']
```

## 동적 Matrix

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - id: set-matrix
        run: |
          echo 'matrix={"version":["3.11","3.12"]}' >> $GITHUB_OUTPUT

  test:
    needs: setup
    strategy:
      matrix: ${{ fromJson(needs.setup.outputs.matrix) }}
    runs-on: ubuntu-latest
    steps:
      - run: echo "Testing ${{ matrix.version }}"
```

## 실용 예제: Node.js + 브라우저

```yaml
jobs:
  e2e:
    strategy:
      matrix:
        node: ['18', '20']
        browser: ['chromium', 'firefox', 'webkit']
    
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci
      - run: npx playwright install --with-deps ${{ matrix.browser }}
      - run: npm run test:e2e -- --project=${{ matrix.browser }}
```

## 실용 예제: 다중 서비스 버전

```yaml
jobs:
  test:
    strategy:
      matrix:
        include:
          - python: '3.11'
            postgres: '15'
            redis: '6'
          - python: '3.12'
            postgres: '16'
            redis: '7'
    
    services:
      postgres:
        image: postgres:${{ matrix.postgres }}
      redis:
        image: redis:${{ matrix.redis }}
    
    steps:
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python }}
```
