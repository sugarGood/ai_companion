# AI 智能伙伴 App - 技术方案（v2）

> 基于需求方案 v2，完全对齐"养成型 AI 生命体"定位

## 一、技术选型

| 层次 | 技术 | 版本 | 选型理由 |
|------|------|------|----------|
| **跨平台框架** | Flutter | 3.x | 一套代码覆盖 iOS/Android，渲染性能优秀，动画能力强 |
| **开发语言** | Dart | 3.x | Flutter 原生语言，强类型，异步友好 |
| **AI 接口** | Claude API | Messages API v1 | 支持流式输出，System Prompt 灵活，推理能力强 |
| **状态管理** | Riverpod | 2.x | 编译时安全，支持异步状态，测试友好 |
| **路由** | GoRouter | 14.x | 声明式路由，支持深度链接，Navigator 2.0 封装 |
| **网络请求** | Dio | 5.x | 拦截器机制，流式响应支持好 |
| **本地数据库** | Isar | 3.x | 高性能 NoSQL，Flutter 原生，支持全文搜索 |
| **2D 角色动画** | rive / Lottie | - | Rive 支持状态机驱动的交互式动画，适合表情/动作系统 |
| **图片生成** | 外部 API (Stable Diffusion / DALL·E) | - | AI 辅助生成角色形象，按需调用 |
| **动画** | flutter_animate | 4.x | 声明式动画，链式调用，性能好 |
| **安全存储** | flutter_secure_storage | 9.x | API Key 等敏感信息加密存储 |
| **生物识别** | local_auth | 2.x | 指纹/Face ID 认证，跨平台统一接口 |
| **Markdown 渲染** | flutter_markdown | 0.7+ | AI 回复中的代码/列表/表格渲染 |
| **本地通知** | flutter_local_notifications | 17.x | 定时通知，任务提醒 |
| **工具类** | uuid, intl, json_annotation | - | ID 生成、日期格式化、JSON 序列化 |

---

## 二、系统架构

### 2.1 整体架构图

```
┌────────────────────────────────────────────────────────────┐
│                       Flutter App                          │
│                                                            │
│  ┌────────────┐  ┌────────────┐  ┌─────────────────────┐  │
│  │ UI Layer   │  │ UI Layer   │  │    UI Layer          │  │
│  │ (Chat +    │  │ (Companion │  │ (Onboarding +       │  │
│  │  Avatar)   │  │  Profile)  │  │  Settings)          │  │
│  └─────┬──────┘  └──────┬─────┘  └──────────┬──────────┘  │
│        │                │                    │             │
│  ┌─────┴────────────────┴────────────────────┴──────────┐  │
│  │              State Layer (Riverpod)                    │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐  │  │
│  │  │ Chat     │ │ Talent   │ │ Rapport  │ │ Avatar  │  │  │
│  │  │ Provider │ │ Provider │ │ Provider │ │ Provider│  │  │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬────┘  │  │
│  └───────┼────────────┼────────────┼────────────┼────────┘  │
│          │            │            │            │            │
│  ┌───────┴────────────┴────────────┴────────────┴────────┐  │
│  │                   Service Layer                        │  │
│  │  ┌────────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐  │  │
│  │  │ Claude     │ │ Memory   │ │ Talent   │ │ Avatar │  │  │
│  │  │ Client     │ │ Service  │ │ Analyzer │ │ Service│  │  │
│  │  └──────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬───┘  │  │
│  └─────────┼────────────┼────────────┼────────────┼──────┘  │
│            │            │            │            │          │
│  ┌─────────┴────────────┴────────────┴────────────┴──────┐  │
│  │                    Data Layer                          │  │
│  │  ┌────────────────┐  ┌──────────────┐  ┌───────────┐  │  │
│  │  │  Isar Database │  │ Secure       │  │ File      │  │  │
│  │  │  (全部业务数据) │  │ Storage      │  │ Storage   │  │  │
│  │  │                │  │ (API Keys)   │  │ (形象资源) │  │  │
│  │  └────────────────┘  └──────────────┘  └───────────┘  │  │
│  └────────────────────────────────────────────────────────┘  │
│            │                                                 │
└────────────┼─────────────────────────────────────────────────┘
             │ HTTPS (SSE Stream)
             ▼
    ┌──────────────────┐     ┌──────────────────┐
    │  Claude API      │     │  Image Gen API   │
    │  (Anthropic)     │     │  (形象生成)       │
    └──────────────────┘     └──────────────────┘
```

### 2.2 分层说明

| 层次 | 职责 | 关键原则 |
|------|------|----------|
| **UI Layer** | 页面展示、用户交互、角色动画 | 仅负责渲染，不含业务逻辑 |
| **State Layer** | 状态管理、数据流转 | Riverpod Provider 管理所有可变状态 |
| **Service Layer** | 业务逻辑、AI 调用、天赋分析 | 可独立测试，不依赖 UI |
| **Data Layer** | 数据持久化、加密存储、文件管理 | 统一数据访问接口 |

---

## 三、核心模块技术设计

### 3.1 生物识别与身份系统

#### 3.1.1 设计理念

- 一台设备支持多人使用，各自独立
- 生物识别（指纹/Face ID）绑定用户身份，按人加载数据
- 不依赖账号系统，纯本地身份

#### 3.1.2 实现方案

```dart
class AuthService {
  final LocalAuthentication _localAuth;
  final SecureStorage _secureStorage;

  /// 检查设备是否支持生物识别
  Future<bool> isBiometricAvailable();

  /// 注册新用户身份
  /// 生成唯一 userId，绑定生物识别特征
  Future<String> registerUser(String companionName) {
    final userId = Uuid().v4();
    // 存储 userId 到安全存储，关联生物识别
    // 后续所有数据按 userId 隔离
    return userId;
  }

  /// 认证并返回 userId
  /// 弹出系统生物识别弹窗，成功后返回对应用户 ID
  Future<String?> authenticate();

  /// 获取当前设备已注册的用户列表
  Future<List<UserProfile>> getRegisteredUsers();
}
```

#### 3.1.3 多用户数据隔离

```
设备本地存储:
├── user_{id_1}/
│   ├── companion_data    (伙伴数据)
│   ├── chat_history      (对话历史)
│   ├── memories          (记忆)
│   └── avatar_assets     (形象资源)
├── user_{id_2}/
│   └── ...
└── shared/
    └── app_settings      (全局设置)
```

---

### 3.2 Claude API 集成

#### 3.2.1 API 客户端封装

```dart
class ClaudeClient {
  final Dio _dio;
  final String _baseUrl = 'https://api.anthropic.com/v1';

  /// 发送消息（流式响应）
  Stream<StreamEvent> sendMessageStream({
    required String systemPrompt,
    required List<Message> messages,
    String model = 'claude-sonnet-4-20250514',
    int maxTokens = 4096,
  });

  /// 发送消息（非流式，用于记忆提取等后台任务）
  Future<String> sendMessage({
    required String systemPrompt,
    required List<Message> messages,
  });
}

/// 流式事件
sealed class StreamEvent {
  const StreamEvent();
}
class StreamDelta extends StreamEvent {
  final String text;
  const StreamDelta(this.text);
}
class StreamComplete extends StreamEvent {
  final int inputTokens;
  final int outputTokens;
  const StreamComplete(this.inputTokens, this.outputTokens);
}
class StreamError extends StreamEvent {
  final String message;
  const StreamError(this.message);
}
```

