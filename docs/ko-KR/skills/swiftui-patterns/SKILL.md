---
name: swiftui-patterns
description: SwiftUI 아키텍처 패턴, @Observable을 이용한 상태 관리, 뷰 구성, 내비게이션, 성능 최적화 및 현대적인 iOS/macOS UI best practices.
---

# SwiftUI 패턴

Apple 플랫폼에서 선언적이고 성능이 뛰어난 사용자 인터페이스를 구축하기 위한 현대적인 SwiftUI 패턴입니다. Observation 프레임워크, 뷰 구성, 타입 세이프(type-safe) 내비게이션 및 성능 최적화를 다룹니다.

## 활성화 시점

- SwiftUI 뷰 구축 및 상태 관리(`@State`, `@Observable`, `@Binding`)
- `NavigationStack`을 이용한 내비게이션 흐름 설계
- 뷰 모델 및 데이터 흐름 구조화
- 리스트 및 복잡한 레이아웃의 렌더링 성능 최적화
- SwiftUI의 environment 값 및 dependency injection 작업

## 상태 관리

### Property Wrapper 선택

상황에 맞는 가장 단순한 wrapper를 선택하세요:

| Wrapper | Use Case |
|---------|----------|
| `@State` | 뷰 로컬 값 타입 (토글, 폼 필드, 시트 프레젠테이션) |
| `@Binding` | 부모의 `@State`에 대한 양방향 참조 |
| `@Observable` 클래스 + `@State` | 여러 프로퍼티를 가진 소유된(owned) 모델 |
| `@Observable` 클래스 (wrapper 없음) | 부모로부터 전달된 읽기 전용 참조 |
| `@Bindable` | `@Observable` 프로퍼티에 대한 양방향 바인딩 |
| `@Environment` | `.environment()`를 통해 주입된 공유 dependency |

### @Observable ViewModel

`ObservableObject`가 아닌 `@Observable`을 사용하세요. 이는 프로퍼티 수준의 변경을 추적하므로 SwiftUI는 변경된 프로퍼티를 읽는 뷰만 다시 렌더링합니다:

```swift
 @Observable
final class ItemListViewModel {
    private(set) var items: [Item] = []
    private(set) var isLoading = false
    var searchText = ""

    private let repository: any ItemRepository

    init(repository: any ItemRepository = DefaultItemRepository()) {
        self.repository = repository
    }

    func load() async {
        isLoading = true
        defer { isLoading = false }
        items = (try? await repository.fetchAll()) ?? []
    }
}
```

### ViewModel을 사용하는 뷰

```swift
struct ItemListView: View {
    @everything-claude-code/tests/lib/state-store.test.js private var viewModel: ItemListViewModel

    init(viewModel: ItemListViewModel = ItemListViewModel()) {
        _viewModel = State(initialValue: viewModel)
    }

    var body: some View {
        List(viewModel.items) { item in
            ItemRow(item: item)
        }
        .searchable(text: $viewModel.searchText)
        .overlay { if viewModel.isLoading { ProgressView() } }
        .task { await viewModel.load() }
    }
}
```

### Environment 주입

`@EnvironmentObject`를 `@Environment`로 교체하세요:

```swift
// Inject
ContentView()
    .environment(authManager)

// Consume
struct ProfileView: View {
    @Environment(AuthManager.self) private var auth

    var body: some View {
        Text(auth.currentUser?.name ?? "Guest")
    }
}
```

## 뷰 구성

### 무효화(Invalidation)를 제한하기 위한 서브뷰 추출

뷰를 작고 집중된 구조체로 분할하세요. 상태가 변경될 때 해당 상태를 읽는 서브뷰만 다시 렌더링됩니다:

```swift
struct OrderView: View {
    @everything-claude-code/tests/lib/state-store.test.js private var viewModel = OrderViewModel()

    var body: some View {
        VStack {
            OrderHeader(title: viewModel.title)
            OrderItemList(items: viewModel.items)
            OrderTotal(total: viewModel.total)
        }
    }
}
```

### 재사용 가능한 스타일링을 위한 ViewModifier

```swift
struct CardModifier: ViewModifier {
    func body(content: Content) -> some View {
        content
            .padding()
            .background(.regularMaterial)
            .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}

extension View {
    func cardStyle() -> some View {
        modifier(CardModifier())
    }
}
```

