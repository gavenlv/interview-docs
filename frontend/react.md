# 前端 · React（面试题库）

本文件考察候选人在真实 React 工程中的实践能力，而非八股背诵。重点覆盖：列表与重渲染性能定位、状态管理与数据流设计（Context/Redux/Zustand/请求缓存）、副作用与并发特性（useEffect/startTransition）的正确使用、组件拆分与自定义 Hook 抽象、SSR/微前端等复杂架构下的取舍，以及 React 18/19 新特性的落地判断。所有问题均来自一线生产场景，期望候选人用真实项目经验、量化手段与取舍判断作答。

### Q1. 列表用 index 当 key，删除中间项后状态错位、输入内容串行，问题出在哪？

**问题：** 列表接口返回的数据没有唯一 id，你用数组下标 index 当 key。删除中间一项后，后面行的输入框内容、勾选状态全部串位。请解释 key 的作用机制，并给出正确做法。

**期望加分项：**
- 能讲清 key 是 React 判断"组件实例是否复用"的依据，而不是"序号"
- 能具体解释"删除后下标前移、React 复用旧实例导致 state 串位"的完整链路
- 能给出无唯一 id 时的兜底方案（复合字段、业务主键、渲染时生成并缓存稳定 id）
- 能主动提到 key 还影响动画定位、滚动位置、焦点保持等隐蔽症状
- 能区分 key 与 useEffect 依赖这两个概念，不被混淆

**减分项：**
- 只背"key 必须唯一"，说不出为什么
- 认为列表纯展示时 index 当 key 也没问题，答不出边界
- 分不清"key 变化触发重建"与"diff 复用"的关系

**解答：**

先给判断：key 不是给 React 当序号用的，而是调和（diff）阶段判断"同位置兄弟节点是否为同一个组件实例"的依据。React 对兄弟列表做 diff 时，同位置 key 相同就复用实例（保留 DOM 与 state），不同才卸载重建。用 index 当 key，一旦列表顺序变化（删除、头部插入、排序），所有项的下标整体前移或错位，React 认为 key=0、1、2… 对应的是同一批组件，于是复用旧实例，输入框内容、勾选状态就"串行"了。

```jsx
// 错误：index 当 key，删除/排序后会串位
{items.map((it, i) => <Row key={i} data={it} />)}
// 正确：业务唯一 id
{items.map((it) => <Row key={it.id} data={it} />)}
```

数据确实没有唯一 id 时，退而求其次：用复合字段拼接（如 `${type}-${name}-${time}`），或在首次渲染时生成稳定 id 并缓存（useMemo 按内容做一次映射），但要注意后端排序字段变化会让"看似唯一"的 key 漂移，引发整列表闪烁。实践中更隐蔽的症状是：列表动画（FLIP 定位）错乱、删除后滚动位置跳变、`<input>` 自动聚焦到错的行——这些都比"状态串位"难排查。一条经验：只要列表项"可排序、可增删、内部有状态"，就必须用稳定且唯一的 key。

**延伸考点：** 列表项是纯展示、无内部 state 时，index 当 key 还有问题吗？为什么？

### Q2. 搜索表单每次击键整块区域重渲染，受控组件是元凶吗？该怎么选型？

**问题：** 一个 10 多个输入项的搜索表单用受控组件实现，每次击键整个表单区域都重渲染，出现卡顿。换成非受控组件能解决吗？你会怎么选型？

**期望加分项：**
- 能说清受控/非受控的本质区别：值归 React state 管还是归 DOM 管
- 能点破"卡顿根源是 value 挂在高层级 + onChange 触发整棵子树重渲染"，而非受控本身
- 能给出针对性解法：拆分字段组件让 state 下沉、提交时才取值、必要时非受控 + ref
- 能讲 defaultChecked/defaultValue 这些非受控入口与使用场景
- 有在真实项目里混合使用、并说明取舍的经历

**减分项：**
- 直接下结论"非受控性能好所以全用非受控"，忽略校验、联动、重置需求
- 只会背概念，讲不出"击键 → setState → 父组件 → 整树重渲染"这条链路
- 不知道提交时用 ref/FormData 取值及其时机问题

**解答：**

先纠正常见误区：卡顿不是"受控"造成的，而是 value 放在高层组件、每次击键 onChange 触发父组件连带整棵子树重渲染造成的。受控与非受控的本质是数据源不同：受控组件的值由 React state 驱动，适合需要校验、联动、清空重置的场景；非受控的值留在 DOM 里，提交时才读取，渲染路径短但难干预。所以选型看"这个值需不需要被 React 感知"。

优化的第一选择是"缩小受控范围"而不是切换模式：把每个输入项拆成独立子组件，state 下放到字段内部，只在失焦/提交时上报。这样击键只重渲染当前字段，父组件不参与。

```jsx
function Field({ name, onSubmit }) {
  const [v, setV] = useState('');
  return <input value={v} onChange={e => setV(e.target.value)}
                onBlur={() => onSubmit(name, v)} />;
}
```

如果表单真的很大（几十上百字段），可以考虑非受控 + `FormData`/ref 提交时取值，或直接用 React Hook Form（默认非受控 + 校验方案）。实践中的坑：一是半受控模式（只给 value 不给 onChange，或 value + onChange 不 setState）会让输入框"打不出字"，是最常见的低级 bug；二是非受控 + 提交取值的时机问题——onSubmit 里读 ref 时 DOM 一定已更新，但"已输入未失焦"的值要确认事件顺序；三是混合模式下提交前务必确认数据已同步，否则会出现"界面有值、提交空值"。

**延伸考点：** React Hook Form 为什么推荐非受控？它的校验如何在不触发整表单重渲染的前提下读取字段值？

### Q3. 业务页面写了 2000 行，改一个字段要全局排查，组件该怎么拆？

**问题：** 一个业务页面 2000 行，数据获取、业务规则、UI 布局、表单控制全混在一起，任何改动都要全局排查。请给出组件拆分的判断标准与实操步骤。

**期望加分项：**
- 有明确的拆分信号（单一职责、渲染频率、复用需求、变更影响面），而非凭感觉
- 能区分容器组件（数据与逻辑）与展示组件（纯 UI），且不教条化
- 能联系性能：拆分后的子树渲染隔离本身就是优化
- 能主动讨论拆分代价：props 钻取、通信成本、跳转成本
- 有从大页面重构下来的真实案例与前后对比

**减分项：**
- 只会说"高内聚低耦合"这类空话，没有可执行信号
- 把"拆组件"等同于"拆文件"，换个文件但状态照样耦合
- 不考虑拆分后的通信成本，拆完更难看

**解答：**

先给判断：不是行数决定拆分，是"变化的原因"决定拆分。三个最实用的信号：一是渲染范围——哪些数据变化只该影响哪块 UI，拆开才能隔离重渲染；二是复用——同一段 UI 或逻辑出现第二次，就值得抽出；三是职责——一个组件同时管数据请求、业务规则、布局、表单控制，任何改动风险面都太大。出现任何一个，就拆。