#### 3.2.2 流式响应处理

```
Client                    Claude API
  │                           │
  │── POST /messages ────────>│
  │   (stream: true)          │
  │                           │
  │<── event: message_start ──│
  │<── event: content_block ──│  逐块接收
  │<── event: content_delta ──│  实时更新 UI + 表情
  │<── event: content_delta ──│
  │<── event: message_stop ───│
  │                           │
  │   解析完成 → 保存消息      │
  │   → 异步提取情绪标签       │
  │   → 更新形象表情           │
```

**技术要点**:
- 使用 Dio 的 `responseType: ResponseType.stream` 接收 SSE
- 逐行解析 `data:` 前缀的 JSON 事件
- 通过 Dart `Stream<StreamEvent>` 暴露给 UI 层
- UI 使用 `StreamBuilder` 实时渲染文字 + 同步更新形象表情

#### 3.2.3 System Prompt 动态组装

```dart
class PromptBuilder {
  /// 组装完整的 System Prompt
  String build({
    required Companion companion,
    required List<MemoryEntry> memories,
    required List<Talent> activeTalents,
    required double rapportLevel,
  }) {
    return '''
${_basePersonality(companion.name)}
${_talentTraits(activeTalents)}
${_rapportStage(rapportLevel)}
${_memoryContext(memories)}
${_behaviorRules()}
${_emotionTagInstruction()}
''';
  }
}
```

**Prompt 分层结构**:

| 层级 | 内容 | 说明 |
|------|------|------|
| 基础人格 | AI 伙伴的通用行为准则 | "你是用户的专属 AI 伙伴，名叫{name}。你有自己的性格和情感..." |
| 天赋特征 | 根据已觉醒天赋注入行为指令 | "你擅长编程领域，回答代码问题时会更深入..." |
| 默契阶段 | 根据默契值调整沟通风格 | 低默契→礼貌生疏；高默契→直接亲密 |
| 记忆上下文 | 已知的用户信息 | "已知：用户是大学生，喜欢 Python..." |
| 行为规则 | 天赋系统相关指令 | "在回复末尾附加情绪标签 [emotion:happy]..." |
| 情绪标签 | 要求 AI 在回复中标注情绪 | 用于驱动形象表情同步 |

**情绪标签机制**:

在 System Prompt 中要求 AI 在每条回复末尾附加隐藏的情绪标签：

```
请在每条回复的最后一行添加情绪标签（此行不展示给用户）：
[emotion:happy|thinking|sad|surprised|angry|neutral|sleepy|excited]
```

客户端解析此标签后驱动形象表情切换，标签行不渲染给用户。

---

### 3.3 每日会话机制

#### 3.3.1 设计理念

- 用户永远感觉在跟"同一个人"聊天，没有"新建会话"概念
- 每天自动开启新的对话上下文，控制 token 成本
- 长期记忆保证连续感

#### 3.3.2 实现方案

```dart
class SessionManager {
  /// 获取或创建今日会话
  /// App 启动时调用，检查是否有今日会话
  Future<ChatSession> getTodaySession(String companionId) async {
    final today = DateTime.now().toDateOnly();
    final existing = await _chatDao.getSessionByDate(companionId, today);

    if (existing != null) return existing;

    // 创建新的今日会话
    final session = ChatSession(
      companionId: companionId,
      date: today,
      createdAt: DateTime.now(),
    );

    // 触发前日对话的记忆提取（异步，不阻塞）
    _triggerMemoryExtraction(companionId, today.subtract(Duration(days: 1)));

    return await _chatDao.createSession(session);
  }

  /// 异步提取前日对话中的关键记忆
  Future<void> _triggerMemoryExtraction(String companionId, DateTime date) async {
    final previousSession = await _chatDao.getSessionByDate(companionId, date);
    if (previousSession == null) return;

    final messages = await _chatDao.getMessages(previousSession.id);
    if (messages.isEmpty) return;

    // 调用 Claude API 提取记忆（后台任务）
    await _memoryService.extractAndSave(companionId, messages);

    // 同时触发天赋分析
    await _talentAnalyzer.analyzeSession(companionId, messages, date);
  }
}
```

#### 3.3.3 对话上下文构建

每次发送消息时，上下文仅包含今日会话的消息 + 长期记忆注入到 System Prompt：

```
┌─────────────────────────────────────────┐
│ System Prompt                            │
│  - 基础人格                              │
│  - 天赋特征                              │
│  - 默契阶段                              │
│  - 长期记忆（结构化信息）                  │
│  - "今天是 2026-03-17，昨天你们聊了..."   │
├─────────────────────────────────────────┤
│ Messages（仅今日）                       │
│  - User: ...                            │
│  - Assistant: ...                       │
│  - User: ...（当前输入）                 │
└─────────────────────────────────────────┘
```

---

### 3.4 记忆系统

#### 3.4.1 架构设计

```
┌──────────────────────────────────────────────┐
│                 Memory System                 │
│                                               │
│  ┌──────────┐  ┌──────────────┐  ┌────────┐  │
│  │ 短期记忆  │  │   长期记忆    │  │ 对话   │  │
│  │ (今日会话 │  │ (结构化条目)  │  │ 摘要   │  │
│  │  上下文)  │  │  (Isar 持久) │  │ (Isar) │  │
│  └─────┬────┘  └──────┬───────┘  └───┬────┘  │
│        │              │              │        │
│  ┌─────┴──────────────┴──────────────┴─────┐  │
│  │            Memory Manager                │  │
│  │  - 每日自动提取关键信息                    │  │
│  │  - 合并/去重/更新记忆条目                  │  │
│  │  - 按相关性筛选注入 System Prompt          │  │
│  │  - 提供用户查看/删除接口                   │  │
│  └──────────────────────────────────────────┘  │
└───────────────────────────────────────────────┘
```

#### 3.4.2 记忆提取策略

```dart
class MemoryExtractor {
  final ClaudeClient _claude;

  /// 每日对话结束后（或次日启动时）自动提取记忆
  Future<List<MemoryEntry>> extractFromConversation(
    List<ChatMessage> messages,
    List<MemoryEntry> existingMemories,
  ) async {
    final response = await _claude.sendMessage(
      systemPrompt: _extractionPrompt,
      messages: [
        Message.user('''
已有记忆：
${existingMemories.map((m) => '- ${m.category}: ${m.key} = ${m.value}').join('\n')}

今日对话：
${messages.map((m) => '${m.role}: ${m.content}').join('\n')}

请提取新的或需要更新的用户信息，以 JSON 数组返回：
[{"category": "...", "key": "...", "value": "...", "action": "add|update|unchanged"}]

category 可选值：
- user_profile（职业、年龄、性别等基本信息）
- preference（沟通风格、兴趣爱好、习惯偏好）
- skill_level（各领域技能水平）
- important_event（用户提到的重要经历/事件）
- goal（短期/长期目标）
- relationship（社交关系，如提到的朋友/家人）
'''),
      ],
    );

    return _parseMemoryEntries(response);
  }
}
```

#### 3.4.3 记忆数据模型

```dart
class MemoryEntry {
  String id;
  String companionId;
  String category;      // 'user_profile' | 'preference' | 'skill_level' | 'important_event' | 'goal' | 'relationship'
  String key;           // 如 'profession', 'favorite_language'
  String value;         // 如 '大学生', 'Python'
  double confidence;    // 置信度 0-1（多次确认 → 置信度提高）
  DateTime createdAt;
  DateTime lastUpdated;
  String sourceSessionId; // 来源会话 ID
}
```