## 내비게이션

### 타입 세이프 NavigationStack

프로그래밍 방식의 타입 세이프한 라우팅을 위해 `NavigationStack`과 `NavigationPath`를 사용하세요:

```swift
 @Observable
final class Router {
    var path = NavigationPath()

    func navigate(to destination: Destination) {
        path.append(destination)
    }

    func popToRoot() {
        path = NavigationPath()
    }
}

enum Destination: Hashable {
    case detail(Item.ID)
    case settings
    case profile(User.ID)
}

struct RootView: View {
    @everything-claude-code/tests/lib/state-store.test.js private var router = Router()

    var body: some View {
        NavigationStack(path: $router.path) {
            HomeView()
                .navigationDestination(for: Destination.self) { dest in
                    switch dest {
                    case .detail(let id): ItemDetailView(itemID: id)
                    case .settings: SettingsView()
                    case .profile(let id): ProfileView(userID: id)
                    }
                }
        }
        .environment(router)
    }
}
```

## 성능

### 대규모 컬렉션에 Lazy 컨테이너 사용

`LazyVStack`과 `LazyHStack`은 뷰가 화면에 보일 때만 생성합니다:

```swift
ScrollView {
    LazyVStack(spacing: 8) {
        ForEach(items) { item in
            ItemRow(item: item)
        }
    }
}
```

### 고유 식별자

`ForEach`에서는 항상 고유하고 변하지 않는 ID를 사용하세요. 배열 인덱스 사용을 피하세요:

```swift
// Use Identifiable conformance or explicit id
ForEach(items, id: \.stableID) { item in
    ItemRow(item: item)
}
```

### body 안에서 비용이 큰 작업 피하기

- `body` 내부에서 I/O, 네트워크 호출 또는 무거운 계산을 수행하지 마세요.
- 비동기 작업에는 `.task {}`를 사용하세요. 뷰가 사라질 때 자동으로 취소됩니다.
- 스크롤 뷰에서 `.sensoryFeedback()`과 `.geometryGroup()`은 가급적 적게 사용하세요.
- 리스트에서 `.shadow()`, `.blur()`, `.mask()` 사용을 최소화하세요. 오프스크린 렌더링을 유발합니다.

### Equatable 준수

`body` 연산 비용이 큰 뷰의 경우, `Equatable`을 준수하여 불필요한 재렌더링을 건너뛰세요:

```swift
struct ExpensiveChartView: View, Equatable {
    let dataPoints: [DataPoint] // DataPoint must conform to Equatable

    static func == (lhs: Self, rhs: Self) -> Bool {
        lhs.dataPoints == rhs.dataPoints
    }

    var body: some View {
        // Complex chart rendering
    }
}
```

## Previews

빠른 반복 작업을 위해 인라인 목(mock) 데이터와 함께 `#Preview` 매크로를 사용하세요:

```swift
#Preview("Empty state") {
    ItemListView(viewModel: ItemListViewModel(repository: EmptyMockRepository()))
}

#Preview("Loaded") {
    ItemListView(viewModel: ItemListViewModel(repository: PopulatedMockRepository()))
}
```

## 피해야 할 안티 패턴

- 새로운 코드에서 `ObservableObject` / `@Published` / `@StateObject` / `@EnvironmentObject` 사용 - `@Observable`로 마이그레이션하세요.
- `body`나 `init`에 비동기 작업을 직접 배치 - `.task {}` 또는 명시적인 로드 메서드를 사용하세요.
- 데이터를 소유하지 않는 자식 뷰 내부에서 `@State`로 뷰 모델 생성 - 대신 부모로부터 전달하세요.
- `AnyView` 타입 삭제(type erasure) 사용 - 조건부 뷰에는 `@ViewBuilder`나 `Group`을 선호하세요.
- actor와 데이터를 주고받을 때 `Sendable` 요구 사항 무시

## 참고 자료

actor 기반 persistence 패턴은 skill: `swift-actor-persistence`를 참조하세요.
프로토콜 기반 DI 및 Swift Testing을 이용한 테스트는 skill: `swift-protocol-di-testing`을 참조하세요.

