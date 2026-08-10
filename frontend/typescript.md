# 前端 · TypeScript（面试题库）

本文件考察前端工程师在 TypeScript 上的**真实工程实践能力**，重点包括：类型设计（any/unknown/never 的选择、联合类型收窄、对象类型建模）、泛型与类型运算（extends/keyof/infer、映射与条件类型）、类型安全的数据流（API 层类型化、运行时校验）、以及与 React/Vue 框架结合、严格模式与渐进迁移、any 泄漏治理、类型测试与工程规范。题目均为场景化提问，不考概念背诵，侧重候选人在真实项目中的取舍判断、问题定位与边界意识。

### Q1. any / unknown / never 如何选，any 是怎么泄漏的

**问题：** 项目里有人图省事把接口返回类型写成 any，后来排查线上 bug 时发现类型系统完全没拦住。你怎么从类型层面判断什么时候用 any、unknown、never，又怎么阻止 any 悄悄扩散到整个项目？

**期望加分项：**
- 能说清三者语义差异：any 关闭检查、unknown 是顶部类型需收窄、never 是底部类型表示不可达/穷尽
- 能指出 any 泄漏的具体路径：显式 any、`as any`、第三方包返回 any、`JSON.parse` 返回 any、数组解构
- 知道 `unknown` 的收窄代价（每次使用都要守卫），以及 never 用于穷尽性检查的写法
- 能给出工程化约束：`@typescript-eslint/no-explicit-any` + `no-unsafe-*` 系列规则
- 有线上事故或踩坑案例佐证
**减分项：**
- 只会背定义，举不出真实场景
- 不区分 unknown 与 any 的实际使用差异
- 认为 never 只是理论概念、用不到
- 说不出 any 是怎么从入口污染到整个数据流的

**解答：**
判断思路：能用明确类型就用明确类型；暂时拿不准、但使用前会收窄的用 `unknown`；只有「确实不关心类型」或「迁移过渡期」才允许 any，且必须立刻加 lint 约束。never 平时用不上，但它是穷尽性检查的安全网。先看收窄与穷尽检查的写法：

```ts
function isUser(v: unknown): v is User {
  return typeof v === 'object' && v !== null && 'id' in v
}
function assertNever(x: never): never {
  throw new Error('unexpected value: ' + JSON.stringify(x))
}
switch (action.type) {
  case 'ADD': ...
  default: assertNever(action) // 新增 type 分支忘处理，这里编译报错
}
```

实践中的坑：any 最危险的扩散路径不是显式写，而是 `JSON.parse`（返回 any）、第三方包回包、数组解构后的元素——一旦进了数据流，后面所有推导都被污染。`as any` 还会静默绕过只读/必选约束，比如把必填字段断言掉。典型线上事故：后端接口字段改名后旧代码仍能编译，因为上游 any 掩盖了错误——类型系统给了虚假安全感，这才是 any 最大的代价。约束手段：eslint 开 `no-explicit-any`（显式 any 直接报错）+ `no-unsafe-assignment / argument / member-access`（专门拦截 any 的下游泄漏点），新文件强制全类型，老文件用 type-coverage 量化（详见 Q17）。

**延伸考点：** 面试官追问：如果把所有 any 换成 unknown，你的代码要付出什么代价？什么时候 any 反而比 unknown 合理？

### Q2. 联合类型的收窄：判别联合、in、is 的适用场景与失效边界

**问题：** 组件里 `props` 可能是 `{ type: 'a'; data: X } | { type: 'b'; data: Y }` 这样的联合类型，你在 switch 里想用 `props.data`，编辑器却报类型错误。怎么正确收窄？什么场景下收窄会失效甚至误导你？

**期望加分项：**
- 能说出 in、typeof、instanceof、判别联合（discriminated union）、自定义守卫 `is` 各自的适用场景
- 知道判别联合要选字面量类型字段做判别，switch 后各分支自动收窄
- 能举出收窄失效的实例：可选属性、`in` 判断不了「存在但为 undefined」、闭包/回调里重赋值、解构发生在收窄前
- 知道 `is` 守卫是编译期断言，运行时只执行你写的逻辑，写错会静默埋雷
**减分项：**
- 只会说 `if (typeof x === 'string')`，举不出更复杂的场景
- 把 TS 收窄当成运行时机制（实际是编译期静态分析）
- 不知道判别联合的好处与边界：共享字段可访问、非共享字段必须收窄后才能访问
- 说不出收窄失效导致线上问题的案例

**解答：**
判别联合是首选：共用一个字面量类型字段做判别，switch 后每个分支自动收窄，代码即文档：

```ts
type Msg = { kind: 'text'; text: string } | { kind: 'img'; url: string }
function render(m: Msg): string {
  switch (m.kind) {
    case 'text': return m.text
    case 'img':  return m.url
  }
}
```

如果判别字段是普通 string 而非字面量联合，收窄会退化，此时用 `'data' in props` 判断存在性。自定义守卫适用于「无法静态判断」的运行时形状：

```ts
function isProduct(v: unknown): v is Product {
  return typeof v === 'object' && v !== null && 'id' in v
}
```

实践中的坑：1) `in` 对可选属性也返回 true——`{ a?: string }` 与 `{ b: string }` 的联合里，`'a' in x` 无法区分「存在但 undefined」，要结合 `!== undefined` 判断；2) 闭包/回调内 TS 不追踪变量重赋值，收窄在函数边界失效，需要先 `const v = x` 再使用；3) 解构发生在收窄前，会把「可空字段」解构成 undefined 丢失收窄信息，应先收窄再解构；4) `is` 守卫是给编译器的断言，函数体没真正校验时，运行时照样拿错数据——守卫要写得和类型声明一样严格。

**延伸考点：** `props.data` 在收窄前访问会报错，但如果用一个可选属性做判别字段，收窄还可靠吗？为什么推荐把判别字段设计成必选字面量？

### Q3. interface 与 type 别名：语义定位与团队统一

**问题：** 团队代码里有人用 interface 有人用 type，遇到两个类型合并时还有人写 interface 继承。你怎么给团队定规矩？两者到底差在哪，为什么这个选择会影响一个大型项目的长期维护？

**期望加分项：**
- 说清核心差异：interface 可声明合并（declaration merging）与继承，type 可表达联合、映射、条件等一切计算类型
- 能给出团队约定：描述「对象形状 + 可扩展」用 interface，表达「计算出来的类型」用 type
- 知道同名 interface 会被无声合并、type 重名直接报错，明白这是差异不是 bug
- 能谈库作者视角：interface 对外部扩展更友好
- 不拿「性能差异」当主要理由（对普通项目基本无感）
**减分项：**
- 认为两者完全等价、随便用
- 不知道声明合并的存在
- 用 interface 表达联合类型报错时一脸懵
- 只会背官方文档的等价性表格，给不出团队可执行的标准

**解答：**
两者在大多数场景可互换，但语义定位不同：interface 描述「契约 / 形状」，支持声明合并与继承，适合对外 API、组件 props；type 是「类型的计算」，能表达联合、交叉、映射、条件等一切形式，适合内部推导与工具类型结果。给团队的建议可以是一句话：**「是什么」用 interface，「怎么算出来」用 type**；交叉类型（`&`）与继承（extends）二选一，不要混用。