#### 3.4.4 记忆查看与管理

用户可在"记忆管理"页面查看 AI 记住了哪些信息，支持手动删除：

```dart
class MemoryManagerService {
  /// 获取所有记忆，按分类分组
  Future<Map<String, List<MemoryEntry>>> getGroupedMemories(String companionId);

  /// 用户手动删除某条记忆
  Future<void> deleteMemory(String memoryId);

  /// 筛选与当前对话相关的记忆，注入 Prompt
  /// 控制总 Token 量，优先注入高置信度和近期更新的条目
  Future<List<MemoryEntry>> selectRelevantMemories(
    String companionId,
    {int maxTokenBudget = 800}
  );
}
```

---

### 3.5 天赋系统 ⭐

#### 3.5.1 核心规则

- 总共 **5 个天赋槽位**
- 完全 **自动觉醒**，用户不参与选择
- 天赋有强度值，**用进废退**
- 5 个槽位满后，新天赋 **挤掉最弱的**
- 天赋 **不可手动添加或删除**

#### 3.5.2 天赋定义

```dart
/// 天赋类型枚举
enum TalentType {
  // === 性格类天赋 ===
  nightOwl,      // 夜猫子：深夜互动多
  talkative,     // 话痨：长对话多
  listener,      // 树洞：情绪倾诉多
  comedian,      // 段子手：积极互动多
  meticulous,    // 较真怪：追问细节多

  // === 能力类天赋 ===
  coder,         // 代码脑：编程领域
  writer,        // 笔杆子：写作/文案
  butler,        // 管家：日程/提醒
  socialite,     // 社交达人：沟通/社交
  bookworm,      // 读书虫：阅读/书籍
  fitCoach,      // 健身教练：运动/饮食
}

/// 天赋实例
class Talent {
  String id;
  String companionId;
  TalentType type;
  double strength;        // 强度值 0-100
  DateTime awakenedAt;    // 觉醒时间
  DateTime lastActiveAt;  // 最后活跃时间

  /// 天赋是否为性格类
  bool get isPersonality => type.index < 5;

  /// 天赋是否为能力类
  bool get isAbility => type.index >= 5;
}
```

#### 3.5.3 天赋元数据配置

```dart
class TalentMetadata {
  final TalentType type;
  final String name;            // 显示名称
  final String description;     // 描述
  final String icon;            // 图标
  final String awakenMessage;   // 觉醒时 AI 的独白
  final double awakenThreshold; // 觉醒所需的累计信号分数

  // 信号检测关键词/模式
  final List<String> signalKeywords;
  final SignalDetector detector;
}

// 预定义所有天赋的元数据
final talentRegistry = {
  TalentType.nightOwl: TalentMetadata(
    type: TalentType.nightOwl,
    name: '夜猫子',
    description: '深夜模式，语气更柔和',
    awakenMessage: '我发现你经常深夜找我聊天呢...看来我们都是夜猫子。以后深夜我会更温柔地陪你。',
    awakenThreshold: 50.0,
    signalKeywords: [],  // 基于时间检测，不需要关键词
    detector: TimeBasedDetector(startHour: 22, endHour: 5),
  ),
  TalentType.coder: TalentMetadata(
    type: TalentType.coder,
    name: '代码脑',
    description: '编程领域深度能力',
    awakenMessage: '我好像越来越能理解代码了！以后遇到编程问题，放心交给我吧。',
    awakenThreshold: 80.0,
    signalKeywords: ['代码', '编程', 'bug', '函数', 'API', '开发', 'Python', 'Java', 'JavaScript', '调试', '报错'],
    detector: KeywordDetector(),
  ),
  // ... 其他天赋
};
```

#### 3.5.4 天赋分析引擎

```dart
class TalentAnalyzer {
  /// 每日对话结束后分析天赋信号
  Future<TalentAnalysisResult> analyzeSession(
    String companionId,
    List<ChatMessage> messages,
    DateTime sessionDate,
  ) async {
    // 1. 收集本次对话的天赋信号
    final signals = _detectSignals(messages, sessionDate);

    // 2. 更新信号累积分数
    await _updateSignalScores(companionId, signals);

    // 3. 获取当前已觉醒天赋
    final activeTalents = await _talentDao.getActiveTalents(companionId);

    // 4. 检查是否有新天赋达到觉醒阈值
    final newAwakening = await _checkAwakening(companionId, activeTalents);

    // 5. 应用用进废退衰减
    await _applyDecay(activeTalents, signals);

    return TalentAnalysisResult(
      newAwakening: newAwakening,
      updatedTalents: activeTalents,
    );
  }

  /// 检测对话中的天赋信号
  List<TalentSignal> _detectSignals(List<ChatMessage> messages, DateTime date) {
    final signals = <TalentSignal>[];

    for (final talent in TalentType.values) {
      final metadata = talentRegistry[talent]!;
      final score = metadata.detector.detect(messages, date);
      if (score > 0) {
        signals.add(TalentSignal(type: talent, score: score));
      }
    }

    return signals;
  }

  /// 检查觉醒条件
  Future<Talent?> _checkAwakening(
    String companionId,
    List<Talent> activeTalents,
  ) async {
    final scores = await _getAccumulatedScores(companionId);

    for (final entry in scores.entries) {
      final type = entry.key;
      final score = entry.value;
      final metadata = talentRegistry[type]!;

      // 已觉醒的跳过
      if (activeTalents.any((t) => t.type == type)) continue;

      // 未达到阈值的跳过
      if (score < metadata.awakenThreshold) continue;

      // 达到觉醒条件
      if (activeTalents.length < 5) {
        // 还有空槽位，直接觉醒
        return _awakenTalent(companionId, type);
      } else {
        // 槽位已满，挤掉最弱的
        final weakest = activeTalents.reduce(
          (a, b) => a.strength < b.strength ? a : b,
        );
        await _removeTalent(weakest);
        return _awakenTalent(companionId, type);
      }
    }

    return null;
  }

  /// 用进废退：长期不涉及的天赋强度衰减
  Future<void> _applyDecay(
    List<Talent> talents,
    List<TalentSignal> todaySignals,
  ) async {
    for (final talent in talents) {
      final hasSignal = todaySignals.any((s) => s.type == talent.type);
      if (hasSignal) {
        // 有活跃信号，增强
        talent.strength = (talent.strength + 2.0).clamp(0, 100);
        talent.lastActiveAt = DateTime.now();
      } else {
        // 无活跃信号，每日衰减 0.5
        talent.strength = (talent.strength - 0.5).clamp(0, 100);
      }
      await _talentDao.updateTalent(talent);
    }
  }
}
```

#### 3.5.5 信号检测器