实操分两步：先"分层"，把数据获取和业务逻辑收敛到容器层，UI 拆成纯展示组件（只依赖 props，便于测试与复用）；再"按渲染粒度拆"——高频变化的小块（输入框、倒计时、进度条）拆小，低频稳定的大块（整张表格）不必过度细分。注意区分"拆文件"与"拆组件"：拆文件只解决行数，拆组件才解决状态耦合，很多人拆了文件但 state 还全挂父级，等于没拆。

```jsx
// 容器组件：管数据与逻辑
function ReportPage() {
  const { data, loading } = useReport();
  return <ReportTable data={data} loading={loading} />;
}
// 展示组件：纯 props 驱动，可独立测试
function ReportTable({ data, loading }) { /* ... */ }
```

实践中的坑：过度拆分会让 props 层层钻取、Context 满天飞，反而更难维护。一条实用红线：拆分后若出现"为传一个值改 5 层组件"，就要考虑组合模式（children 插槽）或状态下沉。组件拆分的本质是管理"变更影响面"，让一次改动只触碰一个文件、一个职责，而不是美学问题。

**延伸考点：** props 钻取严重时，什么场景该上 Context，什么场景该用 children 组合？

### Q4. 用 useEffect 写定时器，count 永远加不到 2，闭包陷阱是怎么回事？

**问题：** 用 `useEffect(() => { const t = setInterval(() => setCount(count + 1), 1000); return () => clearInterval(t); }, [])` 实现计数器，发现 count 停在 1。请解释原因并给出修复方案及取舍。

**期望加分项：**
- 能讲清闭包捕获的是"本次渲染快照"的值，空依赖数组导致 effect 只跑一次、闭包绑定初始值
- 能给出多种修复：函数式更新、补依赖数组、useRef 存最新值，并说明各自适用场景
- 能讲清"把 count 加进依赖"为什么能修、为什么会反复销毁重建定时器
- 能联系 exhaustive-deps 的作用与局限（lint 提示但不保证业务正确）
- 能延伸到事件监听、订阅等同样场景的闭包问题

**减分项：**
- 只会说"把 count 加进依赖"，说不清机制，也不知道会无限重建
- 把函数式更新讲成"套路"，不理解它为什么能绕开旧值
- 不讲 effect 清理（clearInterval）的必要性，或把清理和闭包混为一谈

**解答：**

先讲根因：每次渲染都有自己的 props 与 state，effect 是"这次渲染"的副产品，它内部的闭包天然绑定这次渲染的值。空依赖数组意味着 effect 只在首次挂载执行一次，闭包里的 count 永远是最初的 0，`setCount(0 + 1)` 结果恒为 1，所以永远停在 1。这是 React 心智模型的核心：不要想象"组件里有一个活的 count 变量"，要想象"每次渲染创建一套独立的作用域"。

修复有三条路，取舍不同。函数式更新最省事，直接绕开旧值：

```js
useEffect(() => {
  const t = setInterval(() => setCount(c => c + 1), 1000);
  return () => clearInterval(t);
}, []);
```

如果闭包里还要用别的 props/state，就把它加进依赖数组，让 effect 随值重建（清理函数会先跑，旧定时器被清掉，不会叠加）；若想"定时器稳定 + 读到最新值"，用 useRef 维护一个最新值容器，回调里读 ref。实践中的坑：一是无脑补依赖导致 effect 高频重建，定时器反复销毁重建反而有性能损耗；二是"事件回调"里的闭包——监听器绑定瞬间捕获旧 props，用户点了很久才触发，读到的一直是旧值，这类问题最隐蔽，要用 ref 或 useEffect 内重建监听解决；三是 `exhaustive-deps` 只能提示你"哪些变量被用了"，它不懂业务，依赖数组写什么、要不要重建，最终判断在你自己。

**延伸考点：** 依赖数组里放了一个每次渲染都是新引用的对象，effect 会怎样？如果换成订阅外部事件的监听器，修复思路一样吗？

### Q5. 搜索防抖后仍出现"旧关键词的结果覆盖新关键词"，竞态怎么修？

**问题：** 搜索框做了防抖后请求后端，但快速切换关键词时，偶尔先输入的关键词结果晚返回，把后输入的结果覆盖了。怎么定位和修复？

**期望加分项：**
- 能点出这是典型的请求竞态（race condition），防抖只减请求次数、不保证响应顺序
- 能给出两种方案：请求序号比对、AbortController 取消旧请求，并说明取舍
- 能讲 useEffect 清理函数（cleanup）在取消/丢弃旧响应中扮演的角色
- 能主动提到 React 18 StrictMode 下 effect 双执行对 abort 代码的干扰
- 能考虑"取消请求 ≠ 服务端不执行"，写操作场景不能靠它兜底

**减分项：**
- 只会说"加防抖"，不解决响应乱序本身
- 不知道 AbortController 或请求序号方案
- 把 cleanup 只理解成"清定时器"，不讲取消请求

**解答：**

先判断：防抖只减少请求发起次数，不改变并发响应的到达顺序——后端并行处理时，先发的请求可能后返回，慢响应把新结果覆盖，这就是竞态。排查时打开网络面板确认两条请求的发起/返回时间存在交错，即可实锤。

修复的核心是"只让最后一次请求的结果生效"，两条主流路线。一是请求序号比对：用 ref 维护递增序号，响应回来时检查序号是否仍是最新，不是就丢弃，简单且与请求库无关；二是 AbortController：新请求发起前 abort 掉旧请求，浏览器层面直接丢弃旧响应，前端代码也收不到。

```js
useEffect(() => {
  const ctrl = new AbortController();
  fetch(url, { signal: ctrl.signal })
    .then(r => r.json())
    .then(setData)
    .catch(e => { if (e.name !== 'AbortError') setError(e); });
  return () => ctrl.abort();
}, [keyword]);
```

实践中的坑：一是 React 18 StrictMode 开发模式下 effect 会"挂载→卸载→重挂载"，abort 后的 catch 若不过滤 AbortError，会打出一堆幽灵报错；二是 loading 态防闪烁——被丢弃的旧响应不该关闭 loading，通常用"最新请求序号"控制；三是"取消请求不等于服务端不执行"，写操作（下单、保存）绝不能靠 abort 兜底，要另做防重复提交；四是序号方案要小心闭包，用 ref 而不是 state，否则读到的序号又落后一拍。

**延伸考点：** 竞态发生在"响应与当前 UI 状态"层面（如切换 tab 后旧 tab 数据回填）时，除了取消请求还有什么保证一致性的手段？

### Q6. 把主题和用户信息放进 Context，改主题时整个应用都在重渲染，哪里错了？

**问题：** 你把主题色和用户信息放进 Context，发现切换主题时所有消费该 Context 的组件全部重渲染。Context 不是"全局状态管理"吗？问题出在哪？

**期望加分项：**
- 能讲清 Context 的更新传播机制：Provider 的 value 变化会触发所有消费者重渲染，且不受 React.memo 拦截
- 能指出"高频变化的状态放 Context"是设计问题，主题/用户信息这类低频值才合适
- 能给出优化手段：拆多个 Context、value 用 useMemo 稳定引用、selector 订阅
- 能主动提到"Context 是跨层传值机制，不是状态管理库"这一心智纠正
- 能结合状态选型给出整体判断：高频值该放局部 state 或外部 store