实践中的坑：1) 同名 interface 会被自动合并，多人协作时可能无意扩充了公共类型，type 则直接报重名错误——这不是 bug，而是 interface 的「开放」特性，要意识到它意味着全局命名空间共享；2) 深度继承链的报错信息可读性差，层级多时错误会指向很远的祖先；3) 类 `implements` 接口时，interface 的成员缺失会给出较清晰的报错，而 type 表达复杂交叉类型时错误信息可能绕；4) 第三方库的类型修复经常靠 interface 声明合并（module augmentation），这也是 interface 的实用价值。团队规范的核心不是「哪个对」，而是「统一」，因为混用导致的是代码审查噪音和新人困惑。

**延伸考点：** 如果第三方库的某个 interface 你需要在业务侧补充字段，用 interface 还是 type 能实现？这背后的机制是什么？

### Q4. enum 与 as const：序列化、编译产物与选型

**问题：** 重构一个老项目，代码里有一堆 `enum` 用于状态码，但发现 `console.log` 打印出的是数字、前后端对不上号。现在你给团队定规矩：什么时候用 enum、什么时候用 `as const` 对象？反过来，什么场景必须用 enum？

**期望加分项：**
- 知道数字枚举有反向映射（编译产物含 `Enum[Enum.A]` 对象），字符串枚举没有
- 知道 `const enum` 会被内联，与 isolatedModules（Vite 默认）存在冲突
- 能写出 `as const` + `keyof typeof` 推导联合类型的现代方案
- 能谈序列化、tree-shaking、可读性与前后端约定的联动
- 知道必须用 enum 的场景：运行时反向查找（值→键）、遍历枚举成员
**减分项：**
- 只会背「能用 const 就别用 enum」，说不出理由
- 不知道数字枚举反向映射带来的运行时产物副作用
- 说不出 as const 方案的具体推导写法
- 不考虑与后端约定、落库数据的联动

**解答：**
现代项目默认优先 `as const` 对象：

```ts
const Status = { Draft: 'DRAFT', Published: 'PUBLISHED' } as const
type Status = typeof Status[keyof typeof Status] // 'DRAFT' | 'PUBLISHED'
```

好处：运行时就是普通对象，可序列化、可 JSON 传递、可 tree-shaking，没有隐藏的反向映射产物。数字枚举的问题在于编译产物里会生成 `Enum[Enum.A]` 反向映射对象，一旦状态码作为接口字段传递，对方拿到的是难以读懂的裸数字，且调整枚举成员顺序会改变值、破坏已落库数据。

必须用 enum 的场景：1) 需要值→键的反向查找且不想手写映射表；2) 遗留代码约定，迁移成本大于收益；3) 某些库（如后端 SDK）要求枚举语义。另外注意 `const enum`：它会内联为字面量，在 isolatedModules（Vite/tsup 默认启用）下跨模块引用会有问题，官方明确不建议这种组合，babel 转译也不支持。实践中的坑：枚举成员做联合类型时，`keyof typeof Enum` 拿到的是键的联合而不是值的联合，as const 方案里两者都可推导；团队里前后端共用一套状态码时，用 as const + 生成文档的方式更容易对齐契约。

**延伸考点：** 如果后端把状态码定义为字符串常量表，前端要用 TS 类型表达「所有合法值」且不让运行时存在，你会怎么设计？`satisfies` 在这个场景有什么用？

### Q5. 索引签名：动态 key 对象的安全建模

**问题：** 给对象加索引签名 `[key: string]: string` 时编译器报错说和某个具名属性不兼容；或者遍历对象做统计时，编辑器提示取值「可能为 undefined」。怎么设计带动态 key 的对象类型？什么情况应该改用 Map 或 Record？

**期望加分项：**
- 知道索引签名的语义：约束对象所有属性都满足该签名，具名属性也必须兼容
- 知道 key 类型只能是 string / number / symbol 或其字面量联合
- 会用 `Record<string, T>`、`Partial<Record<K, T>>`、Map 替代裸索引签名
- 知道 `noUncheckedIndexedAccess` 开启后读取返回 `T | undefined`，以及它的取舍
- 知道模板字面量受限 key（TS 4.4+）的写法与局限
**减分项：**
- 不知道索引签名会「污染」同对象其他具名属性
- 一遇动态对象就 `[key: string]: any`
- 不知道 `Object.keys` 返回 `string[]` 与索引签名 key 类型不一致的坑
- 分不清对象 vs Map 的选择依据

**解答：**
先判断业务本质：动态 key 是「已知有限集合」还是「完全未知」？有限集合用字面量联合 + 可选属性；完全未知且值同构，用 `Record<string, T>`；需要频繁增删、key 非字符串或要保证遍历顺序，用 Map。

坑 1：索引签名要求所有属性兼容。`{ [k: string]: string; age: number }` 会报错，因为 age 不兼容 string，解决方式是 `[k: string]: string | number`，或拆分成两个类型。坑 2：`noUncheckedIndexedAccess` 开启后 `obj[key]` 返回 `T | undefined`——这是好事，逼你处理「运行时键可能不存在」的事实，代价是每次取值都要写守卫，很多团队为省事关掉它，结果埋下运行时 TypeError 的隐患。坑 3：`Object.keys` 的返回类型是 `string[]` 而不是 `(keyof T)[]`，直接用返回值去索引签名对象会报错，正确做法是先把 key 收窄或用 `Object.entries` 配合类型断言。坑 4：模板字面量受限 key（TS 4.4+）可以用 `` `data-${string}` `` 这类模式约束 key 形态，但它只能约束「存在这些键」而约束不了「键的完整集合」，别指望它做严格枚举。

什么时候用 Map？key 非字符串、需要 O(1) 增删且频繁遍历、或者对象本身会被频繁整体替换——Map 的迭代顺序和类型（`Map<K, V>`）更贴合这类需求；单纯「以字符串为 key 的字典」用 `Record` 更贴合 JSON 场景。

**延伸考点：** `noUncheckedIndexedAccess` 开启后，`Record<string, number>` 的 `obj[key]++` 为什么报错，怎么改写？给「状态码 → 中文文案」建模，用 Record 和 Map 分别怎么表达，哪种更安全？

---

### Q6. 泛型与条件类型：extends、infer 到底怎么用
**问题：** 你想写一个工具类型，提取函数返回值的类型、提取数组元素的类型，或者根据入参类型推断返回类型。`extends` 和 `infer` 分别解决什么问题？`infer` 为什么只能出现在条件类型里？写一个你项目里真实用过的泛型工具类型。