```dart
/// 基于关键词的信号检测（用于能力类天赋）
class KeywordDetector implements SignalDetector {
  @override
  double detect(List<ChatMessage> messages, DateTime date) {
    final allText = messages
        .where((m) => m.role == 'user')
        .map((m) => m.content)
        .join(' ');

    int hitCount = 0;
    for (final keyword in keywords) {
      hitCount += RegExp(keyword, caseSensitive: false)
          .allMatches(allText).length;
    }

    // 每次命中得 2 分，单次会话上限 20 分
    return (hitCount * 2.0).clamp(0, 20);
  }
}

/// 基于时间的信号检测（用于夜猫子天赋）
class TimeBasedDetector implements SignalDetector {
  final int startHour;
  final int endHour;

  @override
  double detect(List<ChatMessage> messages, DateTime date) {
    final nightMessages = messages.where((m) {
      final hour = m.createdAt.hour;
      return hour >= startHour || hour < endHour;
    });

    // 深夜每条消息得 1 分
    return nightMessages.length.toDouble().clamp(0, 15);
  }
}

/// 基于对话模式的信号检测（用于性格类天赋）
class PatternDetector implements SignalDetector {
  final PatternType pattern; // longConversation, emotionalContent, detailSeeking, positiveReaction

  @override
  double detect(List<ChatMessage> messages, DateTime date) {
    switch (pattern) {
      case PatternType.longConversation:
        // 话痨：单次对话超过 20 轮
        return messages.length > 40 ? 10.0 : 0;  // 40条 = 20轮（一问一答）
      case PatternType.emotionalContent:
        // 树洞：使用 AI 分析情绪倾诉频率（简化版用关键词）
        return _countEmotionalMessages(messages) * 2.0;
      // ...
    }
  }
}
```

#### 3.5.6 天赋觉醒仪式

天赋觉醒时触发全屏动画 + AI 独白：

```dart
class TalentAwakeningEvent {
  final Talent talent;
  final TalentMetadata metadata;

  /// 觉醒展示流程
  /// 1. 全屏粒子/光效动画（约 3 秒）
  /// 2. 天赋图标从中心展开
  /// 3. AI 独白文字逐字显示
  /// 4. 形象出现新的视觉元素
  /// 5. 用户点击确认，回到对话
}
```

---

### 3.6 成长体系

#### 3.6.1 默契值

默契值是一个简单的数值，代表"我和伙伴有多了解彼此"。

```dart
class RapportSystem {
  /// 默契值范围 0.0 - 100.0
  /// 增长因子：
  ///   - 每日互动：+0.5
  ///   - 连续互动天数加成：+0.1 * consecutiveDays
  ///   - 深度对话（超过 10 轮）：额外 +0.3
  ///   - 天赋觉醒：+2.0
  /// 不衰减，只增不减

  Future<double> updateRapport(String companionId, RapportEvent event) async {
    final companion = await _companionDao.get(companionId);
    double delta = 0;

    switch (event) {
      case RapportEvent.dailyChat:
        delta = 0.5 + 0.1 * companion.consecutiveDays;
      case RapportEvent.deepConversation:
        delta = 0.3;
      case RapportEvent.talentAwakening:
        delta = 2.0;
    }

    companion.rapport = (companion.rapport + delta).clamp(0, 100);
    await _companionDao.update(companion);
    return companion.rapport;
  }

  /// 默契阶段（用于 System Prompt 调整）
  RapportStage getStage(double rapport) {
    if (rapport < 10) return RapportStage.stranger;    // 刚认识
    if (rapport < 30) return RapportStage.familiar;     // 熟悉中
    if (rapport < 60) return RapportStage.close;        // 默契较高
    return RapportStage.deepBond;                       // 深度默契
  }
}
```

#### 3.6.2 AI 自述成长

AI 的成长感知通过 System Prompt 注入实现。根据里程碑事件，在 Prompt 中添加引导：

```dart
class GrowthNarrative {
  /// 生成成长自述引导（注入 System Prompt）
  String buildNarrativeHint(Companion companion) {
    final hints = <String>[];

    // 天数里程碑
    final daysTogether = DateTime.now().difference(companion.createdAt).inDays;
    if (daysTogether == 7) {
      hints.add('今天是你们认识的第 7 天，可以自然地提到这个里程碑。');
    } else if (daysTogether == 30) {
      hints.add('今天是你们认识满一个月的日子，可以表达对这段关系的感受。');
    }

    // 天赋变化感知
    if (companion.recentTalentChange != null) {
      hints.add('你最近感知到用户的兴趣发生了变化：${companion.recentTalentChange}，可以自然地提到。');
    }

    // 默契阶段变化
    if (companion.rapportJustLeveledUp) {
      hints.add('你感觉和用户的默契又提升了，可以用自己的话表达这种感受。');
    }

    return hints.isEmpty ? '' : '【成长感知提示】\n${hints.join('\n')}';
  }
}
```

---

### 3.7 形象系统 ⭐

#### 3.7.1 形象创建流程

```
用户描述 ──> AI 理解需求 ──> 调用图片生成 API ──> 生成多个方案
    │                                                  │
    │              ┌──────────────────────────┐         │
    │              │  用户选择 / 微调细节      │ <───────┘
    │              └───────────┬──────────────┘
    │                          │
    │              ┌───────────▼──────────────┐
    │              │  生成一组情绪表情素材     │
    │              │  (happy, sad, thinking,   │
    │              │   surprised, angry,       │
    │              │   neutral, sleepy,        │
    │              │   excited)                │
    │              └───────────┬──────────────┘
    │                          │
    │              ┌───────────▼──────────────┐
    │              │  生成 Rive/Spine 动画    │
    │              │  状态机文件              │
    │              └───────────┬──────────────┘
    │                          ▼
    │              形象定稿，存入本地文件系统
```

#### 3.7.2 形象服务

```dart
class AvatarService {
  final ClaudeClient _claude;
  final ImageGenClient _imageGen;

  /// Step 1: 用户用自然语言描述想要的形象
  /// AI 将描述转化为结构化的图片生成 Prompt
  Future<String> buildImagePrompt(String userDescription) async {
    final response = await _claude.sendMessage(
      systemPrompt: '你是一个角色形象设计师。请将用户的描述转化为适合 AI 图片生成的英文 Prompt...',
      messages: [Message.user(userDescription)],
    );
    return response;
  }

  /// Step 2: 调用图片生成 API，生成多个候选方案
  Future<List<ImageCandidate>> generateCandidates(String prompt, {int count = 4});

  /// Step 3: 用户选定后，生成一组情绪表情
  Future<AvatarAssetPack> generateEmotionSet(ImageCandidate selected) async {
    // 基于选定的形象，生成 8 种情绪状态的变体
    final emotions = ['happy', 'sad', 'thinking', 'surprised',
                      'angry', 'neutral', 'sleepy', 'excited'];
    // 每种情绪生成一张变体图
    // 返回完整的资源包
  }

  /// Step 4: 生成天赋视觉元素叠加层
  Future<Map<TalentType, String>> generateTalentOverlays(ImageCandidate base);
}

/// 形象资源包
class AvatarAssetPack {
  String basePath;              // 本地存储根路径
  String baseImage;             // 基础形象
  Map<String, String> emotions; // emotion_name -> image_path
  Map<TalentType, String> talentOverlays; // 天赋视觉叠加层
}
```

#### 3.7.3 形象动画系统

```dart
class AvatarAnimationController {
  /// 当前情绪状态
  String _currentEmotion = 'neutral';

  /// 待机动画计时器
  Timer? _idleTimer;

  /// 根据 AI 回复中的情绪标签更新表情
  void updateEmotion(String emotion) {
    if (_currentEmotion == emotion) return;
    _currentEmotion = emotion;
    // 触发表情切换动画（淡入淡出）
    _animateTransition(emotion);
  }

  /// 待机行为：用户长时间不说话
  void startIdleMode() {
    _idleTimer = Timer.periodic(Duration(seconds: 30), (_) {
      // 随机播放待机动作：眨眼、东张西望、打哈欠等
      _playIdleAnimation();
    });
  }

  /// 时段感知：根据当前时间调整状态
  String getTimeBasedState() {
    final hour = DateTime.now().hour;
    if (hour >= 6 && hour < 9) return 'morning';   // 早上：伸懒腰
    if (hour >= 22 || hour < 6) return 'sleepy';    // 深夜：犯困
    return 'active';                                 // 其他：正常
  }

  /// 天赋觉醒时的特殊动画
  Future<void> playAwakeningAnimation(TalentType talent);

  /// 叠加天赋视觉元素
  void applyTalentOverlay(TalentType talent, String overlayPath);
}
```