**减分项：**
- 认为 Context 更新只影响"用到它的组件"
- 推荐把 Context 当全局 store 用，讲不出更新链路
- 不知道 value 对象每次渲染都是新引用这个细节坑

**解答：**

先纠正心智模型：Context 不是"全局状态管理"，它只是"跨层传递值的机制"。更新传播规则是：Provider 的 value 变了，所有消费该 Context 的组件（useContext 或 Consumer）都会重渲染，且 React.memo 拦不住——memo 只能按 props 浅比较拦截，拦截不了 context 变化。所以把"频繁变化的状态"放进 Context，等于让一批无关组件绑定重渲染，代价随消费者数量线性放大。

正确的做法是按"变化频率"分层：低频值（主题、语言、用户信息、权限标记）放 Context 几乎零成本，还很方便；高频值（表单输入、轮询数据、实时看板）放 Context 就是性能灾难，应该用组件局部 state 或外部状态库。

```jsx
// value 每次渲染都是新对象，父组件任何一次重渲染都会牵连所有消费者
<ThemeContext.Provider value={{ theme, setTheme }}>
// 正确：useMemo 稳定引用；或把 theme 与 setTheme 拆成两个 Provider
</ThemeContext.Provider>
```

实践中的坑：一是 value 不 memo，父组件随便重渲染一次，所有消费者跟着遭殃，这是 Context 性能问题的最常见来源；二是多个无关状态共用一个 Context，改 A 牵连 B 的消费者，应该拆成多个；三是把"某组件内部状态"误提升到全局 Context，完全没必要。选型上记住一句话：低频值走 Context，高频值走局部 state 或 store。React 19 的 `use()` 与更好的 Context 支持能简化写法，但"按频率分层"的原则不变；另外跨包/跨应用共享状态时 Context 不适用（Provider 穿透不了模块边界），要用外部 store。

**延伸考点：** React.memo 能阻止 Context 变化引起的重渲染吗？如果高频值必须进 Context，怎么把重渲染范围压到最小？

### Q7. 团队规定"所有函数和计算都用 useMemo/useCallback 包一层"，结果更卡了，为什么？

**问题：** 团队规定所有函数和计算都必须用 useMemo/useCallback 包裹，结果页面重渲染更频繁、内存更高。请解释这两个 API 的正确使用边界。

**期望加分项：**
- 能说清 useMemo/useCallback 的本质是"缓存"，本身有比较依赖、存储旧值的成本
- 能指出"计算便宜时缓存是负优化"，"缓存收益 > 缓存成本"才划算
- 能列出真正值得用的场景：昂贵计算、稳定引用（配合 memo 子组件、effect 依赖）
- 能有"先量化再优化"的工程态度，提到 DevTools Profiler 验证
- 能联系 React 19 React Compiler 自动记忆化的趋势

**减分项：**
- 把 useMemo/useCallback 当性能银弹，见函数就包
- 说不清"依赖变化导致缓存失效重建"的成本
- 用 useMemo 包简单表达式，却讲不出任何收益

**解答：**

先纠错：useMemo 缓存计算结果，useCallback 缓存函数引用，它们的本质是"缓存"——每次渲染要比较依赖、读旧值、失效时重建，这些都要成本。只有当"缓存收益 > 缓存成本"时才值得用。一个简单表达式（字符串拼接、取个长度）包 useMemo，收益为零、成本照付，是教科书级的负优化；团队"统一包裹"的规矩，正是把优化手段变成仪式感的典型反例。

真正值得用的只有两类场景。一是昂贵计算：千级数组的过滤/排序、复杂格式化、传递给重子组件的派生数据，这类计算每次重渲染都重跑，缓存能实打实省时间。二是稳定引用：子组件被 React.memo 包裹且依赖这个函数/对象做 props 浅比较，或函数被放进 effect 依赖数组——引用不稳定会导致 memo 失效、effect 反复执行。

```js
// 值得：千行数据过滤，重渲染时省一次全量计算
const visible = useMemo(() => filter(list, kw), [list, kw]);
// 不值得：简单拼接包一层，纯属摆设
const label = useMemo(() => `共${n}条`, [n]);
```

工程姿势：先用 Profiler 确认"重渲染确实在拖性能"，再针对热点下手；React 19 的 React Compiler 会在编译期自动做记忆化，届时手写 useMemo 的收益进一步下降。实践中的坑：一是依赖数组里放了"每次渲染都是新引用"的对象，useMemo 缓存永远失效，等于白写还加心智负担；二是 useCallback 包了函数但子组件根本没 memo，等于白包；三是过早优化成瘾，代码可读性先崩。原则一句话：性能优化要"先测后优"，缓存是手段不是纪律。

**延伸考点：** React Compiler 自动记忆化上线后，手写 useMemo/useCallback 还需要吗？什么代码模式会让 Compiler 失效？

### Q8. 修改表格某一行，整个页面都在闪烁、图表重新动画，怎么定位与修复？

**问题：** 修改表格中的一行数据，结果整页组件都在重渲染，图表重新播放动画、表单闪烁。请给出从怀疑、定位到修复的完整路径。

**期望加分项：**
- 能用 React DevTools Profiler 查看火焰图，识别每个 commit 中各组件的重渲染原因
- 能讲清"父组件重渲染默认导致所有子组件重渲染"的机制，及"协调 vs 提交"两个阶段
- 能按性价比排序给出优化手段：state 下沉、拆分组件、React.memo、useCallback 稳定引用
- 能区分"重渲染"与"性能问题"：重渲染不等于卡顿，要看 commit 成本
- 有真实案例：优化前后 commit 次数/耗时的量化对比

**减分项：**
- 一上来就全局 React.memo，不做定位
- 把"重渲染多"直接等同于"卡顿"，讲不清中间环节
- 只会背优化手段清单，没有定位方法

**解答：**

先建立概念：重渲染分两步——协调（reconcile，运行组件函数生成新的虚拟结果）和提交（commit，diff 后更新真实 DOM）。性能问题大多出在协调环节，因为 React 默认"父组件重渲染，子组件全部重渲染"，不管子组件 props 有没有变，组件树越深、节点越多，浪费越大。所以第一步永远是定位，不是加 memo。

用 React DevTools Profiler 录制一次"改行数据"的交互，火焰图会标出每个 commit、每个组件的重渲染原因（props changed / state changed / context changed / parent changed）。先看"改一行"引发了哪些不该动的子树。常见元凶按频率排：状态放得过高（每行都依赖页面级 state）、Context value 未 memo、传入 memo 子组件的回调每次新建、列表 key 不稳定导致整表重建。

修复按性价比排序：① state 下沉/拆分，让变化只影响局部（收益最大）；② 给"重但 props 稳定"的子树包 React.memo；③ 配合 useCallback/useMemo 稳定引用；④ 最后才考虑引入状态库。实践中的坑：memo 是浅比较，props 里塞了每次新建的对象（如 `style={{}}`、内联 `.map()` 结果），memo 直接失效——"看似 memo 了却没生效"是真实项目里最常见的返工点。优化完用 Profiler 复测，对比 commit 数量与耗时，用数据收尾，而不是"感觉快了"。

**延伸考点：** 除了稳定引用，还有什么办法绕过 memo 浅比较失效的问题（如自定义 compare 函数）？什么时候值得自己写 compare？

