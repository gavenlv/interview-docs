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

坑 1：索引签名要求所有属性兼容。`{ [k: string]: string; age: number }` 会报错，因为 age 不兼容 string，解决方式是 `[k: string]: string | number`，或拆分成两个类型。坑 2：`noUncheckedIndexedAccess` 开启后 `obj[key]` 返回 `T | undefined`——这是好事，逼你处理「运行时键