#### 3.7.4 形象进化

天赋觉醒时，形象上新增对应的视觉元素：

| 天赋 | 视觉元素 |
|------|----------|
| 代码脑 | 科技感光效装饰、数据流粒子 |
| 笔杆子 | 羽毛笔/墨水元素 |
| 健身教练 | 运动头带/哑铃配件 |
| 夜猫子 | 月亮/星星光环 |
| 段子手 | 笑脸气泡 |

实现方式：叠加层渲染，在基础形象上方绘制天赋装饰图层。

---

### 3.8 训练模式

#### 3.8.1 设计理念

- 训练模式是加速天赋觉醒的快捷入口
- 用户选择训练方向后，AI 进入专项对话模式
- 训练对话产生的天赋信号分数翻倍

#### 3.8.2 实现方案

```dart
class TrainingMode {
  /// 预设训练方向
  static const directions = [
    TrainingDirection(
      id: 'oral_english',
      name: '练口语',
      description: '与伙伴进行英语对话练习',
      relatedTalent: TalentType.socialite,
      systemPromptOverride: '现在进入英语口语训练模式。请用英语和用户对话...',
    ),
    TrainingDirection(
      id: 'coding',
      name: '练编程',
      description: '一起做编程题、讨论技术',
      relatedTalent: TalentType.coder,
      systemPromptOverride: '现在进入编程训练模式。请给用户出编程题...',
    ),
    TrainingDirection(
      id: 'writing',
      name: '练写作',
      description: '写作练习和文案打磨',
      relatedTalent: TalentType.writer,
      systemPromptOverride: '现在进入写作训练模式...',
    ),
    // ... 更多方向
  ];

  /// 训练模式下的天赋信号加成
  static const double signalMultiplier = 2.0;
}
```

---

## 四、本地数据库设计

### 4.1 Isar Collections

```dart
// ===== Companion（伙伴）=====
@collection
class CompanionEntity {
  Id id = Isar.autoIncrement;
  String userId;              // 关联的用户 ID（生物识别绑定）
  String name;                // 伙伴名字
  double rapport = 0;         // 默契值 0-100
  int consecutiveDays = 0;    // 连续互动天数
  DateTime createdAt;
  DateTime lastInteractionAt;
  String? avatarPackPath;     // 形象资源包路径
}

// ===== Talent（天赋）=====
@collection
class TalentEntity {
  Id id = Isar.autoIncrement;
  String companionId;
  @enumerated
  TalentType type;
  double strength;            // 强度值 0-100
  DateTime awakenedAt;
  DateTime lastActiveAt;
}

// ===== TalentSignalScore（天赋信号累积分数）=====
@collection
class TalentSignalScoreEntity {
  Id id = Isar.autoIncrement;
  String companionId;
  @enumerated
  TalentType type;
  double accumulatedScore;    // 累积信号分数
  DateTime lastUpdated;
}

// ===== ChatSession（对话会话 - 按日）=====
@collection
class ChatSessionEntity {
  Id id = Isar.autoIncrement;
  String companionId;
  DateTime date;              // 会话日期（按天归档）
  DateTime createdAt;
  int messageCount = 0;
  bool memoryExtracted = false; // 是否已提取记忆
  String? summary;            // 对话摘要（可选）
}

// ===== ChatMessage（对话消息）=====
@collection
class ChatMessageEntity {
  Id id = Isar.autoIncrement;
  String sessionId;
  String role;                // 'user' | 'assistant'
  String content;
  String? emotionTag;         // AI 回复的情绪标签
  DateTime createdAt;
  int? tokenCount;
  bool isTrainingMode = false; // 是否训练模式消息
  String? trainingDirection;   // 训练方向 ID
}

// ===== MemoryEntry（记忆条目）=====
@collection
class MemoryEntryEntity {
  Id id = Isar.autoIncrement;
  String companionId;
  String category;            // 'user_profile' | 'preference' | 'skill_level' | 'important_event' | 'goal' | 'relationship'
  String key;
  String value;
  double confidence;          // 置信度 0-1
  DateTime createdAt;
  DateTime lastUpdated;
  String sourceSessionId;
}

// ===== UserProfile（用户身份）=====
@collection
class UserProfileEntity {
  Id id = Isar.autoIncrement;
  String biometricId;         // 生物识别关联 ID
  String displayName;         // 显示名（可选）
  DateTime registeredAt;
}
```

### 4.2 数据访问层（DAO）

```dart
class CompanionDao {
  Future<CompanionEntity> create(CompanionEntity companion);
  Future<CompanionEntity?> getByUserId(String userId);
  Future<void> updateRapport(String companionId, double newRapport);
  Future<void> updateConsecutiveDays(String companionId, int days);
}

class TalentDao {
  Future<List<TalentEntity>> getActiveTalents(String companionId);
  Future<void> awakenTalent(TalentEntity talent);
  Future<void> updateTalent(TalentEntity talent);
  Future<void> removeTalent(String talentId);
}

class ChatDao {
  Future<ChatSessionEntity> createSession(String companionId, DateTime date);
  Future<ChatSessionEntity?> getSessionByDate(String companionId, DateTime date);
  Future<List<ChatSessionEntity>> getSessionHistory(String companionId, {int limit, int offset});
  Future<void> addMessage(ChatMessageEntity message);
  Future<List<ChatMessageEntity>> getMessages(String sessionId, {int limit, int offset});
  Future<List<ChatSessionEntity>> searchSessions(String companionId, String keyword);
}

class MemoryDao {
  Future<void> upsertMemory(MemoryEntryEntity entry);
  Future<List<MemoryEntryEntity>> getMemories(String companionId, {String? category});
  Future<void> deleteMemory(String id);
  Future<List<MemoryEntryEntity>> getRecentMemories(String companionId, {int limit = 50});
}
```

---

## 五、状态管理（Riverpod）