### Q9. 新项目 20 个页面，状态管理用 Redux Toolkit 还是 Zustand/Jotai？怎么选？

**问题：** 新项目有登录态、用户资料、购物车以及大量页面局部状态，技术负责人纠结用 Redux Toolkit 还是 Zustand（或 Jotai）。请给出选型依据与决策框架。

**期望加分项：**
- 能按状态类型分类：服务端状态、客户端全局状态、局部 UI 状态，管理策略各不相同
- 能讲清 Redux 的定位与代价：可预测性、DevTools 时间旅行、范式约束 vs 样板代码、学习成本
- 能讲 Zustand 的定位：无 Provider、外部 store、selector 按需订阅天然避免全树重渲染
- 能给出具体决策条件：团队规模、状态复杂度、调试与协作需求、迭代速度
- 能批判看待"Redux 已死"论调，承认它在大型协作中的规范价值

**减分项：**
- 只按个人喜好推荐，说不出依据
- 把"全局状态"当唯一答案，忽略服务端状态该交给请求缓存库
- 不知道 Zustand 的 selector 订阅机制与 useShallow 的坑

**解答：**

先做比选库更重要的区分：状态按来源分三类——服务端状态（接口数据）交给请求缓存库（TanStack Query 等），客户端全局状态（登录态、主题、权限）才需要 store，局部 UI 状态（弹窗开关）留在组件里。很多人把接口数据硬塞进 Redux，正是"状态管理越管越乱"的根源，选库之前先把这个分层定下来。

选型看"协作复杂度"：团队大、状态流转规则多、需要严格可追溯（交易/合规类业务）→ Redux Toolkit 的范式约束 + DevTools 时间旅行价值大，样板代码是买"团队一致性"的成本；中小团队、状态简单、追求迭代速度 → Zustand 的外部 store + selector 订阅（`useStore(s => s.count)`）天然做到"只重渲染依赖该 slice 的组件"，且无 Provider 包裹、心智负担低，是目前性价比最高的默认选择。Jotai 适合"原子化、粒度细"的场景（多字段联动的表单），代价是范式太灵活，大项目约束不足。React 19 后 context+`use()` 对轻量共享也够用，但跨模块/跨应用还是要 store。

实践中的坑：一是"为了用而用"——两个页面共享一个值就上 Redux，结果一半状态在 store、一半在 props，查状态要翻 5 个文件；二是 Zustand 里 selector 返回新对象（`s => ({a: s.a, b: s.b})`）会破坏按需订阅导致重渲染，需要 `useShallow`；三是多人协作没有规范（哪些状态进 store、命名规则），store 变成垃圾桶。结论没有银弹，能说出"按状态类型分层 + 按团队规模取舍"的决策框架，就比报菜名式推荐高一个段位。

**延伸考点：** Zustand 的 selector 订阅机制如何实现"只重渲染依赖该值的组件"？坚持把服务端数据放 Redux 会遇到哪些具体问题？

### Q10. 一张表 1 万行 × 20 列，直接 map 渲染卡到不能滚动，怎么办？

**问题：** 数据表要展示 1 万行、每行 20 列，直接 map 渲染后首屏 3 秒、滚动卡顿。请从"为什么卡"出发，给出方案选型与实现要点。

**期望加分项：**
- 能量化卡顿来源：节点数量与 DOM/布局成本近线性，滚动时每帧重新布局
- 能区分可行方案：服务端分页、windowed 虚拟列表、时间分片（并指出分片治标不治本）
- 能讲虚拟列表原理：固定高度容器 + 占位撑高 + 可视区窗口 + overscan，及不定高内容的测量难点
- 能给出库选型：react-window / TanStack Virtual 的取舍，以及表格场景的特殊性（固定列、合并单元格）
- 能主动考虑"1 万行数据本身在内存"还是"需要分页"的业务语义

**减分项：**
- 只会说"用虚拟列表"，讲不出原理与坑
- 不知道不定高行（图片、换行）的测量难题
- 把分页当唯一答案，忽略"数据必须整体在内存做筛选排序"的场景
- 忽略 20 列本身带来的渲染与样式开销

**解答：**

先量化"卡在哪"：1 万行 × 20 列 ≈ 20 万个单元格节点，DOM 节点数与首帧布局（Layout）耗时近线性增长，滚动时每帧都要重新计算布局，卡顿是必然。方向不是"让渲染更快"，而是"只渲染该渲染的"。

三条路线按场景选：数据有明确分页语义（查询类报表）→ 服务端分页，最省；数据必须整体在内存做筛选/排序/导出 → 虚拟列表；两者都不满足再谈时间分片（把渲染拆帧，但最终仍全量挂载，治标不治本，只适合过渡）。虚拟列表核心：外层容器固定高度 + 一个撑总高度的占位 div + 根据 scrollTop 和行高算出可视窗口 + 只渲染窗口内行 + overscan 多渲染几行缓冲滚动。坑全在"行高不定"：图片加载、文本换行会让占位高度失准，需要动态测量（ResizeObserver 量完缓存每行高度，或用"估算行高 + 滚动时校正"策略）。

```js
// TanStack Virtual 支持动态测量
const virtualizer = useVirtualizer({
  count: 10000,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 35, // 估算行高，真实高度会被测量修正
});
```

表格场景更复杂：固定列需要左右双层结构、合并单元格破坏"行内均匀高度"假设、横向滚动与纵向虚拟化叠加后坐标计算出错。选 react-window（轻、成熟、适合均匀行高）还是 TanStack Virtual（灵活、内置动态测量）取决于表格复杂度。实践中的坑：虚拟列表包进滚动容器而不是页面级 scroll 元素，导致 getScrollElement 拿错节点白屏；虚拟化后"全选、滚动到指定行、导出当前可视区"这些功能都要重新实现，别等上线才发现。

**延伸考点：** 虚拟列表里做"筛选后滚动位置复位"怎么做？不定高行测量对滚动性能的实际影响有多大？

### Q11. 首屏 JS 包 2MB，其中图表库、富文本编辑器占 60% 且只在少数页面用，怎么拆？

**问题：** 首屏 JS 包 2MB+，60% 是图表库和富文本编辑器，只在个别页面用到。请给出代码分割与加载优化的完整方案，包括验证手段。

**期望加分项：**
- 能用 React.lazy + Suspense 做路由级/组件级代码分割，说明 chunk 划分边界
- 能说清 Suspense 的 fallback 语义：lazy 组件挂起时展示，以及它不处理加载失败
- 能给出验证手段：webpack-bundle-analyzer / Vite manualChunks，按 chunk 构成验收
- 能讲拆包的坑：公共依赖重复打包、chunk 过碎导致请求数爆炸、部署后旧 chunk 404
- 能主动聊预加载策略：hover 预热、路由预取、空闲加载

**减分项：**
- 只会说"用 React.lazy"，说不清拆包边界怎么定
- 把 Suspense 当成"loading 组件"，不清楚它和错误边界的配合
- 拆完不验证，不知道入口 chunk 是否真的变小

**解答：**