**期望加分项：**
- 能说清 `extends` 的双重含义：约束泛型入参（`T extends SomeType`）与条件类型分发（`T extends X ? A : B`）
- 知道 `infer` 必须写在 `extends` 的右侧条件里，用于「从类型中提取子类型」
- 能手写常用工具类型：`ReturnType`、`Parameters`、`ElementType`（数组元素）、`Awaited`（Promise 展开）
- 知道条件类型在裸类型参数上会做分发（distributive），用 `[T] extends [U]` 可禁用分发
- 能结合业务：API 的返回类型从 `T extends Promise<infer R> ? R : never` 这类模式中提取
**减分项：**
- 分不清泛型约束和条件类型
- 以为 infer 能随便放在类型表达式里（语法上就不允许）
- 写不出 `ReturnType` 的手写实现，只会 `import { ReturnType } from 'ts-toolbelt'`
- 不知道条件类型的分发行为，导致工具类型结果不符合预期

**解答：**
`extends` 在类型层面有两层含义，先分清楚：第一层是**泛型约束**，`function f<T extends { id: number }>(x: T)` 表示入参必须满足「有 id 字段」，这是对入参的限制；第二层是**条件类型**，`T extends string ? 'is-string' : 'not-string'` 表示「如果 T 是 string 的子类型，取前者」。`infer` 只能出现在条件类型的 `extends` 右侧，用来声明一个「待推断的类型变量」，让 TS 从传入类型中推导：

```ts
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never
type ElementType<T> = T extends (infer E)[] ? E : never
type Awaited<T> = T extends Promise<infer R> ? Awaited<R> : T
```

使用场景：API 层把「请求函数」和「返回类型」绑定——写一个 `createApi` 泛型组件，`Promise<infer R>` 自动解出响应类型，前端不用手动维护两套类型。三个容易踩的坑：一是**裸类型参数分发**，`type IsArray<T> = T extends any[] ? true : false`，当 `T = string | number[]` 时会逐个分支计算再合并成 `boolean`，而不是整体判断，用 `[T] extends [any[]]` 包裹就能禁用分发；二是 infer 提取后可能得到联合或 never，如空数组的 `ElementType` 是 never，使用时要有兜底；三是 infer 不能出现在条件分支结果里，只能出现在条件里，这是语法硬约束。实践建议：工具类型写完一定用几个边界用例验证（联合类型、空数组、Promise 嵌套），因为条件类型的行为经常「看起来对、跑起来错」。

**延伸考点：** 写一个 `DeepNonNullable<T>`，把对象里所有可空字段变成必填（递归），注意联合类型分发的坑怎么处理？`infer` 一次能推断多个变量吗，写一个提取 `{ name: string; age: number }` 中 name 类型的例子？

---

### Q7. 工具类型手写：Partial、Pick、Omit、Record 的实现与局限
**问题：** 面试官让你手写 `Partial`、`Pick`、`Omit`、`Record` 的实现，并追问：它们各自在什么业务场景下用？`Omit` 和 `Pick` 组合能互相实现吗？嵌套对象用 `Partial` 够吗？

**期望加分项：**
- 能手写四个工具类型的关键实现：`keyof`、`in`、`extends`、`Exclude` 的组合运用
- 知道 `Partial` 只作用一层、深层要手写递归 `DeepPartial`
- 知道 `Omit = Pick<T, Exclude<keyof T, K>>` 的推导关系
- 能结合实际：表单编辑页用 `Partial<Form>`，列表接口返回带 id 的完整类型、表单用 `Omit<Row, 'id'>`
- 能指出工具类型对 readonly/optional 属性的处理差异
**减分项：**
- 只会 `import`，写不出实现
- 不知道 `Exclude<keyof T, K>` 这一步在 Omit 里的作用
- 用 `Partial` 处理深层嵌套对象，运行时字段缺失
- 不知道 `keyof` 对联合类型和接口的差异
**解答：**
四个工具类型是映射类型（mapped types）的入门组合：

```ts
type Partial<T> = { [K in keyof T]?: T[K] }
type Pick<T, K extends keyof T> = { [P in K]: T[P] }
type Record<K extends keyof any, T> = { [P in K]: T }
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>
```

核心：`keyof T` 取出所有键，`in` 逐个映射生成新键，`?` 加可选、`-?` 去可选、`readonly` 同理。`Omit` 的推导链是「先剔除 K 得到剩余键的联合，再 Pick」。业务场景对应：接口返回 `User`，表单编辑页需要 `Partial<User>` 表示「可能只改一部分字段」；新建表单需要 `Omit<User, 'id' | 'createdAt'>`，让后端生成 id；字典表用 `Record<StatusKey, string>`。

坑：1) `Partial` 只处理第一层，表单是三层嵌套对象时，`Partial<Form>` 内层字段照样必填，运行时提交会缺字段——要手写 `DeepPartial` 递归；2) `keyof any` 就是 `string | number | symbol`，Record 的键类型必须是这三者之一；3) 带 `readonly` 的属性用 `-readonly` 才能去掉，工具类型默认保留 readonly 修饰；4) `Exclude<keyof T, K>` 里 K 不存在的键会被自然忽略，所以 Omit 不会报错，这既是便利也是「类型没拦住拼写错误」的隐患，可以加 `K extends keyof T` 约束补上。

**延伸考点：** 写一个 `RequiredByKey<T, K>`，只把指定键设为必填；`DeepPartial` 对数组和 Date 这类对象递归时要怎么处理才不会破坏类型？

---

### Q8. 函数重载与实现签名：类型安全的重载设计
**问题：** 一个 `parse` 函数，传字符串返回 `Result`，传 `null` 返回 `null`，传 undefined 返回 undefined——你用重载怎么声明？为什么 TS 的函数重载写法是「多个签名 + 一个实现」？写重载时最容易犯什么错？

**期望加分项：**
- 知道重载是「声明多个调用签名 + 一个宽泛实现签名」，实现签名不对外暴露
- 能写出参数与返回值一一对应的重载，而不是全返回联合类型
- 知道实现签名要能兼容所有重载，实现签名类型错误是常见失败点
- 能识别重载的局限：参数顺序敏感、可选参数与联合类型的边界
- 能对比「重载 vs 联合参数 + 收窄」两种方案的取舍
**减分项：**
- 把实现签名暴露出去，导致调用方拿到错误的类型
- 重载顺序写错，匹配到了错误的签名
- 不知道「最后一个签名是实现签名」的语法
- 用重载解决「本来用泛型更合适」的场景
**解答：**
重载的写法是「上面多个调用签名（对调用方可见）+ 最后一个是实现签名（对调用方不可见、只供内部实现）」：

```ts
function parse(input: string): Result
function parse(input: null): null
function parse(input: undefined): undefined
function parse(input: string | null | undefined): Result | null | undefined {
  if (input == null) return input
  return JSON.parse(input)
}
```

这样调用 `parse('{"a":1}')` 得到 `Result`，`parse(null)` 得到 `null`，类型与返回值严格对应；如果只写一个签名返回 `Result | null | undefined`，调用方就得自己收窄。四个常见错误：1) 实现签名的参数/返回类型必须兼容所有重载，实现签名写成 `input: string` 会编译不过；2) 重载顺序敏感，TS 从上到下匹配第一个「参数兼容」的签名，把宽泛签名（如 `string | null`）写在具体签名前面会先被匹配，具体签名永远不生效；3) 实现签名写错位置，把 `implements` 或者把实现放在第一个；4) 重载之外的调用（如 `parse(123)`）在编译期就被拦下，运行时要保证真不出现。