```dart
// ===== 认证与用户 =====
final authServiceProvider = Provider<AuthService>((ref) => AuthService());
final currentUserProvider = StateProvider<String?>((ref) => null); // userId

// ===== 伙伴 =====
final companionProvider = StateNotifierProvider<CompanionNotifier, AsyncValue<Companion?>>((ref) {
  final userId = ref.watch(currentUserProvider);
  return CompanionNotifier(ref, userId);
});

// ===== 对话 =====
final chatProvider = StateNotifierProvider<ChatNotifier, ChatState>((ref) {
  return ChatNotifier(ref);
});

// ===== 天赋 =====
final talentsProvider = FutureProvider<List<Talent>>((ref) async {
  final companion = await ref.watch(companionProvider.future);
  if (companion == null) return [];
  return ref.read(talentDaoProvider).getActiveTalents(companion.id);
});

// ===== 默契值 =====
final rapportProvider = Provider<double>((ref) {
  final companion = ref.watch(companionProvider).valueOrNull;
  return companion?.rapport ?? 0;
});

// ===== 形象 =====
final avatarProvider = StateNotifierProvider<AvatarNotifier, AvatarState>((ref) {
  return AvatarNotifier(ref);
});

// ===== 记忆 =====
final memoryProvider = FutureProvider<Map<String, List<MemoryEntry>>>((ref) async {
  final companion = await ref.watch(companionProvider.future);
  if (companion == null) return {};
  return ref.read(memoryManagerProvider).getGroupedMemories(companion.id);
});

// ===== Claude 客户端 =====
final claudeClientProvider = Provider<ClaudeClient>((ref) {
  final apiKey = ref.watch(apiKeyProvider);
  return ClaudeClient(apiKey: apiKey);
});

// ===== API Key =====
final apiKeyProvider = FutureProvider<String?>((ref) async {
  return ref.read(secureStorageProvider).read(key: 'claude_api_key');
});

// ===== 训练模式 =====
final trainingModeProvider = StateProvider<TrainingDirection?>((ref) => null);
```

---

## 六、关键流程时序

### 6.1 首次使用流程

```
User              App                AuthService        CompanionDao       AvatarService
 │                 │                     │                   │                  │
 │── 打开 App ───>│                     │                   │                  │
 │                 │── 检查注册状态 ────>│                   │                  │
 │                 │<── 未注册 ──────────│                   │                  │
 │                 │                     │                   │                  │
 │<── 欢迎页 ─────│                     │                   │                  │
 │── "我叫它小星"─>│                     │                   │                  │
 │                 │                     │                   │                  │
 │<── 形象创建页 ──│                     │                   │                  │
 │── "戴墨镜的猫" >│                     │                   │                  │
 │                 │── 生成形象 ────────────────────────────────────────────────>│
 │<── 4个候选方案 ─│                     │                   │                  │
 │── 选择方案 2 ──>│                     │                   │                  │
 │                 │── 生成表情包 ──────────────────────────────────────────────>│
 │                 │                     │                   │                  │
 │<── 设置生物识别 │                     │                   │                  │
 │── 录入指纹 ────>│── 注册用户 ────────>│                   │                  │
 │                 │<── userId ──────────│                   │                  │
 │                 │── 创建伙伴 ────────────────────────────>│                  │
 │                 │                     │                   │                  │
 │<── 首次对话 ────│  AI 主动打招呼       │                   │                  │
 │                 │  "嘿！我是小星..."   │                   │                  │
```

### 6.2 日常发送消息流程

```
User        ChatScreen     ChatNotifier    PromptBuilder    ClaudeClient    AvatarCtrl    Isar DB
 │              │               │               │               │              │            │
 │─ 输入消息 ──>│               │               │               │              │            │
 │              │─ send ───────>│               │               │              │            │
 │              │               │─ saveUserMsg ─────────────────────────────────────────────>│
 │              │               │               │               │              │            │
 │              │               │─ buildPrompt >│               │              │            │
 │              │               │  (companion   │               │              │            │
 │              │               │   + memory    │               │              │            │
 │              │               │   + talents   │               │              │            │
 │              │               │   + rapport)  │               │              │            │
 │              │               │<─ prompt ─────│               │              │            │
 │              │               │               │               │              │            │
 │              │               │─ stream ─────────────────────>│              │            │
 │              │               │               │               │─ HTTP SSE ─> Claude       │
 │              │               │               │               │<─ stream ─── Claude       │
 │              │               │<─ chunks ─────────────────────│              │            │
 │              │<─ update UI ──│               │               │              │            │
 │<─ 流式显示 ──│               │               │              │            │
 │              │               │               │               │              │            │
 │              │               │─ parseEmotion─│               │              │            │
 │              │               │─ updateAvatar────────────────────────────────>│            │
 │              │               │               │               │   (表情切换) │            │
 │              │               │               │               │              │            │
 │              │               │─ saveAiMsg ──────────────────────────────────────────────>│
 │              │               │─ updateRapport───────────────────────────────────────────>│
```

### 6.3 每日会话切换 & 天赋分析流程

```
App启动        SessionManager      MemoryExtractor     TalentAnalyzer      UI
  │                 │                    │                   │                │
  │─ getTodaySession>                    │                   │                │
  │                 │─ 检查今日会话       │                   │                │
  │                 │  (不存在→新建)      │                   │                │
  │                 │                    │                   │                │
  │                 │─ 提取前日记忆(异步)>│                   │                │
  │                 │                    │─ Claude API ─>    │                │
  │                 │                    │<─ 记忆条目 ───    │                │
  │                 │                    │─ 保存到 Isar      │                │
  │                 │                    │                   │                │
  │                 │─ 分析前日天赋(异步)────────────────────>│                │
  │                 │                    │                   │─ 检测信号       │
  │                 │                    │                   │─ 更新分数       │
  │                 │                    │                   │─ 检查觉醒       │
  │                 │                    │                   │                │
  │                 │  (如果有新天赋觉醒)  │                   │                │
  │                 │<─ TalentAwakening ─────────────────────│                │
  │                 │─ 通知 UI ──────────────────────────────────────────────>│
  │                 │                    │                   │    (觉醒动画!) │
  │                 │                    │                   │                │
  │                 │─ 应用用进废退 ─────────────────────────>│                │
  │                 │                    │                   │─ 衰减不活跃天赋│
```

---

## 七、UI/UX 技术方案

### 7.1 页面路由结构

```dart
final router = GoRouter(
  initialLocation: '/',
  redirect: (context, state) {
    final isAuthenticated = ref.read(currentUserProvider) != null;
    final isOnboarding = state.matchedLocation.startsWith('/onboarding');

    if (!isAuthenticated && !isOnboarding) return '/onboarding/welcome';
    return null;
  },
  routes: [
    // === 引导页 ===
    GoRoute(path: '/onboarding/welcome', builder: (_, __) => WelcomeScreen()),
    GoRoute(path: '/onboarding/name', builder: (_, __) => NameCompanionScreen()),
    GoRoute(path: '/onboarding/avatar', builder: (_, __) => CreateAvatarScreen()),
    GoRoute(path: '/onboarding/biometric', builder: (_, __) => BiometricSetupScreen()),

    // === 主页（对话为主界面）===
    ShellRoute(
      builder: (_, __, child) => MainShell(child: child),
      routes: [
        GoRoute(path: '/', builder: (_, __) => ChatScreen()),            // 对话 + 形象
        GoRoute(path: '/companion', builder: (_, __) => CompanionProfileScreen()), // 伙伴档案
      ],
    ),

    // === 对话历史 ===
    GoRoute(path: '/history', builder: (_, __) => ChatHistoryScreen()),
    GoRoute(path: '/history/:date', builder: (_, state) => ChatHistoryDetailScreen(
      date: state.pathParameters['date']!,
    )),

    // === 记忆管理 ===
    GoRoute(path: '/memory', builder: (_, __) => MemoryManageScreen()),

    // === 训练模式 ===
    GoRoute(path: '/training', builder: (_, __) => TrainingSelectScreen()),

    // === 设置 ===
    GoRoute(path: '/settings', builder: (_, __) => SettingsScreen()),
    GoRoute(path: '/settings/api-key', builder: (_, __) => ApiKeyScreen()),
    GoRoute(path: '/settings/biometric', builder: (_, __) => BiometricManageScreen()),
  ],
);
```

### 7.2 主界面布局