先给方向：把"首屏不用的重型库"变成按需 chunk，是收益最高的首屏优化。React.lazy 包裹图表/编辑器页面组件，打包器为每个 lazy 组件生成独立 chunk，首屏只下载入口 chunk；Suspense 提供加载态，加载完成后渲染组件。

```jsx
const ChartPage = lazy(() => import('./pages/ChartPage'));
<Route path="/chart" element={
  <Suspense fallback={<PageLoading />}><ChartPage /></Suspense>
} />
```

拆包边界：按路由拆是基础；组件级拆更细——某个弹窗里才用的编辑器单独 lazy；库级拆分交给 splitChunks 缓存组，把 echarts、编辑器这类大户抽成独立 vendor chunk 并设长缓存。验证必须用工具：webpack-bundle-analyzer（或 Vite 构建产物分析）看每个 chunk 构成，目标是入口 chunk 压到可接受基线（常见参考 < 300KB gzip），并确认重型库确实进了异步 chunk。

实践中的坑：一是 Suspense 只能包 lazy 组件，同步 import 的大库必须配 splitChunks 才能拆；二是拆太碎导致请求数爆炸，弱网下 fallback 来回闪烁（体验比不拆还差）；三是部署后旧 chunk hash 失效 404，用户卡在 loading 或白屏——必须给 Suspense 外再包错误边界并做"失败自动重试一次/刷新兜底"；四是 Vite 默认 chunk 策略与 webpack 不同，需要 manualChunks 手动控制。进阶是预加载：hover 到导航链接时 `import()` 预热对应 chunk，把"点击后转圈"变成"几乎秒开"，这是大型中后台的标配。

**延伸考点：** 部署新版本后用户停留在旧页面，异步 chunk 404，Suspense 会如何表现？如何优雅降级与自愈？

### Q12. 搜索框输入时结果列表（几千条）立即重渲染导致掉帧，startTransition 能解决吗？

**问题：** 搜索框输入时，下方几千条的结果列表每次都立即重渲染，输入掉帧。用 startTransition 包住结果更新后输入流畅了。请解释原理、适用边界与坑。

**期望加分项：**
- 能讲清 startTransition 的语义：把更新标记为可中断/可延迟，React 优先处理紧急更新（输入）
- 能说明 useTransition 的 isPending 如何用于 UI（过渡提示、旧结果遮罩）
- 能说清适用边界：非紧急、可丢弃的 UI 更新（结果列表、筛选），而非必须同步的数据写入
- 能谈过渡中断时的状态一致性：旧值展示 + isPending 提示的心智
- 能对比 useDeferredValue，说清两者使用场景的差异

**减分项：**
- 把 startTransition 当通用性能优化到处乱包
- 不知道它不适用于"受控输入框自身的 value"更新
- 说不清 isPending 的作用，或过渡期间新旧值不一致的困惑

**解答：**

先讲原理：React 18 引入并发渲染，startTransition 会把内部的状态更新标记为"过渡"（non-urgent），React 可以在渲染过渡更新时，被更高优先级的紧急更新（如输入框每次击键）打断——被打断的部分直接丢弃，先保证输入响应，再回头补渲染结果列表，所以输入不掉帧。useTransition 额外返回 isPending，用于过渡未完成时的 UI 反馈（旧结果加遮罩、轻微置灰）。

```jsx
const [isPending, startTransition] = useTransition();
const onChange = (e) => {
  setKeyword(e.target.value); // 紧急：输入框必须立即响应
  startTransition(() => setFiltered(filter(list, e.target.value))); // 过渡：列表可以慢
};
```

适用边界是"结果可以延迟、可以被丢弃"的更新：搜索联想、筛选、图表联动、tab 切换。绝不能用于：表单提交后的状态同步、任何用户明确等待结果的写操作——延迟它们只会让用户更焦虑，且过渡被反复打断时结果迟迟不落地，感知更糟。与 useDeferredValue 的区别：startTransition 是你主动选择"哪个更新是过渡"；useDeferredValue 是把"某个值的派生结果"自动延后，适合输入值与派生结果天然同源的场景（输入框 value + 过滤列表）。

实践中的坑：一是过渡渲染是异步的，测试里同步断言状态会测不准，需要 `act` 配合；二是在过渡里更新受控输入框自身的 value，输入会"感觉迟钝"（光标都卡）；三是无差别包裹所有 setState，反而增加调度开销。判断标准一句话：这个更新"晚 100ms 完成，用户能接受吗"？能，就过渡；不能，就同步。

**延伸考点：** 过渡期间用户再次输入，React 的调度策略是什么？isPending 的 UI 怎么设计才不会闪烁？

### Q13. 三个页面都有"滚动加载更多"，各写了一遍还漏了去重，怎么设计自定义 Hook？

**问题：** 三个页面分别实现了"滚动加载更多"逻辑，其中一处漏了 loading 去重导致重复请求。请设计一个可复用的自定义 Hook，并说明设计原则与边界处理。

**期望加分项：**
- 能给出 hook 设计：入参（请求函数、分页配置、字段映射）、出参（list、loading、hasMore、loadMore、refresh）
- 能讲抽象原则：复用"交互模式"而非"业务"，不硬编码接口与字段
- 能主动处理边界：loading 去重、页码竞态、错误重试、卸载后 setState
- 能谈职责边界：Hook 管逻辑，谁触发加载由组件决定
- 能提到测试方式（renderHook）与函数式更新防闭包旧值

**减分项：**
- Hook 里硬编码接口和业务字段，换页面没法复用
- 忘记并发保护（loading 期间重复触发 loadMore）
- 用 state 存页码导致闭包旧值，或依赖数组写死 lint 报错

**解答：**

先给设计原则：自定义 Hook 复用的是"逻辑模式"，不是业务。入参给"能力"——请求函数、列表字段映射、分页大小；出参给"状态机"——list、loading、hasMore、loadMore、refresh、error。业务页面只传自己的请求函数，三处复用且互不干扰。核心实现必须覆盖三个坑：loading 去重（防重复请求）、页码不依赖闭包（用 ref 而非 state）、组件卸载后不再 setState。

```js
function useInfiniteList({ fetcher, pageSize = 20, listKey = 'list' }) {
  const [list, setList] = useState([]);
  const [hasMore, setHasMore] = useState(true);
  const loadingRef = useRef(false);
  const pageRef = useRef(1);
  const loadMore = useCallback(async () => {
    if (loadingRef.current || !hasMore) return; // 去重
    loadingRef.current = true;
    try {
      const { data } = await fetcher(pageRef.current, pageSize);
      setList(l => [...l, ...data[listKey]]); // 函数式更新，防闭包旧值
      pageRef.current += 1;
    } catch { /* 暴露 error 状态供 UI 重试 */ }
    loadingRef.current = false;
  }, [fetcher, hasMore, pageSize, listKey]);
  return { list, loading: loadingRef.current, hasMore, loadMore };
}
```

实践中的坑：一是 fetcher 引用不稳定会让 loadMore 依赖数组频繁变化——更稳的做法是用 ref 存最新 fetcher；二是错误处理不能只 catch 不提示，要留 error 状态让 UI 提供重试入口；三是"谁触发 loadMore"（IntersectionObserver、滚动事件、按钮）由组件决定，Hook 只暴露能力，别把 observer 也焊死在 Hook 里；四是测试用 renderHook + 假 fetcher，验证并发触发只发一次请求。另外要分清复用边界：Hook 复用逻辑，组件复用 UI——同一种列表在不同页面长得不一样时，"Hook + 展示组件"配合才是正解。