什么时候别用重载：参数和返回类型有「同构」关系时，用泛型更优雅——比如 `identity<T>(x: T): T`；重载适合「不同类型入参 → 不同类型返回」的固定组合，如事件处理器、解析器。注意：重载与联合参数 + 收窄并不冲突，函数体内部依然用类型守卫收窄实现逻辑，重载只负责对外暴露「精确的调用类型」。

**延伸考点：** 重载 + 泛型能结合吗？写一个 `create<T>(kind: 'user'): User` 与 `create<T>(kind: 'order'): Order` 的重载，用泛型怎么表达？实现签名为什么必须兼容所有重载，违反会怎样？

---

### Q9. 类、抽象类与装饰器：OOP 在 TS 里的正确姿势
**问题：** 老项目用 class 写了一大堆业务模型，还用了装饰器做依赖注入。你接手后要评估：class + 装饰器这套是不是 TS 的推荐路线？抽象类、implements、mixin 分别什么时候用？装饰器在 TS 里的现状是什么？

**期望加分项：**
- 能说清 class 的三种抽象手段：继承（extends）、抽象类（abstract）、接口实现（implements）各自的适用边界
- 知道 TS 的 `private`/`protected`/`readonly` 是编译期检查，不是运行时私有（`#` 才是真正的 JS 私有）
- 知道装饰器在 TS 中仍是实验特性（experimentalDecorators），标准装饰器（stage 3）与旧装饰器的语义差异
- 能给出现代推荐：能用 interface + 类型组合表达的数据模型，不必非用 class
- 知道 mixin 模式处理多继承缺失，避免「继承链过深」
**减分项：**
- 迷信 class，把所有模型都写成 class
- 以为 TS 的 private 是运行时保护
- 不知道装饰器现状，把旧装饰器当标准
- 抽象类、接口、mixin 的使用场景说不清
**解答：**
先纠正一个常见误解：TS 的 `private`、`protected`、`readonly` 都是**编译期检查**，编译后是普通属性，运行时照样能访问；真要运行时私有，用 ES 的 `#field`。数据建模建议：纯数据（DTO、接口返回体）用 interface/type 足够，加 class 只会引入 `new`、继承等运行时负担；只有需要「行为 + 状态封装」或「多态调度」时才用 class。抽象类（abstract class）适合「有公共实现 + 有抽象方法待子类完成」的模板方法场景；接口（implements）适合「约定形状、无实现」的契约场景；多个行为要复用且不想加深继承链时用 mixin（组合函数注入方法）。装饰器的现状：TS 的装饰器是实验特性（`experimentalDecorators`），Angular/NestJS 生态大量使用；ES 标准装饰器已进入 stage 3，语法与行为有差异（参数位置、接收的元数据不同），两者不可混用，用第三方 DI 库（如 `inversify`、Nest 自带的）前先确认它支持哪种语义。实践建议：不要为「图省事」把每个 service 写成 class + 装饰器，函数式（纯函数 + 依赖注入参数）在大多数前端/Node 项目里更好测、更好 tree-shaking；class 只在真正需要实例状态或继承多态时使用。判卷时的加分点是候选人能说出「class 不是万能的，数据用 interface、行为用函数/class 按需选」这种取舍判断。

**延伸考点：** 标准装饰器（stage 3）与 `experimentalDecorators` 下写 `@Component` 有什么区别？实现一个 `WithLogger` mixin，说明为什么 mixin 能替代部分继承？

---

### Q10. 类型推导失效的典型场景：as const、断言滥用与泛型陷阱
**问题：** 写 `const theme = { color: 'red', size: 'md' }` 然后 `theme.color` 的类型是 `string` 而不是 `'red'`，导致很多本来能安全收窄的地方变宽；又有同事到处用 `as` 断言。怎么系统性改进「类型过宽 / 断言滥用」问题？

**期望加分项：**
- 知道对象字面量默认推导为宽类型，`as const` 能推导为字面量类型并让属性 readonly
- 知道 `as` 断言会绕过类型检查，能举出断言掩盖真实 bug 的例子
- 知道 `as const` 的副作用：对象全属性 readonly，与可变操作冲突
- 能提出「收窄优先、断言兜底」的原则，并给出 lint 约束（`no-unnecessary-type-assertion`）
- 能讲泛型推导失效场景：函数参数被推导成宽类型导致工具类型无法精确定位
**减分项：**
- 遇到类型报错就用 `as` 硬扛
- 不知道字面量类型与宽类型推导的区别
- 不知道 `as const` 会让属性 readonly
- 说不出「哪些类型错误该改类型设计、哪些该用断言」
**解答：**
先明确推导规则：对象字面量初始化时，TS 会推导出「宽类型」——`color` 是 `string` 而不是 `'red'`，因为变量可以被重新赋值；要让字面量类型成立，加 `as const`：

```ts
const theme = { color: 'red', size: 'md' } as const
type Color = typeof theme.color // 'red'
```

副作用：`as const` 让所有属性变 `readonly`，如果后续要 `theme.color = 'blue'` 会报错，需要用 `as const` 只作用于「需要精确类型的局部」。断言滥用的危害：`as` 是「告诉编译器你知道自己在做什么」的逃生门，它会掩盖两类 bug——类型声明与实际运行时形状不一致、以及可空值被断言成非空后运行时崩溃。改进原则：1) 优先用「更精确的初始化」让推导自己收敛（`as const`、字面量联合、模板字面量类型）；2) 确实需要断言时用更安全的 `satisfies`（TS 4.9+），它做「校验 + 保留推导」而不是「绕过」；3) 开 eslint 的 `no-unnecessary-type-assertion` 拦截「断言类型等于推导类型」的无意义断言；4) 泛型场景注意：`function get<T>(key: keyof T)` 里的 `keyof T` 会被推导成宽联合，配合 `as const` 传参会让推导精确定位。

实践中常见的「推导失效」排查套路：报错「类型 'string' 不能赋给类型 'A'」时，先看源头有没有 `as` 或第三方 any，再看对象初始化有没有 `as const`，最后才怀疑工具类型写错——80% 的宽类型问题出在前两步。

**延伸考点：** `satisfies` 和 `as const` 组合怎么写？用 `const obj = {...} as const satisfies SomeShape` 报错时说明什么？为什么说 `as` 断言在泛型推导里「最伤人」？

---

### Q11. React + TS：props 类型设计与泛型组件的类型推断
**问题：** 团队 React 项目里 props 类型经常写 `any`，或者一个通用 Select 组件的外部调用者拿不到「options 的 label 字段名」这种约束。你如何设计组件 props 类型，让使用方类型安全、IDE 提示友好？泛型组件怎么推断？