```
┌──────────────────────────────────┐
│  ≡  小星  ●  默契 72%    ⚙️     │ ← 顶栏：菜单、伙伴名、默契值、设置
│─────────────────────────────────│
│                                  │
│         ┌──────────┐             │
│         │          │             │ ← 形象区域（约占屏幕 1/3）
│         │  形象    │             │    Live2D/Rive 动画
│         │  (动态)  │             │    实时表情反应
│         │          │             │
│         └──────────┘             │
│  ┌──────┐ ┌──────┐ ┌──────┐    │ ← 天赋标签（最多5个）
│  │夜猫子│ │代码脑│ │话痨  │    │
│  └──────┘ └──────┘ └──────┘    │
│                                  │
│─────────────────────────────────│
│                                  │
│  ┌─────────────────────────┐    │ ← 消息区域（可滚动）
│  │ 用户消息                │    │
│  └─────────────────────────┘    │
│        ┌───────────────────┐    │
│        │ AI 回复（流式）    │    │
│        └───────────────────┘    │
│                                  │
│─────────────────────────────────│
│  ┌─────────────────────┐  📎  │ ← 输入区域
│  │ 输入消息...          │  ➤  │
│  └─────────────────────┘      │
│  [练口语] [练编程] [练写作]     │ ← 训练模式快捷入口
└──────────────────────────────────┘
```

### 7.3 主题设计

```dart
class AppColors {
  // 亮色主题
  static const primary = Color(0xFF6C63FF);       // 主色：紫蓝色
  static const secondary = Color(0xFF00D9A6);     // 辅色：翠绿色
  static const background = Color(0xFFF8F9FE);    // 背景色
  static const surface = Color(0xFFFFFFFF);        // 卡片色
  static const text = Color(0xFF2D3142);           // 主文字
  static const textSecondary = Color(0xFF9698A9);  // 次要文字
  static const userBubble = Color(0xFF6C63FF);     // 用户消息气泡
  static const aiBubble = Color(0xFFF0F0FF);       // AI 消息气泡

  // 天赋标签颜色
  static const talentColors = {
    TalentType.nightOwl:  Color(0xFF5C6BC0),  // 靛蓝
    TalentType.talkative: Color(0xFFFF8A65),  // 橙色
    TalentType.listener:  Color(0xFF81C784),  // 柔绿
    TalentType.comedian:  Color(0xFFFFD54F),  // 明黄
    TalentType.meticulous:Color(0xFF4DD0E1),  // 青色
    TalentType.coder:     Color(0xFF7C4DFF),  // 紫色
    TalentType.writer:    Color(0xFFFF5252),  // 红色
    TalentType.butler:    Color(0xFF448AFF),  // 蓝色
    TalentType.socialite: Color(0xFFFF4081),  // 粉色
    TalentType.bookworm:  Color(0xFF8D6E63),  // 棕色
    TalentType.fitCoach:  Color(0xFF66BB6A),  // 绿色
  };
}

class AppDarkColors {
  static const primary = Color(0xFF8B83FF);
  static const background = Color(0xFF1A1B2E);
  static const surface = Color(0xFF252842);
  // ...
}
```

### 7.4 关键 UI 组件

| 组件 | 描述 | 技术要点 |
|------|------|----------|
| AvatarDisplay | 伙伴形象展示 | Rive 状态机 / 图片序列帧 + 表情切换 |
| ChatBubble | 消息气泡 | Markdown 渲染，代码高亮，复制功能 |
| StreamingText | 流式文字 | 逐字显示动画 + StreamBuilder |
| TalentChip | 天赋标签 | 带颜色的 Chip，显示天赋名和强度 |
| RapportBar | 默契进度条 | 渐变色进度条，带数值显示 |
| TalentAwakeningOverlay | 觉醒仪式全屏覆盖 | 粒子动画 + 文字逐字 + 形象特效 |
| TrainingQuickBar | 训练模式快捷栏 | 横向滚动的训练方向卡片 |
| MemoryCard | 记忆条目卡片 | 分类显示 + 滑动删除 |
| EmotionIndicator | 情绪指示器 | 形象旁的小气泡，显示当前情绪 |

---

## 八、安全方案

| 安全项 | 方案 |
|--------|------|
| API Key 存储 | `flutter_secure_storage` 加密存储，不以明文写入任何文件 |
| 生物识别数据 | 由系统管理，App 仅获取认证结果，不存储生物特征 |
| 网络传输 | 全 HTTPS，API Key 通过 Header 传递 |
| 本地数据 | Isar 数据库文件存于 App 沙盒，按用户隔离，随 App 卸载删除 |
| Prompt 注入防护 | 用户输入不直接拼接到 System Prompt，使用 User Message |
| Token 限制 | 单次请求设置 maxTokens 上限，防止异常消耗 |
| 形象资源 | 存于 App 沙盒，不上传到任何服务器 |

---

## 九、性能优化策略

| 场景 | 优化方案 |
|------|----------|
| 消息列表滚动 | `ListView.builder` 懒加载 + 分页查询 |
| 流式渲染 | `RepaintBoundary` 隔离重绘区域，文字区和形象区独立重绘 |
| 形象动画 | Rive 状态机驱动，GPU 渲染，不阻塞主线程 |
| 数据库查询 | Isar 异步查询 + 索引优化（sessionDate、companionId） |
| 记忆提取 | 异步后台执行，不阻塞用户交互 |
| 天赋分析 | 批量分析，每日仅执行一次 |
| System Prompt | 记忆条目按相关性排序，控制 Token 总量（<1000 tokens） |
| 首屏加载 | 骨架屏 + 渐进式数据加载，形象资源预缓存 |
| 图片生成 | 形象创建时一次性生成，后续从本地加载 |

---

## 十、项目依赖清单（pubspec.yaml）

```yaml
dependencies:
  flutter:
    sdk: flutter

  # 状态管理
  flutter_riverpod: ^2.5.1
  riverpod_annotation: ^2.3.5

  # 路由
  go_router: ^14.2.0

  # 网络
  dio: ^5.4.3+1

  # 本地存储
  isar: ^3.1.0+1
  isar_flutter_libs: ^3.1.0+1
  flutter_secure_storage: ^9.2.2
  shared_preferences: ^2.2.3

  # 生物识别
  local_auth: ^2.2.0

  # UI 增强
  flutter_animate: ^4.5.0
  flutter_markdown: ^0.7.3+1
  google_fonts: ^6.2.1
  rive: ^0.13.0              # 交互式动画引擎

  # 工具
  uuid: ^4.4.2
  intl: ^0.19.0
  json_annotation: ^4.9.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  build_runner: ^2.4.9
  json_serializable: ^6.8.0
  isar_generator: ^3.1.0+1
  riverpod_generator: ^2.4.0
  flutter_lints: ^3.0.2
```

---

## 十一、目录结构

