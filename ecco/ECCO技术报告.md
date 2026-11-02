# ECCO 开局名称「边走边播报」技术报告

> 目标：任何安卓象棋 App 都能在走子时实时显示「当前开局/变例名称 + ECCO 编号」，如官方皮卡鱼新版、黄昏象棋。

---

## 一、你上传的两个文件是什么

| 文件 | 身份 | 说明 |
|---|---|---|
| `ECCO.DLL` | **PE32（Windows 32 位 DLL）** | 象棋巫师系 ECCO 判定内核的桌面版 |
| `libecco.so` | **Android JNI 动态库** | 同一判定内核的安卓版（官方皮卡鱼使用） |

两者是**同一套判定逻辑的两个平台编译产物**。它们提供的能力是：

```
输入：一串着法（C2.5 记谱 / ICCS / FEN 序列）
输出：ECCO 编号（如 "B31"）
```

**注意：内核只给编号，不给中文名。** 名称必须由另一张「ECCO 编号 → 开局名/变例名」的字典表查出来。

---

## 二、原理（三步，缺一不可）

### 1) 走法标准化：UCI/坐标 → C2.5 纵线记谱

ECCO 判定内核吃的是 **C2.5 记法**（象棋巫师记谱），不是 UCI。转换规则：

- 棋子字母：`K`帅 `A`士 `B`相 `R`车 `N`马 `C`炮 `P`兵
- 路数编号按**行棋方视角**从右到左 1..9（红黑各自一套）
- 三种动作：
  - **平**：同排移动 `字母起路.终路`（如 `C2.5`）
  - **进/退**：`字母起路+步数` 或 `-步数`；车/炮/兵/帅用**位移步数**，马/相/士用**目标路**

### 2) 编号判定：C2.5 序列 → ECCO 编号

把整局（或前 N 步）的 C2.5 串喂给判定内核，得到编号。判定的本质是**开局树匹配**：

```
root → 第1步 → 第2步 → … → 命中变例节点 → 返回该节点的 ECCO 编号
```

未命中（超出体系覆盖）应当返回「空」，**绝不能猜**——猜就会显示错名。

### 3) 编号 → 名称：字典反查

一张表即可：

```
B31 → 中炮对反宫马
C21 → 中炮过河车对屏风马
…
```

实时播报 = 每走一步都重新走一遍 1→2→3，耗时在微秒级，可直接在主线程做（也可做增量缓存：记住已匹配的树节点，下一步只在子节点里找）。

---

## 三、三种落地方式（按接入成本排序）

| 方式 | 需要什么 | 优点 | 缺点 |
|---|---|---|---|
| **A. 纯 Java/Kotlin 复刻判定** | 无（零依赖） | 体积最小、无 ABI 限制、易改 | 需要准确复刻判定逻辑（我们用的就是这条） |
| **B. JNI 调 `libecco.so`** | 各 ABI 的 so | 与官方皮卡鱼**完全一致** | 需打包 so、需 `System.loadLibrary` |
| **C. 服务端/本地 HTTP** | 单独进程 | 逻辑隔离 | 延迟、部署复杂 |

**推荐给第三方 App**：先用 A（几十 KB，无坑）；要「和官方皮卡鱼一模一样」再上 B。

---

## 四、黄昏象棋的实际实现（可直接拿去用）

源码两个文件，无第三方依赖：

- `EccoNamer.kt` —— 第 1 步：走法 → C2.5 串
- `EccoDict.kt` —— 第 3 步：编号 → 中文名（含变例名）

### 核心接口（极简）

```kotlin
// 走一步，拿到 C2.5 记法；无法识别返回 null
EccoNamer.toC25(fromRow, fromCol, toRow, toCol, type, color): String?

// 累积若干步后，得到「开局名 / 变例名」，未覆盖返回 null
EccoNamer.nameOf(moves: List<String>): EccoDict.Entry?
```

### 接入到任意 Android 象棋 App 的四步

```kotlin
// 1. 走子成功时，把这一步转成 C2.5，追加到列表
val c25 = EccoNamer.toC25(fr, fc, tr, tc, piece.type, piece.color)
if (c25 != null) c25List.add(c25)

// 2. 立刻查询（微秒级，无需异步）
val entry = EccoNamer.nameOf(c25List)

// 3. 有结果才显示；null 就隐藏这一行（宁可不显示，也不显示错名）
if (entry != null) {
    tvOpening.text = "${entry.code}  ${entry.name}" + (entry.variation?.let { " · $it" } ?: "")
}

// 4. 悔棋/重开时同步截断列表
c25List = c25List.subList(0, moveIndex)
```

### 关键设计原则（踩过的坑）

1. **红黑路数各自从右往左**，不能用绝对列号，否则黑方全部反向；
2. **马/相/士**用「目标路」，**车/炮/兵/帅**用「位移步数」，混用必错；
3. **未覆盖就返回 null**，UI 隐藏；绝不 fallback 到“最接近的变例”；
4. 每步都全量重算（列表很短，成本可忽略），比维护增量状态更不易错。

---

## 五、如果要「和官方皮卡鱼一模一样」（JNI 方案）

```kotlin
object EccoNative {
    init { System.loadLibrary("ecco") }          // libecco.so
    external fun lookup(c25: String): String?     // 返回 "B31" 或 null
}

val code = EccoNative.lookup(c25List.joinToString(" "))
val name = code?.let { EccoDict.of(it) }
```

打包要点：

```
app/src/main/jniLibs/arm64-v8a/libecco.so
app/src/main/jniLibs/armeabi-v7a/libecco.so
```

`abiFilters` 里显式包含，避免被裁剪。

**对拍验证（必做）**：同一批棋谱分别走 A 与 B，逐局比对编号；不一致的局就是复刻逻辑的缺口，逐条修。

---

## 六、数据与授权（分享时必须说明）

- **ECCO 编号体系与开局命名**：源自《象棋百科全书》/象棋巫师的 ECCO 体系，**表数据有版权**；
- **libecco / ECCO.DLL**：随皮卡鱼（Pikafish）分发，**Pikafish 为 GPL-3.0**；
- 因此分享给第三方 App 时：
  - 自己复刻的**判定逻辑代码**可以自由分发；
  - **ECCO 名称字典**建议注明来源，或让使用者自行提供/替换字典文件；
  - 直接打包 `libecco.so` 需遵守 **GPL-3.0**（开源其 App 或提供对应源码）。

---

## 七、结论（一句话）

**这套技术的本质不是“库”，而是三件小事：C2.5 记谱转换 + 开局树匹配出编号 + 编号查名。**
任何安卓象棋 App 用一份几十 KB 的 Kotlin 代码即可复刻；要与官方皮卡鱼完全一致，再挂上 `libecco.so`。

---

附：`EccoNamer.kt`、`EccoDict.kt` 已随本报告一并打包，可直接拷进他人工程。