**期望加分项：**
- 能用 `interface` 描述 props，用 `React.FC`/函数组件签名表达 children 等固定字段
- 能用泛型组件 `function Select<T extends string>(props: SelectProps<T>)` 让调用方传入的类型约束 options 字段
- 知道 `useCallback`/`useMemo` 泛型推导依赖，避免推导出宽类型
- 能处理「组件外部扩展 props」的场景：`ComponentProps<typeof X>`、`React.ComponentPropsWithRef`
- 知道避免用 `any` 表达事件对象，用 `React.MouseEvent` 等精确类型
**减分项：**
- props 直接写 any，等于放弃类型安全
- 泛型组件在 `.tsx` 里报「无法推断」时不知道 `<T,>` 的逗号写法
- 不知道 `ComponentProps`、`ElementType` 等 React 内置工具类型
- 事件处理函数类型写错导致频繁 `as`
**解答：**
设计 props 的第一原则：**props 类型是组件对外契约，必须精确到「调用方能自动补全」**。基础做法：

```tsx
interface SelectProps<T extends string> {
  options: Array<{ label: string; value: T }>
  value: T
  onChange: (v: T) => void
}
function Select<T extends string>(props: SelectProps<T>) { ... }
```

`.tsx` 里泛型箭头函数要写 `<T,>`（带逗号），否则 JSX 解析冲突；`.ts` 或 function 声明则不需要。调用方传 `options` 后，`value` 和 `onChange` 的类型会自动收窄成该集合的字面量联合，IDE 补全到位，这是泛型组件最大的价值。避免的坑：1) 事件对象不要写 `any`，用 `React.ChangeEvent<HTMLInputElement>`，写错会导致 `e.target.value` 类型飘；2) 组件扩展场景用 `ComponentProps<typeof Button>` 拿既有组件的 props 做包装，而不是手抄一份（手抄会漂移）；3) `useCallback` 依赖泛型回调时，类型要写成 `(v: T) => void` 而不是匿名推断，否则每次渲染推断成新函数类型；4) children 用 `ReactNode` 而非 `JSX.Element`（后者不接受 Fragment 和 null）。还有一个工程细节：公共组件库的 props 命名要与产品字段对齐（label 不叫 text、value 不叫 code），因为类型既是编译约束也是文档。

**延伸考点：** 泛型组件配合 `forwardRef` 时 ref 的类型怎么推导？写一个 `useToggle` hook，返回值的类型为什么用 `as const` 数组而不是普通数组？

---

### Q12. Vue + TS：props 类型、defineEmits 与模板类型检查
**问题：** Vue 3 + TS 项目里，`defineProps` 直接写 `const props = defineProps<{ items: any[] }>()`，模板里 `items[0].name` 报错，或者 `defineEmits` 的事件参数在模板里类型丢失。你怎么把组件的「输入输出」类型做扎实？

**期望加分项：**
- 用泛型写法 `defineProps<Props>()` 而非运行时校验（默认编译成运行时 props）
- 用 `defineEmits<{ (e: 'update:value', v: string): void }>()` 让事件参数类型化
- 知道 `withDefaults` 给泛型 props 提供默认值，避免 `?.` 一堆
- 知道 `defineModel`（3.4+）让 v-model 类型化且可设默认值
- 模板中的类型检查：`vue-tsc` 检查模板表达式类型，需要 `<script lang="ts">` + `vue-tsc --noEmit`
**减分项：**
- props 用 any，模板里拿不到任何类型
- 不知道泛型 props 和运行时 props 的关系（默认会转成运行时声明）
- 事件类型不声明，子组件 emit 的载荷全变 any
- 不知道用 `vue-tsc` 做模板类型检查，只看 script 类型
**解答：**
Vue 3 的 defineProps 有两条路线：运行时声明（`defineProps(['items'])`）和类型声明（`defineProps<{ items: Item[] }>()`）。推荐类型声明，编译器会把它转成等价运行时 props，同时拿到精确类型：

```ts
interface Props { items: Item[]; title?: string }
const props = withDefaults(defineProps<Props>(), { title: '默认标题' })
const emit = defineEmits<{
  (e: 'update:value', v: string): void
  (e: 'select', item: Item): void
}>()
```

`withDefaults` 解决「泛型 props 没有默认值」的痛点，避免模板里 `props.title?.`。`defineModel<T>()`（3.4+）给 v-model 提供类型化声明，`defineModel<string>('value')` 相当于 props + emit 的一体化写法。事件类型声明要写成「调用签名」形式，因为 Vue 编译器要靠它生成运行时 emits 校验。另一个关键：模板里的类型检查默认不生效——`<script lang="ts">` 只检查 script 部分，模板表达式要运行 `vue-tsc --noEmit`（或 `vue-tsc --noEmit && vite build`）才会报错。团队落地时把 `vue-tsc` 挂到 CI 和 commit hook，否则模板里 `item.name` 写错字面量也不会被发现。坑：1) 泛型 props 的复杂类型（联合、交叉）要确保能被编译器转成运行时声明，转不了的类型（如 `Date`）要用 `PropType` 辅助；2) `defineEmits` 的事件名要在 emits 声明里，否则被当作原生事件透传；3) `reactive` + 泛型的组合里，`ref` 的泛型默认推导可能变宽，显式写 `ref<T>(...)` 更稳。

**延伸考点：** `withDefaults` 与运行时 `default` 的关系，为什么泛型 props 默认值必须在 `withDefaults` 里写？`defineEmits` 声明成「调用签名」与「元组数组」两种写法有什么区别？

---

### Q13. 类型与运行时双校验：zod 类方案为什么必要
**问题：** 后端接口返回的数据，你在前端声明了类型 `interface Order { id: string; amount: number }`，结果某天后端返回 `amount: '12'`（字符串），前端 `order.amount * 2` 得到字符串拼接而不是数字。TS 类型在这里完全没拦住。你怎么办？

**期望加分项：**
- 能点出核心：TS 类型是编译期静态声明，不校验运行时数据，边界（网络/存储/用户输入）必须运行时校验
- 会引入 zod / valibot / yup 等 schema 库，写 `z.object({...})` 并 `parse` / `safeParse`
- 能给出「schema 同时推导类型」的模式：`type Order = z.infer<typeof OrderSchema>`
- 能处理嵌套与可选：`.nullable()`、`.default()`、`.transform()` 的典型用法
- 知道校验失败后的降级策略：默认值、告警、阻断重试，而不是静默吞掉
**减分项：**
- 认为 TS 类型就是运行时保证
- 不知道 zod / valibot 这类库的存在
- 校验失败直接 throw，不区分「可修复」和「致命」
- 不把 schema 作为「类型单一来源」，导致类型与校验规则漂移
**解答：**
关键认知：**TS 类型只在编译期存在，编译后是空**，它对「后端返回的数据形状」没有任何约束力——接口返回的 `amount` 是字符串，`interface` 里写 `number` 只是自欺欺人。所以边界数据必须运行时校验，主流做法是 schema 库（zod 最流行，valibot 体积更小、适合底层库）：

```ts
import { z } from 'zod'
const OrderSchema = z.object({
  id: z.string(),
  amount: z.number().positive(),
  status: z.enum(['pending', 'paid', 'cancelled']).default('pending')
})
type Order = z.infer<typeof OrderSchema> // 类型从 schema 推导，单一来源
const parsed = OrderSchema.safeParse(raw) // 运行时校验
```