**延伸考点：** fetcher 每次渲染都是新引用，loadMore 的依赖数组会发生什么？用 ref 存 fetcher 与用 useCallback 包 fetcher，哪种更稳、为什么？

### Q14. 接口 500 后整个页面白屏，但接口调用明明有 try/catch，怎么回事？

**问题：** 线上偶尔出现"某接口 500 后整个页面白屏"，代码里接口调用明明有 try/catch。请定位原因，并给出完整的异常处理方案。

**期望加分项：**
- 能指出 try/catch 捕获不了"渲染阶段"抛出的错误，需要错误边界（class 组件的 getDerivedStateFromError / componentDidCatch）
- 能讲清错误边界的放置位置：路由级、模块级、关键表单级，分别兜什么
- 能谈错误边界的局限：事件回调、异步代码、Promise rejection 不触发
- 能结合监控上报（Sentry）与降级 UI 设计
- 能主动做防御性渲染（接口数据判空），从源头减少抛错

**减分项：**
- 把 try/catch 当万能方案，不知道渲染期错误的特殊性
- 错误边界包一层就完事，不设计 fallback UI 与上报
- 不知道函数组件不能直接做错误边界（要封装 class）

**解答：**

先定位：接口 500 的响应明明被 catch 住了，页面却白屏，说明报错不在异步回调里，而在渲染阶段——接口失败后你 setState 了异常数据（比如把 undefined 塞进列表），render 时访问 `data.items.map` 抛错。try/catch 只能包住它所在的作用域（异步函数内部），捕获不了 React 渲染组件时抛出的同步异常，于是 React 卸载整棵组件树，页面白屏。解法是错误边界：class 组件实现 `getDerivedStateFromError`（渲染降级 UI）+ `componentDidCatch`（上报监控）。

```jsx
class ErrorBoundary extends Component {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  componentDidCatch(error, info) { reportError(error, info); }
  render() { return this.state.hasError ? <FallbackUI onRetry={...} /> : this.props.children; }
}
```

边界的位置决定兜底粒度：路由级包一个（防整页白屏，提供刷新/回退），每个独立业务模块（表格、看板）各包一个（局部出错不拖垮整页），关键流程（下单表单）再包一层。函数组件不能直接实现错误边界，但可以封装成 `<ErrorBoundary>` 组件使用。

实践中的坑：一是错误边界只拦"渲染期 + 生命周期"的错误，onClick 等事件回调里 throw 不触发，要自己 try/catch 或全局 error 监听；二是 Promise rejection 不触发边界，要加 `window.addEventListener('unhandledrejection')` 兜底；三是 React 18 并发渲染下错误边界的触发时机有变化，升级后要回归测试；四是别把错误边界当第一道防线——接口数据先做默认值/判空（`data?.items ?? []`）从源头减少抛错，错误边界是兜底不是首选。

**延伸考点：** React 19 对错误边界/并发渲染下的错误处理有什么变化？事件回调里的错误除了 try/catch 还有什么统一兜住的手段？

### Q15. 50 个字段的配置表单，每输入一个字符整表重渲染，怎么优化？

**问题：** 50 字段的配置表单，用 state 存整个 form 对象，onChange 里 `setForm({ ...form, [name]: value })`，输入卡顿明显。请分析根因并给出优化方案。

**期望加分项：**
- 能点出根因是两层叠加：form 对象在页面级 state 导致整棵表单树重渲染 + 每次击键拷贝大对象
- 能给出方案分级：字段级拆分（state 下沉）、非受控 + 提交取值、第三方库（React Hook Form）
- 能讲复杂联动（字段间依赖）下的实现权衡
- 能谈 useMemo 包裹派生计算（校验、格式化）的正确姿势与依赖
- 能用 Profiler 量化优化前后 commit 变化

**减分项：**
- 只会说"用 React Hook Form"，讲不出原理
- 把优化等同于"加 memo"，不改 state 结构
- 忽略联动字段的复杂性，方案落地不了

**解答：**

先拆根因，卡顿是两层叠加：一是范围——form 对象挂在页面级 state，任一字段 onChange 都让 50 个字段的整棵子树重渲染；二是成本——`setForm({ ...form })` 每次拷贝含 50 个属性的对象，字段越多越明显。优化从"缩小范围"入手，方向有二。

方向一：字段级拆分。把每个输入项封装成独立组件，state 下放到字段内部，只有提交或联动需要时才上报到父级。这样击键只重渲染当前字段，是零依赖的通用解法。

```jsx
function FormField({ name, initial }) {
  const [v, setV] = useState(initial); // 值留在字段内
  return <input value={v} onChange={e => setV(e.target.value)} />;
}
```

方向二：借成熟方案。React Hook Form 默认非受控（ref 注册 + 提交取值），配合 Controller 处理联动；或保留受控但拆分 + 派生计算放 useMemo。字段间联动（A 变化重置 B）是真正的难点：用 RHF 的 watch 或 onBlur 时同步，避免每个击键都触发全局联动。实践中的坑：一是"受控 + 表单对象"再叠加校验、格式化等派生计算时，计算必须放 useMemo 并缩小依赖，否则每击一键全量重算；二是在渲染周期里做格式化再 setState 会死循环；三是非受控 + 提交取值的"幽灵值"问题——提交时确认数据已同步。最后用 Profiler 验收：优化前每次击键 commit 整个表单，优化后只 commit 单个字段。

**延伸考点：** 50 个字段里有 10 个需要跨字段联动，方案怎么调整？React Hook Form 的 watch 性能边界在哪？

### Q16. 有人把接口数据塞进全局 store，多页面共享但数据总是旧的，问题在哪？

**问题：** 项目里有人把接口数据塞进 Redux/Zustand 全局 store，多个页面共享同一份数据，但每次回到页面数据还是旧的、还要手动刷新。请指出问题根源，并给出正确的数据层设计。

**期望加分项：**
- 能点出"服务端状态"与"客户端状态"的本质区别：前者以"与服务器保持一致"为核心（缓存、失效、重取）
- 能讲 TanStack Query 的核心机制：queryKey、staleTime、gcTime、invalidateQueries、useMutation
- 能举出具体痛点：手动刷新逻辑分散、多页面缓存不一致、loading/error 样板代码
- 能讲与 UI 状态库的分工：Query 管服务端，Zustand/Redux 管客户端全局态
- 能结合乐观更新、轮询、窗口聚焦重取等真实需求谈取舍

**减分项：**
- 坚持"接口数据就放全局 store"，讲不出失效与缓存一致性方案
- 把 React Query 当成"请求封装工具"，不理解缓存语义
- 不知道默认 staleTime=0、refetchOnWindowFocus 的默认行为

**解答：**

先给判断：把接口数据当"客户端状态"塞进全局 store，是状态管理混乱的根源。服务端状态的生命周期由服务器决定——会过期、会被别处修改、需要重取，用 Redux 管理它意味着要自己实现缓存、失效、去重、重试一整套路，每页手动 useEffect 拉数据 + 手动刷新，缓存不一致、样板代码爆炸是必然结果。