```
ai_companion/
├── lib/
│   ├── main.dart                              # 入口：初始化 Isar、ProviderScope
│   ├── app.dart                               # MaterialApp + GoRouter + Theme
│   │
│   ├── core/
│   │   ├── api/
│   │   │   ├── claude_client.dart             # Claude API 封装（流式 + 非流式）
│   │   │   ├── prompt_builder.dart            # System Prompt 动态组装
│   │   │   ├── image_gen_client.dart          # 图片生成 API 封装
│   │   │   └── models/
│   │   │       ├── api_message.dart
│   │   │       ├── api_response.dart
│   │   │       └── stream_event.dart
│   │   │
│   │   ├── database/
│   │   │   ├── app_database.dart              # Isar 初始化
│   │   │   └── daos/
│   │   │       ├── companion_dao.dart
│   │   │       ├── talent_dao.dart
│   │   │       ├── chat_dao.dart
│   │   │       ├── memory_dao.dart
│   │   │       └── user_dao.dart
│   │   │
│   │   ├── auth/
│   │   │   └── auth_service.dart              # 生物识别认证
│   │   │
│   │   ├── theme/
│   │   │   ├── app_theme.dart                 # 亮色/暗色主题定义
│   │   │   └── app_colors.dart                # 色彩常量
│   │   │
│   │   └── utils/
│   │       ├── constants.dart
│   │       └── extensions.dart
│   │
│   ├── features/
│   │   ├── onboarding/
│   │   │   ├── screens/
│   │   │   │   ├── welcome_screen.dart        # 欢迎页
│   │   │   │   ├── name_companion_screen.dart # 命名伙伴
│   │   │   │   ├── create_avatar_screen.dart  # 创建形象（AI辅助生成）
│   │   │   │   └── biometric_setup_screen.dart # 设置生物识别
│   │   │   └── widgets/
│   │   │       └── avatar_candidate_card.dart
│   │   │
│   │   ├── chat/
│   │   │   ├── models/
│   │   │   │   └── chat_state.dart
│   │   │   ├── providers/
│   │   │   │   └── chat_provider.dart
│   │   │   ├── services/
│   │   │   │   └── session_manager.dart       # 每日会话管理
│   │   │   ├── screens/
│   │   │   │   ├── chat_screen.dart           # 主聊天界面（含形象）
│   │   │   │   └── chat_history_screen.dart   # 按日归档的历史
│   │   │   └── widgets/
│   │   │       ├── chat_bubble.dart
│   │   │       ├── chat_input.dart
│   │   │       ├── streaming_text.dart
│   │   │       └── training_quick_bar.dart    # 训练模式快捷栏
│   │   │
│   │   ├── companion/
│   │   │   ├── models/
│   │   │   │   └── companion.dart             # 伙伴领域模型
│   │   │   ├── providers/
│   │   │   │   └── companion_provider.dart
│   │   │   ├── screens/
│   │   │   │   └── companion_profile_screen.dart  # 伙伴档案
│   │   │   └── widgets/
│   │   │       ├── rapport_bar.dart
│   │   │       └── talent_chip.dart
│   │   │
│   │   ├── talent/
│   │   │   ├── models/
│   │   │   │   ├── talent.dart                # 天赋领域模型
│   │   │   │   └── talent_metadata.dart       # 天赋元数据配置
│   │   │   ├── services/
│   │   │   │   ├── talent_analyzer.dart       # 天赋分析引擎
│   │   │   │   └── signal_detectors.dart      # 信号检测器
│   │   │   └── widgets/
│   │   │       └── talent_awakening_overlay.dart # 觉醒仪式动画
│   │   │
│   │   ├── avatar/
│   │   │   ├── models/
│   │   │   │   └── avatar_asset_pack.dart
│   │   │   ├── services/
│   │   │   │   └── avatar_service.dart        # 形象生成与管理
│   │   │   └── widgets/
│   │   │       ├── avatar_display.dart        # 形象动画展示
│   │   │       └── avatar_animation_controller.dart
│   │   │
│   │   ├── memory/
│   │   │   ├── services/
│   │   │   │   ├── memory_extractor.dart      # 记忆提取
│   │   │   │   └── memory_manager.dart        # 记忆管理
│   │   │   ├── providers/
│   │   │   │   └── memory_provider.dart
│   │   │   ├── screens/
│   │   │   │   └── memory_manage_screen.dart  # 记忆查看/删除
│   │   │   └── widgets/
│   │   │       └── memory_card.dart
│   │   │
│   │   ├── training/
│   │   │   ├── models/
│   │   │   │   └── training_direction.dart
│   │   │   └── screens/
│   │   │       └── training_select_screen.dart
│   │   │
│   │   └── growth/
│   │       └── services/
│   │           ├── rapport_system.dart        # 默契值系统
│   │           └── growth_narrative.dart       # AI 自述成长
│   │
│   ├── settings/
│   │   ├── screens/
│   │   │   ├── settings_screen.dart
│   │   │   ├── api_key_screen.dart
│   │   │   └── biometric_manage_screen.dart
│   │   └── providers/
│   │       └── settings_provider.dart
│   │
│   └── shared/
│       ├── widgets/
│       │   ├── app_scaffold.dart
│       │   └── loading_indicator.dart
│       └── models/
│           └── result.dart
│
├── assets/
│   ├── animations/                            # Rive/Lottie 动画文件
│   │   ├── awakening_effect.riv              # 觉醒动画
│   │   └── idle_animations.riv              # 待机动画
│   ├── images/
│   │   └── onboarding/                       # 引导页插图
│   └── fonts/
│
├── test/
│   ├── core/
│   │   ├── api/
│   │   └── database/
│   ├── features/
│   │   ├── talent/                           # 天赋分析测试
│   │   ├── memory/                           # 记忆系统测试
│   │   └── chat/
│   └── integration/
│
└── pubspec.yaml
```

---

## 十二、开发阶段划分

### Phase 1: 基础框架搭建
- 创建 Flutter 项目 + 配置依赖
- 建立完整目录结构
- 实现主题系统（亮/暗）
- 配置路由（GoRouter）
- 初始化 Isar 数据库 + 定义所有 Collection
- 实现生物识别认证服务

### Phase 2: 引导流程
- 欢迎页 UI
- 命名伙伴页
- 形象创建页（先用占位图，后续接入图片生成）
- 生物识别绑定页
- 伙伴数据创建与持久化

### Phase 3: Claude API + 核心对话
- Claude API 客户端封装（含流式 SSE）
- System Prompt 动态组装（基础版）
- 每日会话机制（SessionManager）
- 聊天界面 UI（消息列表 + 输入框 + 形象区域）
- 消息发送 + 流式接收 + 持久化
- 情绪标签解析 + 形象表情联动
- API Key 配置页

### Phase 4: 记忆系统
- 记忆数据模型 + DAO
- 每日对话结束后自动提取记忆（Claude API）
- 记忆注入 System Prompt
- 记忆管理页面（查看/删除）

### Phase 5: 天赋系统
- 天赋元数据配置
- 信号检测器（关键词、时间、对话模式）
- 天赋分析引擎（累积、觉醒、用进废退）
- 天赋觉醒仪式动画
- 天赋标签 UI（伙伴档案 + 对话界面）
- 天赋对 System Prompt 的影响

### Phase 6: 成长体系 + 训练模式
- 默契值系统
- AI 自述成长（Prompt 里程碑引导）
- 伙伴档案页（形象 + 名字 + 天赋 + 默契值 + AI 自述）
- 训练模式入口和训练对话
- 对话历史按日归档 + 搜索

### Phase 7: 形象系统完善
- 接入图片生成 API
- AI 辅助形象创建完整流程
- 情绪表情包生成
- 天赋视觉进化叠加层
- 待机动画 + 时段感知

### Phase 8: 打磨优化
- 觉醒动画打磨
- 加载骨架屏
- 异常处理 + 错误页面
- 性能优化（重绘隔离、分页、缓存）
- 离线模式（无网络时查看历史、档案、形象）
- 整体测试