`z.infer` 让「类型」和「校验规则」只写一份，避免 interface 与 zod 双份维护导致漂移。工程落地要点：1) 校验放在**数据入口**（请求层/API client），而不是每个组件里，一次 parse 全局安全；2) 用 `safeParse` 而不是 `parse`，失败时进入降级分支；3) 降级策略要分级——可选字段缺失用 `.default()` 兜底、关键字段类型错误要告警并阻断渲染（宁可白屏也不渲染错数据）、可自动修复的（数字字符串）用 `.transform()` 处理；4) 性能：高频调用时 zod 校验有开销，可以只在网关层校验一次或对 Schema 做缓存。顺带一提，这条思路同样适用于环境变量、localStorage 读取、第三方 SDK 回调——一切「类型不可信」的入口都该过 schema。

**延伸考点：** zod 的 `.transform()` 和 `.default()` 执行顺序对类型的影响？前端如何与后端的 OpenAPI 生成类型配合，减少手写 schema 的重复劳动？

---

### Q14. OpenAPI 生成类型：前后端契约的自动化
**问题：** 团队前后端靠手写的 `interface` 对齐接口，后端改字段后前端编译不报错、线上才炸。你们打算引入「后端 OpenAPI → 前端 TS 类型」的自动化链路，怎么设计？生成代码 vs 手写代码怎么共存？

**期望加分项：**
- 知道 OpenAPI/Swagger 规范与典型生成工具：openapi-typescript、openapi-generator、orval
- 能说清生成链路：CI 里拉取最新 spec → 生成 TS 类型/请求函数 → 产物提交或构建时生成
- 知道生成类型的取舍：全量类型 vs 按需生成、生成代码的版本管理与 diff 审查
- 能处理「后端字段可选 vs 前端必填」的语义差：用类型层面覆盖而非手改生成文件
- 知道生成代码不能手改（会漂移），要用「本地类型包装」扩展
**减分项：**
- 不知道任何生成工具
- 认为生成了就一劳永逸，不管版本漂移
- 手改生成文件导致下次生成覆盖丢失
- 不处理「可选字段」与「必填字段」的语义冲突
**解答：**
链路设计：后端发布 OpenAPI 规范（SpringDoc / FastAPI / Swagger 均可导出），CI 在每次构建时拉取最新 spec，跑生成工具产出 `types/api.d.ts`（openapi-typescript 只产类型，orval 还能生成带 axios 的请求函数），产物进版本库并做 PR 审查——后端字段变更在前端提交里可见，从源头消除「改字段不报错」。关键取舍：1) **生成文件禁止手改**，手改会被下一次生成覆盖，扩展用「本地类型包装」——`type MyOrder = GeneratedOrder & { computedX: number }`；2) 后端字段 `required: false` 生成出来是 `string | undefined`，前端业务里往往是必填，用包装类型覆盖语义，不要在生成文件里 `!` 硬断言；3) 版本管理：spec 有版本号，前端记录「对接的 spec 版本」，后端 breaking change 时前端明确感知；4) 生成代码 vs 运行时校验：类型生成解决「编译期对齐」，zod 解决「运行时校验」，两者互补——进阶做法是从 OpenAPI 直接生成 zod schema（如 openapi-zod-client），一套 spec 同时产出类型和校验。落地坑：接口路径命名要规范化（restful），否则生成代码可读性差；后端 spec 质量差（字段全是 optional）时，前端生成的类型全是可选，价值大打折扣，要推动后端把必填字段标清楚——这本身也是推动前后端契约质量的抓手。

**延伸考点：** openapi-typescript 与 orval 分别适合什么团队（纯类型 vs 带请求层）？后端字段重命名后，生成代码会报哪些错误，前端怎么主动发现？

---

### Q15. tsconfig 工程配置：strict、moduleResolution、paths 的排障与优化
**问题：** 新项目 `npm run build` 报「无法找到模块」或「模块解析失败」，把同事叫来，第一句问的往往是「你的 tsconfig 里 moduleResolution 是啥」。你怎么理解这些配置项，什么情况下改它们、什么情况下不该改？

**期望加分项：**
- 知道 `strict: true` 包含哪些开关（strictNullChecks、noImplicitAny 等）及其含义
- 知道 `moduleResolution`（node / node16 / bundler）与 `module` 的配套关系，现代 Vite 项目选 `bundler`
- 知道 `paths` + `baseUrl`（或仅 paths）做别名，但别名要与打包器（vite alias）保持一致
- 知道 `isolatedModules`、`verbatimModuleSyntax` 对「单文件转译」的限制
- 知道 `target`/`lib`/`skipLibCheck` 各自解决什么
- 能讲出「改动配置要理解后果」而非盲目试错
**减分项：**
- 遇到报错就改 `skipLibCheck: false` 或关 strict
- 不知道 module 与 moduleResolution 的搭配约束
- paths 别名配了 tsconfig 但 vite 不认，两边各配一套
- 不知道 isolatedModules 对 `export type` 的要求
**解答：**
先分层理解配置的目的：`strict` 是「严格性总开关」，它背后是 strictNullChecks（null/undefined 进类型系统）、noImplicitAny（禁止隐式 any）、noUnusedLocals 等一组开关，新项目必须 `strict: true`，关掉任何一个都是在给线上埋雷。`module` 与 `moduleResolution` 是搭配对：`module: esnext` + `moduleResolution: bundler`（TS 5.0+，为 Vite/webpack 打包器设计，支持 extensionless import、imports 字段）；老 Node 项目用 `node16/nodenext` 会强制带扩展名且区分 ESM/CJS。`paths` 做别名：`"paths": { "@/*": ["./src/*"] }`，但**必须与打包器的 alias 一致**——Vite 里配 `resolve.alias`，两边都配才不会「编辑器认识、构建不认」；`baseUrl` 已废弃，直接写相对 `paths`。`isolatedModules` 是给 esbuild/babel 这类「单文件转译」工具的安全检查：它要求 `import type` 与 `export type` 明确写出，否则转译器无法判断「这个 import 是纯类型」而保留运行时引用，导致类型被编译进产物。`verbatimModuleSyntax` 更严格。`skipLibCheck: true` 是跳过 `.d.ts` 的完整检查，一般建议开着（库类型互相打架时能救命），但它也掩盖了库的类型错误，别为了「让编译过」而盲目关 strict。排障思路：先看报错类型——「找不到模块」先确认路径与 paths 是否配齐、「类型不匹配」先看 strict 开关、「编译慢」再看增量构建（Q16）。

**延伸考点：** `moduleResolution: bundler` 与 `node16` 下 `import './x'` 的扩展名要求有什么不同？verbatimModuleSyntax 开启后 `import { type A, B }` 的写法说明什么？

---

### Q16. 类型检查性能优化：增量编译、project references 与构建提速
**问题：** 项目从 50 万行涨到 200 万行，`vue-tsc` / `tsc` 全量检查从 10 秒涨到 4 分钟，开发时保存一次要等半天。你从哪些层面优化类型检查与构建速度？