正确分工：TanStack Query 管服务端状态，它把"请求"变成"缓存系统"——同一 queryKey 的请求自动共享去重，staleTime 控制新鲜度，写操作成功后 invalidateQueries 使读缓存失效并自动重取，useMutation 支持乐观更新：

```js
const { data } = useQuery({ queryKey: ['users'], queryFn: fetchUsers, staleTime: 30_000 });
const { mutate } = useMutation(updateUser, {
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['users'] }),
});
```

它还把 loading/error/retry 收敛成一个 hook，样板代码大幅下降。而 Zustand/Redux 只存"客户端全局态"：登录态、主题、用户偏好这类由应用自身管理、不依赖服务器生命周期的值。实践中的坑：一是不知道默认 staleTime=0，窗口切回来就疯狂重取，要按数据变化频率显式配置；二是 queryKey 设计不当（漏掉 filter 参数）导致缓存串数据；三是把表单暂存值也丢进 Query 缓存，那本该是本地状态；四是乐观更新失败回滚的复杂度高，大表单编辑场景用"提交后 invalidate"更稳。记住一句话：服务端状态问"如何与服务器保持一致"，客户端状态问"如何跨组件共享"。

**延伸考点：** staleTime 与 refetchOnWindowFocus 的默认行为是什么？什么业务下必须显式调整？乐观更新失败回滚怎么做才不闪屏？

### Q17. Next.js 项目首屏白屏 5 秒、内容出来后交互还卡，SSR 的问题出在哪？

**问题：** 一个 Next.js SSR 项目首屏白屏 5 秒（LCP 差），页面大量内容由服务端渲染，但用户看到骨架屏后内容才突然出现，之后点击还卡。请拆解首屏时间花在哪，并给出优化路径。

**期望加分项：**
- 能拆段量化：TTFB（服务端渲染 + 网络）、HTML 内容到达、可交互（JS 下载 + 水合完成）
- 能区分"白屏"与"交互卡"的不同成因：服务端慢 vs 水合（hydration）阻塞
- 能讲水合成本：服务端 HTML 已渲染但 JS 未执行完前不可交互，水合耗时直接贡献 INP
- 能给出手段：服务端缓存/异步化压 TTFB、代码分割、Streaming SSR（renderToPipeableStream）
- 能讲 Next.js 13+ App Router / Server Components 如何减少客户端 JS 与水合量
- 能谈到 hydration mismatch 的闪变问题

**减分项：**
- 把 SSR 首屏慢直接归因于"服务器慢"，看不到水合
- 不知道 Streaming SSR 与 RSC 的存在
- 说不出 SSR 中 useState 初值不一致导致闪变的细节坑

**解答：**

先拆段量化"5 秒花在哪"：TTFB（服务端渲染耗时 + 网络）→ HTML 内容到达（用户能看到文字）→ 可交互（JS 下载 + 执行 + 水合完成）。SSR 最隐蔽的成本在第三段：服务端 HTML 已经渲染出来，用户能看到内容，但页面"不可交互"——水合阶段 React 要在客户端把每个节点重新跑一遍组件逻辑再绑定事件，JS 越大、组件树越深，水合越慢，直接拉高 INP。若客户端 state 初值还与服务端不一致，水合后内容还会"闪变"。判断方向：TTFB 高 → 服务端问题（数据库慢、同步阻塞、无缓存）；内容出了但交互卡 → 水合阻塞（客户端 JS 太大、全量水合）。

优化按收益排序：① 服务端侧：页面/接口缓存（ISR、full route cache）、异步化阻塞代码，压 TTFB；② 客户端侧：路由级代码分割 + 把重型交互组件（图表）包进 Suspense 延迟水合；③ React 18 Streaming SSR：`renderToPipeableStream` 先吐骨架再吐内容，配合 Suspense 实现"边下边渲染"；④ 升级路径：App Router 的 Server Components 把不需要交互的组件在服务端渲染成纯 HTML，客户端不下发它们的 JS，从根上减少水合工作量——JS 变少对 LCP 和 INP 是双重收益。

实践中的坑：一是 `useEffect` 里 setState 或依赖 `window` 的取值，导致"服务端 HTML 与客户端首渲染不一致"的 hydration mismatch（线上表现为内容闪变）；二是把整页塞进 Suspense，反而让首屏内容也延后；三是 SSR 请求要处理去重与缓存，避免每个组件各拉一次接口。最后用 LCP/TTFB/INP 三个指标 + 分段打点验收优化效果，而不是"感觉快了一点"。

**延伸考点：** Server Components 为什么能同时改善 LCP 和 INP？Streaming SSR 与"先出骨架后出内容"的渲染时序是怎样的？

### Q18. 微前端下多个 React 应用各自打包一份 React，能共享吗？架构怎么设计？

**问题：** 公司要把多个独立系统合并成微前端（qiankun 或 Module Federation），每个子系统都是 React 18 应用，各自打包一份 React，导致首屏变重、版本冲突、跨应用状态难共享。请给出架构方案与取舍。

**期望加分项：**
- 能讲微前端的核心问题：JS 隔离（沙箱）、样式隔离、应用间通信
- 能讨论共享 React 的方案：externals + 单一运行时、Module Federation 的 shared/singleton 配置，及版本协商
- 能诚实评估共享代价：升级联动、singleton 冲突排查难、生态约束
- 能讲状态共享边界：跨应用只传低频快照数据（用户、权限），高频业务状态各自管理
- 能结合样式隔离、路由 base、灰度发布等实操细节
- 能对"单体重构 vs 微前端"给出批判性判断

**减分项：**
- 把微前端当终极架构，不讲成本与替代方案
- 不知道 qiankun 沙箱与样式隔离的基本原理
- 强行共享 React 导致版本冲突，却说不出冲突的具体表现

**解答：**

先给判断：微前端的本质是"运行时集成"，核心要解决三个问题——JS 隔离（沙箱）、样式隔离、应用间通信。qiankun 用 Proxy 沙箱 + 样式处理解决前两个，但"每个子应用独立打包 React"带来的体积与版本问题必须自己解决。共享 React 的前提是版本一致：主应用把 React/ReactDOM 做成 externals（公共资源加载），子应用声明 external 不打包；Module Federation 更成熟，用 `shared: { react: { singleton: true, eager: true } }` 让多应用共享同一实例，并支持版本协商（不匹配时回退到各自版本）。

共享的代价要诚实评估：React 升级变成全公司联动（一个子应用要 19 新特性，得说服所有人一起升）；singleton 模式下运行时版本冲突报错诡异难排查；共享 React 后组件库、路由库往往也要共享，约束越来越紧。所以主流建议是"只共享运行时（React/ReactDOM），不共享业务代码"——业务模块独立演进，靠约定协议或 Manifest 交互。

状态共享边界：子应用间传"低频快照型"数据（用户信息、菜单权限）走 qiankun globalState 或自定义事件；高频业务状态（列表、表单）严禁跨应用共享，各应用自己管，否则状态流变成不可控的全局总线。实践中的坑：子应用间全局变量互相污染（沙箱降级为 legacy 时）、样式注入顺序打架（需要 class 前缀规范）、主子应用路由冲突（router base 配置）。最后一句批判性判断：如果只有 3 个子系统且改动不频繁，单体重构可能比微前端更便宜——微前端是"组织协作问题"的工程解，不是性能解，别为了架构而架构。