--- Content from referenced files ---
Content from @everything-claude-code/tests/lib/state-store.test.js:
```javascript
/**
 * Tests for the SQLite-backed ECC state store and CLI commands.
 */

const assert = require('assert');
const fs = require('fs');
const os = require('os');
const path = require('path');
const { spawnSync } = require('child_process');

const {
  createStateStore,
  resolveStateStorePath,
} = require('../../scripts/lib/state-store');

const ECC_SCRIPT = path.join(__dirname, '..', '..', 'scripts', 'ecc.js');
const STATUS_SCRIPT = path.join(__dirname, '..', '..', 'scripts', 'status.js');
const SESSIONS_SCRIPT = path.join(__dirname, '..', '..', 'scripts', 'sessions-cli.js');

async function test(name, fn) {
  try {
    await fn();
    console.log(`  \u2713 ${name}`);
    return true;
  } catch (error) {
    console.log(`  \u2717 ${name}`);
    console.log(`    Error: ${error.message}`);
    return false;
  }
}

function createTempDir(prefix) {
  return fs.mkdtempSync(path.join(os.tmpdir(), prefix));
}

function cleanupTempDir(dirPath) {
  fs.rmSync(dirPath, { recursive: true, force: true });
}

function runNode(scriptPath, args = [], options = {}) {
  return spawnSync('node', [scriptPath, ...args], {
    encoding: 'utf8',
    cwd: options.cwd || process.cwd(),
    env: {
      ...process.env,
      ...(options.env || {}),
    },
  });
}

function parseJson(stdout) {
  return JSON.parse(stdout.trim());
}

async function seedStore(dbPath) {
  const store = await createStateStore({ dbPath });

  store.upsertSession({
    id: 'session-active',
    adapterId: 'dmux-tmux',
    harness: 'claude',
    state: 'active',
    repoRoot: '/tmp/ecc-repo',
    startedAt: '2026-03-15T08:00:00.000Z',
    endedAt: null,
    snapshot: {
      schemaVersion: 'ecc.session.v1',
      adapterId: 'dmux-tmux',
      session: {
        id: 'session-active',
        kind: 'orchestrated',
        state: 'active',
        repoRoot: '/tmp/ecc-repo',
      },
      workers: [
        {
          id: 'worker-1',
          label: 'Worker 1',
          state: 'active',
          branch: 'feat/state-store',
          worktree: '/tmp/ecc-repo/.worktrees/worker-1',
        },
        {
          id: 'worker-2',
          label: 'Worker 2',
          state: 'idle',
          branch: 'feat/state-store',
          worktree: '/tmp/ecc-repo/.worktrees/worker-2',
        },
      ],
      aggregates: {
        workerCount: 2,
        states: {
          active: 1,
          idle: 1,
        },
      },
    },
  });

  store.upsertSession({
    id: 'session-recorded',
    adapterId: 'claude-history',
    harness: 'claude',
    state: 'recorded',
    repoRoot: '/tmp/ecc-repo',
    startedAt: '2026-03-14T18:00:00.000Z',
    endedAt: '2026-03-14T19:00:00.000Z',
    snapshot: {
      schemaVersion: 'ecc.session.v1',
      adapterId: 'claude-history',
      session: {
        id: 'session-recorded',
        kind: 'history',
        state: 'recorded',
        repoRoot: '/tmp/ecc-repo',
      },
      workers: [
        {
          id: 'worker-hist',
          label: 'History Worker',
          state: 'recorded',
          branch: 'main',
          worktree: '/tmp/ecc-repo',
        },
      ],
      aggregates: {
        workerCount: 1,
        states: {
          recorded: 1,
        },
      },
    },
  });

  store.insertSkillRun({
    id: 'skill-run-1',
    skillId: 'tdd-workflow',
    skillVersion: '1.0.0',
    sessionId: 'session-active',
    taskDescription: 'Write store tests',
    outcome: 'success',
    failureReason: null,
    tokensUsed: 1200,
    durationMs: 3500,
    userFeedback: 'useful',
    createdAt: '2026-03-15T08:05:00.000Z',
  });

  store.insertSkillRun({
    id: 'skill-run-2',
    skillId: 'security-review',
    skillVersion: '1.0.0',
    sessionId: 'session-active',
    taskDescription: 'Review state-store design',
    outcome: 'failed',
    failureReason: 'timeout',
    tokensUsed: 800,
    durationMs: 1800,
    userFeedback: null,
    createdAt: '2026-03-15T08:06:00.000Z',
  });

  store.insertSkillRun({
    id: 'skill-run-3',
    skillId: 'code-reviewer',
    skillVersion: '1.0.0',
    sessionId: 'session-recorded',
    taskDescription: 'Inspect CLI formatting',
    outcome: 'success',
    failureReason: null,
    tokensUsed: 500,
    durationMs: 900,
    userFeedback: 'clear',
    createdAt: '2026-03-15T08:07:00.000Z',
  });

  store.insertSkillRun({
    id: 'skill-run-4',
    skillId: 'planner',
    skillVersion: '1.0.0',
    sessionId: 'session-recorded',
    taskDescription: 'Outline ECC 2.0 work',
    outcome: 'unknown',
    failureReason: null,
    tokensUsed: 300,
    durationMs: 500,
    userFeedback: null,
    createdAt: '2026-03-15T08:08:00.000Z',
  });

  store.upsertSkillVersion({
    skillId: 'tdd-workflow',
    version: '1.0.0',
    contentHash: 'abc123',
    amendmentReason: 'initial',
    promotedAt: '2026-03-10T00:00:00.000Z',
    rolledBackAt: null,
  });

  store.insertDecision({
    id: 'decision-1',
    sessionId: 'session-active',
    title: 'Use SQLite for durable state',
    rationale: 'Need queryable local state for ECC control plane',
    alternatives: ['json-files', 'memory-only'],
    supersedes: null,
    status: 'active',
    createdAt: '2026-03-15T08:09:00.000Z',
  });

  store.upsertInstallState({
    targetId: 'claude-home',
    targetRoot: '/tmp/home/.claude',
    profile: 'developer',
    modules: ['rules-core', 'orchestration'],
    operations: [
      {
        kind: 'copy-file',
        destinationPath: '/tmp/home/.claude/agents/planner.md',
      },
    ],
    installedAt: '2026-03-15T07:00:00.000Z',
    sourceVersion: '1.8.0',
  });

  store.insertGovernanceEvent({
    id: 'gov-1',
    sessionId: 'session-active',
    eventType: 'policy-review-required',
    payload: {
      severity: 'warning',
      owner: 'security-reviewer',
    },
    resolvedAt: null,
    resolution: null,
    createdAt: '2026-03-15T08:10:00.000Z',
  });

  store.insertGovernanceEvent({
    id: 'gov-2',
    sessionId: 'session-recorded',
    eventType: 'decision-accepted',
    payload: {
      severity: 'info',
    },
    resolvedAt: '2026-03-15T08:11:00.000Z',
    resolution: 'accepted',
    createdAt: '2026-03-15T08:09:30.000Z',
  });

  store.close();
}

async function runTests() {
  console.log('\n=== Testing state-store ===\n');

  let passed = 0;
  let failed = 0;

  if (await test('creates the default state.db path and applies migrations idempotently', async () => {
    const homeDir = createTempDir('ecc-state-home-');

    try {
      const expectedPath = path.join(homeDir, '.claude', 'ecc', 'state.db');
      assert.strictEqual(resolveStateStorePath({ homeDir }), expectedPath);

      const firstStore = await createStateStore({ homeDir });
      const firstMigrations = firstStore.getAppliedMigrations();
      firstStore.close();

      assert.strictEqual(firstMigrations.length, 1);
      assert.strictEqual(firstMigrations[0].version, 1);
      assert.ok(fs.existsSync(expectedPath));

      const secondStore = await createStateStore({ homeDir });
      const secondMigrations = secondStore.getAppliedMigrations();
      secondStore.close();

      assert.strictEqual(secondMigrations.length, 1);
      assert.strictEqual(secondMigrations[0].version, 1);
    } finally {
      cleanupTempDir(homeDir);
    }
  })) passed += 1; else failed += 1;

  if (await test('preserves SQLite special database names like :memory:', async () => {
    const tempDir = createTempDir('ecc-state-memory-');
    const previousCwd = process.cwd();

    try {
      process.chdir(tempDir);
      assert.strictEqual(resolveStateStorePath({ dbPath: ':memory:' }), ':memory:');

      const store = await createStateStore({ dbPath: ':memory:' });
      assert.strictEqual(store.dbPath, ':memory:');
      assert.strictEqual(store.getAppliedMigrations().length, 1);
      store.close();

      assert.ok(!fs.existsSync(path.join(tempDir, ':memory:')));
    } finally {
      process.chdir(previousCwd);
      cleanupTempDir(tempDir);
    }
  })) passed += 1; else failed += 1;

  if (await test('stores sessions and returns detailed session views with workers, skill runs, and decisions', async () => {
    const testDir = createTempDir('ecc-state-db-');
    const dbPath = path.join(testDir, 'state.db');

    try {
      await seedStore(dbPath);

      const store = await createStateStore({ dbPath });
      const listResult = store.listRecentSessions({ limit: 10 });
      const detail = store.getSessionDetail('session-active');
      store.close();

      assert.strictEqual(listResult.totalCount, 2);
      assert.strictEqual(listResult.sessions[0].id, 'session-active');
      assert.strictEqual(detail.session.id, 'session-active');
      assert.strictEqual(detail.workers.length, 2);
      assert.strictEqual(detail.skillRuns.length, 2);
      assert.strictEqual(detail.decisions.length, 1);
      assert.deepStrictEqual(detail.decisions[0].alternatives, ['json-files', 'memory-only']);
    } finally {
      cleanupTempDir(testDir);
    }
  })) passed += 1; else failed += 1;

  if (await test('builds a status snapshot with active sessions, skill rates, install health, and pending governance', async () => {
    const testDir = createTempDir('ecc-state-db-');
    const dbPath = path.join(testDir, 'state.db');

    try {
      await seedStore(dbPath);

      const store = await createStateStore({ dbPath });
      const status = store.getStatus();
      store.close();

      assert.strictEqual(status.activeSessions.activeCount, 1);
      assert.strictEqual(status.activeSessions.sessions[0].id, 'session-active');
      assert.strictEqual(status.skillRuns.summary.totalCount, 4);
      assert.strictEqual(status.skillRuns.summary.successCount, 2);
      assert.strictEqual(status.skillRuns.summary.failureCount, 1);
      assert.strictEqual(status.skillRuns.summary.unknownCount, 1);
      assert.strictEqual(status.installHealth.status, 'healthy');
      assert.strictEqual(status.installHealth.totalCount, 1);
      assert.strictEqual(status.governance.pendingCount, 1);
      assert.strictEqual(status.governance.events[0].id, 'gov-1');
    } finally {
      cleanupTempDir(testDir);
    }
  })) passed += 1; else failed += 1;

  if (await test('validates entity payloads before writing to the database', async () => {
    const testDir = createTempDir('ecc-state-db-');
    const dbPath = path.join(testDir, 'state.db');

    try {
      const store = await createStateStore({ dbPath });
      assert.throws(() => {
        store.upsertSession({
          id: '',
          adapterId: 'dmux-tmux',
          harness: 'claude',
          state: 'active',
          repoRoot: '/tmp/repo',
          startedAt: '2026-03-15T08:00:00.000Z',
          endedAt: null,
          snapshot: {},
        });
      }, /Invalid session/);

      assert.throws(() => {
        store.insertDecision({
          id: 'decision-invalid',
          sessionId: 'missing-session',
          title: 'Reject non-array alternatives',
          rationale: 'alternatives must be an array',
          alternatives: { unexpected: true },
          supersedes: null,
          status: 'active',
          createdAt: '2026-03-15T08:15:00.000Z',
        });
      }, /Invalid decision/);

      assert.throws(() => {
        store.upsertInstallState({
          targetId: 'claude-home',
          targetRoot: '/tmp/home/.claude',
          profile: 'developer',
          modules: 'rules-core',
          operations: [],
          installedAt: '2026-03-15T07:00:00.000Z',
          sourceVersion: '1.8.0',
        });
      }, /Invalid installState/);

      store.close();
    } finally {
      cleanupTempDir(testDir);
    }
  })) passed += 1; else failed += 1;

  if (await test('status CLI supports human-readable and --json output', async () => {
    const testDir = createTempDir('ecc-state-cli-');
    const dbPath = path.join(testDir, 'state.db');

    try {
      await seedStore(dbPath);

      const jsonResult = runNode(STATUS_SCRIPT, ['--db', dbPath, '--json']);
      assert.strictEqual(jsonResult.status, 0, jsonResult.stderr);
      const jsonPayload = parseJson(jsonResult.stdout);
      assert.strictEqual(jsonPayload.activeSessions.activeCount, 1);
      assert.strictEqual(jsonPayload.governance.pendingCount, 1);

      const humanResult = runNode(STATUS_SCRIPT, ['--db', dbPath]);
      assert.strictEqual(humanResult.status, 0, humanResult.stderr);
      assert.match(humanResult.stdout, /Active sessions: 1/);
      assert.match(humanResult.stdout, /Skill runs \(last 20\):/);
      assert.match(humanResult.stdout, /Install health: healthy/);
      assert.match(humanResult.stdout, /Pending governance events: 1/);
    } finally {
      cleanupTempDir(testDir);
    }
  })) passed += 1; else failed += 1;

  if (await test('sessions CLI supports list and detail views in human-readable and --json output', async () => {
    const testDir = createTempDir('ecc-state-cli-');
    const dbPath = path.join(testDir, 'state.db');

    try {
      await seedStore(dbPath);

      const listJsonResult = runNode(SESSIONS_SCRIPT, ['--db', dbPath, '--json']);
      assert.strictEqual(listJsonResult.status, 0, listJsonResult.stderr);
      const listPayload = parseJson(listJsonResult.stdout);
      assert.strictEqual(listPayload.totalCount, 2);
      assert.strictEqual(listPayload.sessions[0].id, 'session-active');

      const detailJsonResult = runNode(SESSIONS_SCRIPT, ['session-active', '--db', dbPath, '--json']);
      assert.strictEqual(detailJsonResult.status, 0, detailJsonResult.stderr);
      const detailPayload = parseJson(detailJsonResult.stdout);
      assert.strictEqual(detailPayload.session.id, 'session-active');
      assert.strictEqual(detailPayload.workers.length, 2);
      assert.strictEqual(detailPayload.skillRuns.length, 2);
      assert.strictEqual(detailPayload.decisions.length, 1);

      const detailHumanResult = runNode(SESSIONS_SCRIPT, ['session-active', '--db', dbPath]);
      assert.strictEqual(detailHumanResult.status, 0, detailHumanResult.stderr);
      assert.match(detailHumanResult.stdout, /Session: session-active/);
      assert.match(detailHumanResult.stdout, /Workers: 2/);
      assert.match(detailHumanResult.stdout, /Skill runs: 2/);
      assert.match(detailHumanResult.stdout, /Decisions: 1/);
    } finally {
      cleanupTempDir(testDir);
    }
  })) passed += 1; else failed += 1;

  if (await test('ecc CLI delegates the new status and sessions subcommands', async () => {
    const testDir = createTempDir('ecc-state-cli-');
    const dbPath = path.join(testDir, 'state.db');

    try {
      await seedStore(dbPath);

      const statusResult = runNode(ECC_SCRIPT, ['status', '--db', dbPath, '--json']);
      assert.strictEqual(statusResult.status, 0, statusResult.stderr);
      const statusPayload = parseJson(statusResult.stdout);
      assert.strictEqual(statusPayload.activeSessions.activeCount, 1);

      const sessionsResult = runNode(ECC_SCRIPT, ['sessions', 'session-active', '--db', dbPath, '--json']);
      assert.strictEqual(sessionsResult.status, 0, sessionsResult.stderr);
      const sessionsPayload = parseJson(sessionsResult.stdout);
      assert.strictEqual(sessionsPayload.session.id, 'session-active');
      assert.strictEqual(sessionsPayload.skillRuns.length, 2);
    } finally {
      cleanupTempDir(testDir);
    }
  })) passed += 1; else failed += 1;

  console.log(`\nResults: Passed: ${passed}, Failed: ${failed}`);
  process.exit(failed > 0 ? 1 : 0);
}

runTests();
```