**期望加分项：**
- 知道 `incremental` + `tsBuildInfoFile` 的增量编译原理，改一个文件只重检相关部分
- 知道 project references 让类型检查分层并行
- 知道 `composite: true` 是 project references 的前置条件
- 能用 `type-coverage` 定位 any 泄漏，降低「any 传导导致的无效检查」
- 知道把 `tsc` 拆成「类型检查」与「构建转译」（esbuild/vite）两条路线的取舍
- 避免滥用 `extends` 深层继承链（类型计算指数膨胀）
**减分项：**
- 只会加大内存或换机器
- 不知道 incremental 的存在
- 全项目一个 tsconfig，任何改动全量重检
- 类型工具写得过于复杂导致 `tsc` 卡死
**解答：**
优化分三档。第一档：**增量编译**——tsconfig 开 `"incremental": true`，`tsc` 会把编译图写进 `.tsbuildinfo`，改动后只重检受影响的文件（文件级依赖图），小项目秒过、大项目从分钟级降到秒级；配 `"tsBuildInfoFile"` 指定缓存位置。注意 `incremental` 与 `noEmit` 组合在部分版本有坑，`vue-tsc` 有自己的增量模式。第二档：**分层**——用 project references 把代码按「核心类型层 / UI 层 / 工具层」拆成多个 `composite: true` 的 tsconfig，构建时各层并行检查、依赖层只查接口不查实现；代价是配置复杂度上升，适合确定的多包/分层结构。第三档：**减负**——1) 把「构建」与「类型检查」分离：Vite 用 esbuild 转译（不查类型），类型检查交给独立 `vue-tsc --noEmit`（CI 或 pre-commit），开发时热更新不再被 tsc 拖累；2) 用 `type-coverage` 看 any 覆盖率，any 传导会让类型检查「假快真慢」；3) 避免超深的条件类型/映射类型递归——某些类型运算复杂度指数级，`tsc` 会卡死或内存溢出，能用 `declare` 简化的就别展开。实践顺序建议：先开 incremental（改动最小收益最大）→ 再分离构建与检查 → 最后才考虑 project references。判分加分点：能说出「类型检查本质是图上的可达性分析，增量只在『依赖没变』时有效，任何改接口的操作都要重检下游」这种机制理解。

**延伸考点：** `incremental` 模式下「改了公共类型的字段名」为什么下游文件也会重检？project references 与 monorepo 的 workspaces 怎么配合？

---

### Q17. monorepo 类型共享：跨包类型设计、依赖版本与构建顺序
**问题：** 团队从单仓库拆成 pnpm monorepo，拆出 `@app/ui`、`@app/shared-types`、`@app/api` 几个包。结果类型互相 import 时经常「找不到声明」或「类型版本对不上」，构建顺序还经常出问题。你怎么设计 monorepo 的类型共享与构建？

**期望加分项：**
- 知道包类型入口：`types`/`exports` 字段指定 `.d.ts`，`main`/`module` 指定 JS 入口
- 知道 workspace 协议 `workspace:*` 依赖，避免发布版本对不上
- 知道构建顺序问题：依赖被 import 的包要先构建出产物，或开发时用源码直引（`"types": "src/index.ts"`）
- 知道用 project references 让 tsc 理解包间依赖
- 能处理类型与运行时分离：纯类型包（`.d.ts` only）可以零构建
- 会设计「类型单一来源」：跨包共享的领域类型放 `shared-types`，业务包引用
**减分项：**
- 跨包用相对路径 import，破坏包边界
- 不知道 `exports` 字段对类型解析的影响
- 依赖没有 `workspace:*`，发布后版本漂移
- 构建顺序靠人记，不用工具保证
**解答：**
monorepo 类型共享的核心是「包边界 + 构建产物」两个问题。**包边界**：所有跨包引用走包名（`import { Order } from '@app/shared-types'`），不跨包写相对路径；每个包的 package.json 用 `exports` 字段精确声明入口（`"exports": { ".": { "types": "./dist/index.d.ts", "import": "./dist/index.mjs" } }`），`types` 放前面让类型解析器先命中。**依赖版本**：workspace 内互引必须用 `workspace:*`（pnpm 默认），避免某包发布后其他包锁到旧版本。**构建顺序**：被 import 的包要先生成产物（`dist/*.d.ts`），否则 import 方找不到声明；工程上两类解法——1) 纯类型包（只有 `.d.ts`，无运行时逻辑）配 `"types": "src/index.ts"`，源码直引零构建，这是最省事的方案；2) 有运行时的包用 project references，让 tsc 知道「A 依赖 B」自动按序构建，或用 turbo/nx 这类带拓扑排序的构建器。**类型单一来源**：跨包共享的领域模型（Order、User、枚举）集中在 `@app/shared-types`，业务包只 import 不定义，避免「两个包各有一份 Order 导致结构不兼容」。落地坑：1) `exports` 配置错误时编辑器报「找不到模块」但构建能过，反之亦然，排查时先看 node_modules 里符号链接的 package.json；2) 包间循环依赖（类型层面）会导致构建死锁，用 `import type` 声明纯类型引用可打破；3) tsc 的 `moduleResolution: bundler` 在 monorepo 里要配合 workspace 根 tsconfig 的 `paths`，别名别在子包重复定义。

**延伸考点：** 「类型版本对不上」在 monorepo 里的典型症状和排查手段？纯类型包（`.d.ts` only）为什么能零构建，它和 `declare module` 的关系是什么？

---

### Q18. JS 项目迁移 TypeScript：渐进策略与风险控制
**问题：** 一个运行两年的纯 JS 项目，30 万行代码，业务不能停，你怎么规划迁到 TypeScript？先迁什么、用什么工具、怎么保证「迁移过程不炸」？

**期望加分项：**
- 能给出渐进策略：`allowJs: true` + `checkJs: false` → 逐个文件加 `// @ts-check` → 新文件强制 TS
- 知道 `any` 作为迁移过渡的「出口阀」：先 `strict: false` 再逐步收紧
- 知道用 `ts-migrate` / `jscodeshift` 自动化转换基础语法
- 能按「数据流入口」优先迁移：工具函数、API 层、类型边界，而非先迁 UI
- 知道迁移期间「运行时行为不变」的验证手段：测试覆盖率、diff 空转
- 能给出红线：新代码不许 any、老代码设期限清账
**减分项：**
- 一把梭全量迁移，业务停摆
- 迁移完才想起测试，行为变更无法对比
- 全程 any，迁移完等于没迁
- 没定「新代码标准」，边迁边造新债
**解答：**
迁移的核心是「让 TS 渐进接管而不破坏运行时行为」，路线分四步。第一步：**开启 JS 支持**——tsconfig 里 `"allowJs": true` + `"checkJs": false`，TS 只做转译不检查，构建链路（webpack/babel）先切到 tsc/esbuild 输出，确认产物一致；这一步无风险，是地基。第二步：**入口文件加 `// @ts-check`**——对工具函数、纯函数模块先开检查，`strict: false` 起步，错误以「宽松」模式消化；用 `ts-migrate` 把明显的 `require`/`module.exports` 转成 `import/export`、给回调补 `any` 占位，机械工作自动化。第三步：**按「数据流入口」优先级迁移**——先 API client（fetch/axios 封装，类型边界最重要）、工具函数、全局类型声明（`global.d.ts`），再迁移页面组件；顺序的理由是「上游类型化之后，下游接入成本自然下降」。第四步：**收紧红线**——新代码强制全类型（lint 规则 `no-explicit-any` 对新文件生效）、老代码登记 any 清账清单并设截止日期、`strict` 逐项打开（先 strictNullChecks，它收益最大、破坏最小）。三条纪律：1) 迁移期间「运行时行为不变」是底线，每个文件迁移后跑测试对比行为；2) 迁移不改变逻辑，禁止顺手重构（重构成「重构日」单独做，混在一起无法定位回归）；3) any 是迁移的「过渡阀」不是终点，定期用 type-coverage 量化 any 占比，作为迁移进度的可见指标。判分加分点：能说出「迁移的本质是给每个数据流入口补上类型边界，而不是给每一行代码标类型」。

