# 请解释一下Kotlin的契约机制。

Kotlin 的**契约机制（Contract）** 是 Kotlin 1.3+ 引入的编译期工具，核心目的是**让开发者显式声明函数的行为约定**，帮助编译器理解函数的隐含逻辑（如参数与返回值的关系、Lambda 执行时机等），从而优化类型推断、空安全检查和智能转换，减少不必要的样板代码（如强制拆箱）。


### 一、核心价值：弥补编译器的“认知不足”
编译器默认只能分析函数的语法结构，无法理解函数的“业务约定”。例如：
- 你写了一个 `isNonNull(x)` 函数，返回 `true` 表示 `x` 非空，但编译器不知道这个约定，仍会认为 `x` 是可空的；
- 你写了一个自定义作用域函数 `myRun(block)`，编译器不知道 `block` 是否会立即执行，可能阻止你在 `block` 中初始化变量。

契约机制就是把这些**隐含约定显式告诉编译器**，让编译器做出更精准的判断，同时不影响运行时性能（契约仅在编译期生效，无运行时开销）。


### 二、基本用法
契约通过 `@kotlin.contracts.Contract` 注解和 `contract` 函数声明，核心规则：
1. `contract` 函数必须是**函数体的第一条语句**；
2. 仅支持顶层函数、成员函数或内联函数（部分契约如 `callsInPlace` 仅支持内联函数）；
3. 契约是“编译期承诺”，开发者需确保契约与函数实际行为一致（否则会导致运行时错误）。

#### 核心 API：`ContractBuilder`
通过 `contract { ... }` 块构建契约，`ContractBuilder` 提供了常用的契约声明方法：
| 方法                | 作用                                  | 适用场景                          |
|---------------------|---------------------------------------|-----------------------------------|
| `returns(value)`    | 声明函数返回特定值时的约定            | 关联返回值与参数状态（如返回 `true` 时参数非空） |
| `returnsNotNull()`  | 声明函数返回值一定非空                | 空安全优化（避免强制拆箱）        |
| `callsInPlace(block, kind)` | 声明 Lambda 会被立即执行（及执行次数） | 自定义作用域函数（如 `run`、`apply`） |
| `throws(exception)` | 声明函数可能抛出的异常                | 异常提示（Kotlin 不强制异常声明）  |


### 三、常见契约场景（附代码示例）
#### 场景 1：空安全与参数-返回值关联
最常用场景：声明“返回值与参数状态的关系”（如返回 `true` 时参数非空）。

```kotlin
import kotlin.contracts.Contract
import kotlin.contracts.contract

// 自定义“非空且非空字符串”检查函数
@Contract("it -> true") // 字符串简写：参数（it）满足条件时返回true
fun String?.isNotEmptyOrNull(): Boolean {
    contract {
        // 契约：当函数返回true时，当前对象（this@isNotEmptyOrNull）非空
        returns(true) implies (this@isNotEmptyOrNull != null)
    }
    return this != null && this.isNotEmpty()
}

fun main() {
    val str: String? = "Kotlin"
    if (str.isNotEmptyOrNull()) {
        // 编译器通过契约识别：str此时非空，可安全调用length
        println(str.length) // 无编译错误，无需 !! 强制拆箱
    }
}
```

#### 场景 2：Lambda 执行时机（`callsInPlace`）
自定义作用域函数时，用 `callsInPlace` 告诉编译器“Lambda 会立即执行”，确保变量初始化的正确性。

```kotlin
import kotlin.contracts.Contract
import kotlin.contracts.contract
import kotlin.contracts.InvocationKind

// 自定义类似 run 的作用域函数
@Contract(pure = false) // pure=false：函数有副作用（执行Lambda）
inline fun <T> myRun(block: () -> T): T {
    contract {
        // 声明：block会被精确执行一次（InvocationKind.EXACTLY_ONCE）
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}

fun main() {
    val x: Int // 未初始化
    myRun {
        x = 10 // 编译器通过契约知道Lambda会执行，允许在此初始化
    }
    println(x) // 正常输出 10（无“未初始化”编译错误）
}
```

`InvocationKind` 可选值：
- `EXACTLY_ONCE`：执行一次（如 `run`）；
- `AT_LEAST_ONCE`：至少执行一次（如 `repeat`）；
- `AT_MOST_ONCE`：最多执行一次（如 `takeIf` 中的Lambda）；
- `UNKNOWN`：未知次数（默认）。

#### 场景 3：声明返回值非空（`returnsNotNull`）
明确函数返回值一定非空，避免编译器提示空安全警告。

```kotlin
import kotlin.contracts.Contract
import kotlin.contracts.contract

@Contract("-> not null") // 字符串简写：返回值非空
fun getNonNullName(): String {
    contract {
        returnsNotNull() // 契约：返回值一定非空
    }
    return "Alice" // 实际逻辑确保非空
}

fun main() {
    // 编译器通过契约识别：返回值非空，可直接赋值给非空变量
    val name: String = getNonNullName() 
    println(name.length) // 无编译错误
}
```


### 四、契约的限制与注意事项
1. **契约必须与实际行为一致**：  
   若契约声明“返回 `true` 时参数非空”，但函数实际返回 `true` 时参数可能为空，会导致运行时空指针异常（编译器会允许不安全调用）。

2. **仅编译期生效，无运行时检查**：  
   契约不影响代码运行，仅用于编译器推断，开发者需对契约的正确性负责。

3. **`callsInPlace` 仅支持内联函数**：  
   非内联函数的Lambda可能被传递到其他地方执行，无法保证执行时机，因此 `callsInPlace` 只能用于内联函数。

4. **`contract` 块必须在函数体开头**：  
   若 `contract` 函数不是函数体第一条语句，会直接编译错误。

5. **字符串简写 vs 函数式API**：  
   早期支持字符串简写（如 `@Contract("it -> true")`），但推荐使用 `contract` 块的函数式API（更灵活、类型安全）。


### 五、标准库中的契约应用
Kotlin 标准库大量使用契约优化语法体验，例如：
- 作用域函数（`run`、`apply`、`let`）：通过 `callsInPlace` 确保Lambda执行时机；
- 空检查函数（`requireNotNull`、`checkNotNull`）：通过 `returnsNotNull` 声明返回值非空；
- 条件函数（`takeIf`、`takeUnless`）：通过 `returns` 关联返回值与参数状态。

示例：标准库 `requireNotNull` 的契约（简化版）：
```kotlin
@Contract("null -> fail; !null -> param1")
fun <T> requireNotNull(value: T?): T {
    contract {
        returnsNotNull()
        returns(false) implies (value == null)
    }
    if (value == null) throw IllegalArgumentException()
    return value
}
```


### 总结
Kotlin 契约机制的核心是**“编译期行为约定”**：通过显式声明函数的隐含逻辑，帮助编译器突破语法分析的局限，提供更精准的类型推断和空安全检查。其价值在于：
- 减少样板代码（如 `!!` 强制拆箱）；
- 提升代码安全性（避免隐式空指针）；
- 支持自定义函数的“语法优化”（如自定义作用域函数）。

使用契约时，关键是确保契约与函数实际行为一致，否则会引入潜在风险。日常开发中，除了自定义契约，更要理解标准库契约的设计，充分利用其带来的语法便利。