**延伸考点：** qiankun 的 Proxy 沙箱在什么场景下会失效（原生事件、第三方全局插件）？Module Federation 的 singleton 版本冲突时会有什么表现？

### Q19. 报表页面同时有 1 万行表格、实时看板卡片、图表、拖拽排序，性能架构怎么设计？

**问题：** 一个报表页面同时有：1 万行表格、实时更新的看板卡片、时间轴图表、拖拽排序。滚动卡、看板更新时表格也闪、拖拽后整页重排。请给出该页面的性能架构设计。

**期望加分项：**
- 能按"数据源与渲染成本分层"分析，而非零散优化
- 能设计：实时数据与静态数据分离，订阅隔离避免一处更新全页重渲染
- 能给出模块级方案：表格虚拟化、卡片独立订阅 + 局部更新、图表数据 useMemo 稳定引用
- 能讲拖拽交互隔离：拖拽期只更新位置层，松手才提交顺序
- 能定性能预算与验收：commit 频率、FPS/INP、Profiler 复测
- 有真实优化案例与量化对比

**减分项：**
- 只会零散说"用 memo、用虚拟列表"，没有架构视角
- 把实时数据直接 setState 到页面级，不讲订阅隔离
- 忽略拖拽排序与大数据表格叠加的交互复杂度

**解答：**

先建立架构视角：这种页面的核心矛盾是"一个数据源、多种消费频率、多种渲染成本"。解法是"按频率与成本分层隔离"，而不是全局优化。分三层：

第一层，数据源分离：实时数据（WebSocket/轮询驱动的看板卡片）与静态数据（表格、图表历史数据）分开管理。实时数据更新只影响订阅它的卡片组件——用 Zustand selector 订阅或组件内部独立管理，绝不能 setState 到页面级再让表格跟着重渲染。第二层，渲染成本隔离：表格走虚拟列表（只渲染可视区），图表数据用 useMemo 稳定引用（图表库重渲染成本极高且会闪烁），看板卡片各自独立组件，高频更新在卡片内部消化。第三层，交互隔离：拖拽排序是高频交互，状态必须独立于数据渲染——拖拽期间只更新位置层（transform 位移），松手才提交新顺序，避免拖一下整表重建。

```js
// 实时数据只驱动卡片，不触碰表格
const cards = useCardsStore(s => s.cards); // 按 slice 订阅
// 静态表格数据独立管理，与实时流解耦
```

性能预算 + 验收：先定"页面 commit 频率"预算（如 60FPS 下每帧至多 1-2 次 commit），用 Profiler 记录每次交互引发的 commit 数量与耗时；卡顿回归可接自动化（measure 帧时长、性能 trace）。实践中的坑：一是把"数据变化"和"UI 更新"耦合，节流能缓解但根因是订阅粒度粗；二是 WebSocket 消息频率过高（每秒几十条）时，局部更新也会打满主线程，要按窗口合并消息；三是图表重渲染不仅慢还会闪，稳定引用 + 单次更新最有效；四是 React 18 自动批处理在原生事件/定时器里也生效，但 fetch 回调等异步场景要确认，避免一次 set 多次重渲染。这类页面的性能问题 90% 是"更新范围过大"，架构目标就是让每次数据变化只触碰它该触碰的 UI。

**延伸考点：** 每秒 30 条 WebSocket 消息驱动多个卡片，除了订阅隔离还要做什么才能保住 60FPS？自动批处理在这里起什么作用？

### Q20. 项目还在 React 17，作为技术负责人你如何决策升级到 React 19，并安排落地？

**问题：** 公司项目停留在 React 17，团队想升级到 React 19。作为技术负责人，请给出升级决策、分阶段路径与风险控制，并谈谈你对 React 渲染架构演进的理解。

**期望加分项：**
- 能给出分阶段路径：17→18（createRoot、并发、自动批处理）→19（Actions、use、ref as prop、React Compiler），各步收益与风险
- 能讲 React 19 关键特性对工程的实质影响：Actions 简化表单、use()、React Compiler 消除手写 memo、RSC
- 能批判评估：不追新，能说清"哪些特性解决当前痛点、哪些用不上"
- 能讲升级的工程风险：第三方库兼容矩阵（UI 库、图表库、编辑器）、类型与测试适配、一次切不回头
- 能展示对渲染架构的理解：Fiber → 并发渲染 → Server Components 的逻辑主线
- 有量化的升级收益（包体积、INP、代码删除量）与验收手段

**减分项：**
- 无脑"升级到最新"，说不出收益与成本
- 对 React 19 特性一知半解，讲不出对项目的具体影响
- 没有渐进迁移与回滚方案，风险全裸奔
- 把 Server Components 与 SSR 混为一谈

**解答：**

先给决策框架：升级不是追新，是投资回报决策。收益要落到具体指标：React 18 的并发特性 + 自动批处理改善交互性能（INP）、React 19 的 React Compiler 消除大量手写 useMemo/useCallback（代码量下降）、Actions 简化表单状态机、RSC（若用 Next.js）减少客户端 JS。成本是生态兼容：先做依赖审计——列全部第三方库的 React 19 peerDependencies 兼容矩阵，重点查 UI 组件库、可视化库（图表库升级常是大坑）、编辑器类库；React 19 移除了函数组件 defaultProps 支持、移除了 ReactDOM.render，用废弃 API 的库会直接崩。一次升级还覆盖：类型（React 19 types 差异）、测试（Testing Library 的 act 适配）、构建（render → createRoot 迁移）。

分阶段路径：第一步锁 18——createRoot + 并发 + 自动批处理，风险最低收益直接，且 17 与 18 混用不可行，必须一次切；第二步在 18 上过渡半年，验证第三方库兼容、建立 INP/LCP 基线、让团队适应新心智；第三步切 19，重点验证 Actions/use 是否替换现有手写表单逻辑、React Compiler 能否在该代码库启用（babel 插件 + lint 规则 + 性能回归）。每步都要量化验收：INP 变化、包体积、memo 代码删除量、回归测试全绿，并预留回滚方案（大版本降级意味着同步回退全部新特性代码，所以要小步、可观测）。

对渲染架构的理解要能串成主线：React 的演进是把"控制权从开发者交给运行时"——Fiber 解决"大组件树不可中断"（v16），并发渲染解决"更新优先级调度"（v18），Server Components 把"渲染位置从客户端移到服务端"（v19/RSC），React Compiler 把"记忆化优化从手写交给编译期"。你不必喜欢每个特性，但要说得出取舍：不用 RSC 的团队完全可以把 19 只当"性能更好的 18"用。开放性总结：升级价值 = 解决的问题 - 迁移成本 - 未知风险，数字算得过来就上，算不过来就停在 18 等生态成熟。

**延伸考点：** React Compiler 启用后手写 useMemo/useCallback 还需要吗？什么代码模式会让 Compiler 失效？RSC 与传统 SSR 模板渲染的本质区别是什么？