**延伸考点：** `// @ts-check` 在 JSDoc 注释配合下能到什么程度？迁移时哪些模块「不要迁」反而更合理（如一次性脚本）？

---

### Q19. 类型测试与质量保障：dtslint、expectType 与 CI 集成
**问题：** 你写了一个公共工具类型库，或者一个泛型组件库，如何保证「类型层面」的行为正确？运行时测试（jest）测不到类型错误，怎么办？类型测试怎么写、怎么进 CI？

**期望加分项：**
- 知道类型测试的工具：`tsd`、`dtslint`、`vitest` 的 `expectTypeOf`、`@type-challenges`
- 能写出典型断言：`expectTypeOf(foo).toEqualTypeOf<Bar>()`、`@ts-expect-error` 负向测试
- 知道「负向测试」的价值：断言「错误类型会编译报错」
- 能设计测试矩阵：字面量、联合、泛型实例化、边界（空数组、undefined）
- 知道类型测试进 CI：跑 `tsc --noEmit` + 类型断言测试
**减分项：**
- 认为类型不用测，「能编译过就行」
- 只测正向不测负向
- 不知道 `@ts-expect-error` 与 `@ts-ignore` 的区别
- 工具类型没有测试矩阵，边界用例全凭运气
**解答：**
类型的「正确性」分为两层：编译不报错（tsc 层面）和「类型推导符合预期」（语义层面），后者只有用类型断言测。主流工具：`tsd`（独立 CLI，`expectType` 风格）、`vitest` 的 `expectTypeOf`（项目里已有 vitest 时最省事）、`@type-challenges`（练习与基准）。典型写法：

```ts
import { expectTypeOf } from 'vitest'
expectTypeOf(pick(user, 'name')).toEqualTypeOf<{ name: string }>()
// @ts-expect-error 传入非法 key 应报错
pick(user, 'notExist')
```

正向断言验证「推导结果正确」，负向断言（`@ts-expect-error`）验证「非法输入被类型系统拒绝」——两者都过才算类型正确。注意 `@ts-expect-error` 与 `@ts-ignore` 的区别：expect-error 是「断言下一行有错误」，若下一行没错误，expect-error 自己会报「无用的错误断言」（TS 5.0 起），能防「类型修好了但断言残留」；`@ts-ignore` 是无条件静默，应禁止使用。设计测试矩阵：对每个工具类型测 5 类用例——字面量类型、联合类型、泛型实例化、边界（空数组、`undefined`、空对象）、与运行时行为对照（拿到工具类型的值跑一遍业务逻辑）。CI 集成：类型断言测试与运行时测试同跑（vitest 天然支持），纯类型库再加 `tsc --noEmit` 兜底。实践坑：类型断言断言的是「编辑器看到的类型」，IDE 缓存有时会骗人，CI 里以命令行 tsc 为准；跨 TS 版本测试（dtslint 支持多版本矩阵）适合对外发布的库，内部项目测当前版本即可。

**延伸考点：** `expectTypeOf` 的 `toEqualTypeOf` 与 `toMatchTypeOf` 有什么区别，什么时候用后者？为什么说「负向测试」比「正向测试」更能体现类型质量？

---

### Q20. any 泄漏治理：覆盖率、溯源与团队红线
**问题：** 项目里 any 越来越多，review 时有人辩解「这里 any 一下后面就正规了」，结果 30 万行代码里 any 上千处。你怎么系统性治理，而不是靠口号「不许写 any」？

**期望加分项：**
- 能用 `type-coverage` 量化 any 覆盖率和泄露位置
- 能按「任何处的位置」分级：入口/出口（API、事件）、内部计算、第三方回包
- 能给出治理优先级：先堵「上游 any」（API 层、事件对象），下游自动恢复
- 知道用 eslint 渐进规则：老文件豁免、新文件强制
- 能讲 any 泄漏的连锁反应：一个 any 让整条链路的类型失效
- 能设计「红绿灯」指标：any 数量随时间曲线，进 CI 门槛
**减分项：**
- 只会「看到 any 就骂」
- 不知道 any 会通过函数调用、数组元素、对象属性传导
- 治理停留在「改 this 处」而不追溯源头
- 没有量化指标，治理进度不可见
**解答：**
any 治理的难点在「传导」——一个 API 层 any，会让下游所有依赖该接口的调用点类型失效，所以治理要「堵源头 + 量化进度」双管齐下。第一步**量化**：`type-coverage`（npx type-coverage --detail）输出「类型覆盖率」与 any 出现位置清单，用它画「any 数量随时间」的曲线，作为团队红绿灯指标，而不是靠人工 review 数。第二步**分级**：按 any 位置排优先级——P0 是数据入口（API client、事件对象、localStorage 读取），修好它们下游自动恢复类型；P1 是函数签名里的 any 参数（影响调用方）；P2 是内部临时变量（影响面最小）。第三步**渐进规则**：eslint 的 `no-explicit-any` 对全仓库生效，老文件用 `/* eslint-disable no-explicit-any -- legacy */` 逐文件豁免并登记 owner + 清账期限，新文件零豁免；`no-unsafe-*` 系列（`no-unsafe-assignment`、`no-unsafe-argument`、`no-unsafe-member-access`）专门拦截「any 的传导路径」，比 `no-explicit-any` 更能防「隐式 any 泄漏」（如 `JSON.parse` 返回 any 后赋值给变量）。第四步**治本**：对 P0 数据入口逐一补 schema 或声明，让类型恢复；第三方无类型库用 `declare module` 补最小声明，不要用 `any` 过渡。考核加分点：能说出「any 治理不是消灭 any，而是把 any 关在数据入口的笼子里——入口规范化之后，业务代码里就几乎没有 any 了」这种分层认知，并给出可量化的验收标准（覆盖率从 85% 提到 98% 且保持）。

**延伸考点：** type-coverage 的 `--detail` 输出的 any 位置怎么按「入口/内部」分类做周报？`no-unsafe-argument` 和 `no-unsafe-assignment` 分别拦的是哪类泄漏路径？
