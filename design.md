# 设计系统：token、落地方式与交互规则

适用：小程序和 Web 项目里写页面、组件、样式，以及处理加载、空、错误状态和操作反馈时。

本文件只规定**结构和规则**：token 有哪些、叫什么、在各端写在哪里、怎么被组件库使用，以及交互该怎么反馈。下面给出的数值是**默认值**，每个项目都可以覆盖。好不好看属于审美判断，交给 `design-taste-frontend` 去做；截图验收交给 `visual-render-checker`。

**取值优先级：设计稿 > 项目已有的 token / 主题 > 本文件的默认值。** 现有项目已经有自己的主题变量或组件库时，沿用它，不要迁移到本文件的命名（见 `SKILL.md`「第零步：先看现状」）。

## 核心规则

- 业务代码里**只用语义 token**，不写具体色值：不写 `#fff`、`rgb()`，Web 不写 `bg-white`、`text-gray-500` 这类调色板类名。色值只出现在 token 文件里。
- 覆盖默认值 = 改项目自己的 token 文件，不改组件库源码，也不在业务组件里逐个覆盖组件样式。
- 需要新 token 时（比如一个新的业务状态色），在 token 文件里按同样的命名方式加，不在页面里临时写死。
- **暗色模式：结构预留，默认不做。** 因为只用语义 token，以后加暗色只需要给同一组变量补一套值。没有用户要求时，不加主题切换，不开启小程序的 `darkmode` 配置，也不用 wot 的 `theme="dark"`。

## Token 定义

两端用同一组语义名称。Web 直接沿用 shadcn/ui 的变量名，小程序用同名 CSS 变量，在两端之间搬代码时不用改名。

### 颜色

| token | 用途 | 默认值 |
|---|---|---|
| `--primary` | 品牌色：主按钮、选中态、链接 | `#1C64FD` |
| `--primary-foreground` | 主色背景上的文字 | `#FFFFFF` |
| `--background` | 页面底色 | `#F7F8FA` |
| `--foreground` | 主要文字 | `#1D1F29` |
| `--card` | 卡片、列表项、弹层的底色 | `#FFFFFF` |
| `--muted` | 弱化区块的底色（标签、分组背景） | `#F2F3F5` |
| `--muted-foreground` | 次要文字、说明、时间 | `#666B7F` |
| `--border` | 分割线、卡片边界（只作装饰） | `#E5E6EB` |
| `--input` | 输入框、复选框等控件的边框 | `#868A9C` |
| `--destructive` | 危险操作、错误提示 | `#D42B2B` |
| `--success` | 成功状态的文字和图标 | `#0B6F51` |
| `--warning` | 警告状态的文字和图标 | `#A94605` |

- 默认值核对过的搭配（覆盖颜色后要按同样的搭配重新检查，见「可访问性」）：
  - 文字色（`--foreground`、`--muted-foreground`、`--primary`、`--destructive`、`--success`、`--warning`）在 `--card` 和 `--background` 上 ≥ 4.5:1。
  - `--muted` 底上：除了 `--primary`（4.41:1），其余文字色都 ≥ 4.5:1。所以**主色文字不放在 `--muted` 上**，选中的标签用 `--primary` 底配 `--primary-foreground` 字。
  - 白字放在 `--primary`、`--destructive`、`--success`、`--warning` 上，以及小程序里它们的按下态上，都 ≥ 4.5:1。
  - `--input` 在 `--card`、`--background` 上 ≥ 3:1（控件边界的要求，见「可访问性」）。`--border` 只有 1.25:1，只能用于装饰性的分割线和卡片边界，不能作为控件唯一的边界。
- 默认主色和 wot 的默认主色是同一个蓝，没改品牌色时两端不需要额外处理组件库色阶。

### 字号

以 375 宽设计稿的 px 为准；小程序乘 2 换成 rpx（设计稿 750）。

| 档位 | Web（Tailwind 默认） | 小程序 token | 用途 |
|---|---|---|---|
| xs | `text-xs` 12px | `--text-xs: 24rpx` | 辅助说明、角标 |
| sm | `text-sm` 14px | `--text-sm: 28rpx` | **正文默认** |
| base | `text-base` 16px | `--text-base: 32rpx` | 强调正文、Web 输入框 |
| lg | `text-lg` 18px | `--text-lg: 36rpx` | 卡片标题 |
| xl | `text-xl` 20px | `--text-xl: 40rpx` | 页面标题 |
| 2xl | `text-2xl` 24px | `--text-2xl: 48rpx` | 数字、大标题 |

- 一个页面用到的字号不超过 4 档。层级靠字号、字重、颜色（`--foreground` / `--muted-foreground`）拉开，不要每处都换字号。
- 行高：Web 用 Tailwind 字号类自带的行高，不另外设置；小程序正文 1.5，标题 1.3。
- Web 表单里输入框用 `text-base`（原因见 `web/guide.md`），其余正文用 `text-sm`。

### 间距

- 所有间距用 4px 的倍数（小程序 8rpx 的倍数）。Web 用 Tailwind 的整数刻度（`1` `2` `3` `4` `5` `6` `8` `10` `12` …，如 `p-4` = 16px），不用 `0.5` `1.5` `2.5` `3.5` 这类半档，也不写任意值。
- 三个固定的布局间距：

| 用途 | 默认值 | Web | 小程序 token |
|---|---|---|---|
| 页面左右边距 | 16px | `px-4` | `--page-gutter: 32rpx` |
| 卡片内边距 | 16px | `p-4` | `--card-padding: 32rpx` |
| 卡片之间、区块之间 | 12px | `gap-3` / `space-y-3` | `--section-gap: 24rpx` |

### 圆角

- Web：只改 shadcn 的基础变量 `--radius`（shadcn 生成的是 `0.625rem`，本规范默认改成 `0.5rem`，即 8px），`rounded-sm` 到 `rounded-4xl` 都由它推导，不单独改某一档。
- 小程序：

| token | 默认值 | 用途 |
|---|---|---|
| `--radius-sm` | `8rpx` | 标签、角标 |
| `--radius-md` | `16rpx` | 卡片、输入框、自定义按钮 |
| `--radius-lg` | `24rpx` | 弹窗、底部面板 |

- wot 组件的圆角用它自己的默认值，不覆盖（wot 的尺寸变量是 px，和本表的 rpx 体系不同）。

### 阴影与层级

- 平铺的卡片靠 `--card` 和 `--background` 的底色差、或者 `--border` 区分，不加阴影。阴影只给浮在页面上的东西（悬浮按钮、下拉菜单）。Web 用 Tailwind 的 `shadow-md`；小程序用 `--shadow-float: 0 8rpx 24rpx rgba(0, 0, 0, 0.08)`。
- 弹窗、toast、下拉层用组件库的组件，层级由组件库管理。业务自己写的层级只用三档：吸顶 `10`、底部固定栏和悬浮按钮 `20`、自定义浮层 `50`。不写 `9999`。

### 动效

- 过渡时长 150–250ms，用 ease-out。只给状态变化（展开、出现、切换）加动效，不给首屏内容加入场动画。
- Web：有动效的地方加 `motion-reduce:` 关闭或减弱。

## Web 落地（Next.js + Tailwind v4 + shadcn/ui）

- 新项目用 `npx shadcn@latest init` 初始化，token 全部写在它生成的 `app/globals.css` 里（用了 `src/` 目录的项目是 `src/app/globals.css`）。现有项目没有用 shadcn 时，不为了本规范引入。
- 把 `:root` 里对应变量的值换成上面颜色表的默认值；shadcn 生成的其他变量（`secondary`、`accent`、`popover`、`input`、`ring`、`chart-*`、`sidebar-*`）保留，`input` 取颜色表里 `--input` 的值，`ring` 取 `--primary` 的值。
- shadcn 默认没有的 `--success`、`--warning`，按 shadcn 的自定义 token 方式补上：

  ```css
  :root {
    --success: #0B6F51;
    --warning: #A94605;
  }

  @theme inline {
    --color-success: var(--success);
    --color-warning: var(--warning);
  }
  ```

  之后就能写 `text-success`、`text-warning`。
- 生成的 `.dark` 块原样保留但不启用（不给 `<html>` 加 `dark` 类，不装主题切换），留给以后做暗色模式。
- 用法：`bg-background`、`bg-card`、`text-foreground`、`text-muted-foreground`、`border-border`（分割线、卡片）、`border-input`（自己写的输入类控件的边框）、`bg-primary text-primary-foreground`、`text-destructive`。
- 图标用 shadcn 默认的 `lucide-react`，颜色跟随文字（`currentColor`），不单独给图标写色值。
- `components/ui/` 下是 shadcn 生成的组件代码，改主题只改 `globals.css`；要改组件行为时可以改这里的代码，但不要在里面写死颜色。

## 小程序落地（uni-app + wot UI）

- 新项目用 wot UI v2，npm 包名是 **`@wot-ui/ui`**（旧包 `wot-design-uni` 停在 1.x，已经在用它的项目沿用，不主动升级）。按官方文档在 `pages.json` 的 `easycom.custom` 里加 `"^wd-(.*)": "@wot-ui/ui/components/wd-$1/wd-$1.vue"`，组件免注册。
- token 写在 `src/styles/tokens.scss`，在 `App.vue` 的全局样式里引入。如果项目还引入了 wot 的主题文件（`@wot-ui/ui/styles/theme/index.scss`），tokens 要放在它**后面**引入，否则会被覆盖。

  ```scss
  // src/styles/tokens.scss
  page,
  .wd-root-portal {
    --primary: #1C64FD;
    --primary-foreground: #FFFFFF;
    --background: #F7F8FA;
    --foreground: #1D1F29;
    --card: #FFFFFF;
    --muted: #F2F3F5;
    --muted-foreground: #666B7F;
    --border: #E5E6EB;
    --input: #868A9C;
    --destructive: #D42B2B;
    --success: #0B6F51;
    --warning: #A94605;

    --text-xs: 24rpx;
    --text-sm: 28rpx;
    --text-base: 32rpx;
    --text-lg: 36rpx;
    --text-xl: 40rpx;
    --text-2xl: 48rpx;

    --page-gutter: 32rpx;
    --card-padding: 32rpx;
    --section-gap: 24rpx;

    --radius-sm: 8rpx;
    --radius-md: 16rpx;
    --radius-lg: 24rpx;

    --shadow-float: 0 8rpx 24rpx rgba(0, 0, 0, 0.08);

    // 让 wot 组件跟着同一组 token 走（只映射颜色）
    // 主色：默认值就是 wot 的默认蓝，不用映射；改品牌色见下文
    // 状态色：常态和悬停都用 token 本身，按下态用 wot 同色系深一档
    // disabled / particular / surface（浅色底）沿用 wot 默认
    --wot-danger-main: var(--destructive);
    --wot-danger-hover: var(--destructive);
    --wot-danger-clicked: #BC2626; // wot red-8
    --wot-success-main: var(--success);
    --wot-success-hover: var(--success);
    --wot-success-clicked: #074A36; // wot green-9
    --wot-warning-main: var(--warning);
    --wot-warning-hover: var(--warning);
    --wot-warning-clicked: #813604; // wot orange-9
    // 控件边界：未选中的复选框、单选框和关闭的开关轨道，wot 默认是 #E5E6EB（1.25:1），改用 --input
    --wot-checkbox-unchecked-color: var(--input);
    --wot-radio-unchecked-color: var(--input);
    --wot-switch-color-inactive-bg: var(--input);
    --wot-switch-color-inactive-icon: var(--foreground);
    --wot-switch-color-inner-inactive-text: var(--foreground);
    --wot-text-main: var(--foreground);
    --wot-text-auxiliary: var(--muted-foreground);
    --wot-border-main: var(--border);
    --wot-filled-bottom: var(--background);
  }

  page {
    background-color: var(--background);
    color: var(--foreground);
    font-size: var(--text-sm);
  }
  ```

- 改了 `--destructive`、`--success`、`--warning` 的，对应的 `-clicked` 也要换成新颜色深一档的值，并检查白字对比度。
- wot 的表单行（`wd-form-item`、`wd-cell`）底线走 `--wot-border-main`，也就是 `--border`，按装饰处理，不改成 `--input`。原因：`wd-input` 本身没有边框，一行输入框是靠左侧的标题和整行布局识别的，不是靠这条线。所以前提是**每个输入行都有可见的标题**（`wd-form-item` / `wd-cell` 的 `title`），不能只靠 placeholder 提示要填什么。
- wot 组件内部用浅色底配状态色文字的地方（浅色标签、提示条，用 `-particular` / `-surface`），沿用 wot 默认，没有逐个核对对比度。用到这类组件时要自己检查。
- **改品牌色时**：wot 的主色是 `--wot-primary-1` 到 `--wot-primary-10` 一整套色阶（浅色背景、按下态都用到别的档位），只改 `-6` 会出现按下态、浅色底仍是默认蓝。所以 10 档都要覆盖，并让 `--primary` 等于第 6 档。色阶从设计稿或色板工具拿。下面用 wot 自带的 grape 色板举例，组件只带了 SCSS 兜底值，没有引入 wot 主题文件时 `--wot-grape-*` 变量不存在，所以写色值：

  ```scss
  page,
  .wd-root-portal {
    --primary: #AE3EC9;
    --wot-primary-1: #FBF6FD;
    --wot-primary-2: #F3D9FA;
    --wot-primary-3: #EEBEFA;
    --wot-primary-4: #E599F7;
    --wot-primary-5: #DA77F2;
    --wot-primary-6: var(--primary);
    --wot-primary-7: #9731AF;
    --wot-primary-8: #7B288F;
    --wot-primary-9: #601F70;
    --wot-primary-10: #451650;
  }
  ```

  换完检查：`--primary` 在 `--card`、`--background` 上和白字在 `--primary`、第 7 档上都 ≥ 4.5:1；再同步改 Web 的 `--primary` 和 `pages.json`（见下一条）。
- 页面导航栏和 tabBar 的颜色在 `pages.json` 里配置，不能用 CSS 变量，改品牌色时要同步改这里。
- 业务样式里用 `var(--xxx)`，不写色值。`rpx` 的规则见 `miniprogram/guide.md`。
- 图标：优先用 wot 的 `wd-icon`；自定义图标用 SVG 文件放 `static/`，通过 `<image>` 引用。注意 `<image>` 里的 SVG 不能用 CSS 改颜色，需要多种颜色就准备多个文件。

## 组件化

目标：同一种 UI 模式，整个项目只有一个实现、一种写法。新页面是用已有组件拼出来的，而不是每个页面各写一遍样式。

### 分层

| 层 | 放什么 | Web | 小程序 |
|---|---|---|---|
| 组件库 | 按钮、输入框、弹窗、骨架屏、空状态、卡片、表单项等基础控件 | `components/ui/`（shadcn 生成） | `wd-*`（`@wot-ui/ui`） |
| 通用组件 | 项目自己的统一模式，和具体业务无关（见下方「必备通用组件」） | `components/common/` | `components/组件名/组件名.vue` |
| 业务组件 | 某个业务模块的展示单元，如问卷卡片 | `components/<模块>/` | 多处用到的放 `components/`；只在一个分包用的放该分包的 `components/`，easycom 不扫描分包目录，要手动 import |
| 页面 | 取数据、组合组件、处理跳转 | `app/**/page.tsx` | `pages/`、`subpkg-*/` |

只能上层引用下层：页面引用业务组件和通用组件，业务组件引用通用组件和组件库，反过来不行。

### 规则

- **组件库有的不自己写。** 按钮、输入框、开关、弹窗、骨架屏、空状态、卡片都用组件库的；组件库没有的，才写通用组件。
- **页面里不写「块」的外观。** 页面文件只写布局（flex、grid、gap、宽度、对齐）和文字层级（Web 可以用 `text-sm text-muted-foreground` 这类 token 类名）。背景、边框、圆角、阴影只出现在组件里：页面上需要一个有背景、有圆角的块，说明该用（或该抽）一个组件。
- **第二次出现就抽。** 写新 UI 前先在 `components/` 里找有没有能用的。同一个结构第二次出现时抽成组件，并把第一次那处也换掉，不要复制过来再改。
- **组件接口用语义，不传样式值。**
  - 外观差异用有限的枚举 props 表达（`variant: 'default' | 'danger'`、`size: 'sm' | 'md'`），不接收颜色、圆角、字号这类具体值。需要新外观时加一个 variant，不在使用的地方覆盖样式。
  - Web 组件可以接收 `className`，只用来加外部布局（外边距、宽度、grid 位置），不用来改组件内部外观。小程序同理，不从外面用 `:deep()` 改通用组件的内部样式。
  - 通用组件不请求接口、不读 store，数据和回调都由页面传进来，这样同一个组件在哪个页面都能用。
- 现有项目：先找项目里已有的同类组件沿用。没有分层的项目不重构目录，只在新写的代码里按这套来；发现重复的写法，在回复里列出来建议抽取，由用户决定。

### 必备通用组件

新项目开工时先建这几个，两端同名、职责相同。页面状态和操作反馈的规则（见「交互规则」）都靠它们落实，页面不要自己再写一遍。

| 组件 | 职责 | 基于 |
|---|---|---|
| `PageContainer` | 页面底色、左右边距（`--page-gutter`）、底部安全区；Web 还负责内容区最大宽度和居中 | — |
| `EmptyState` | 统一的空状态：一句说明 + 可选的操作按钮 | Web `Empty`，小程序 `wd-empty` |
| `ErrorState` | 统一的加载失败：通用错误文案 + 「重试」按钮 | 组件库按钮 |
| `StateView` | 传入 `loading`、`error`、`empty` 和重试回调，按「页面状态」表显示骨架屏、`ErrorState`、`EmptyState`，正常时渲染内容 | 上面两个 + 骨架屏 |
| `BottomBar` | 底部固定操作栏：处理安全区，并在页面底部留出同样高度，内容不会被挡住 | — |
| `usePagedList` | 列表分页的 hook（Web）/ composable（小程序）：支持 `SKILL.md` 里的页码和游标两种分页，管理加载更多、到底、失败重试、刷新 | — |
| 反馈封装 | toast 和危险操作确认的唯一入口，固定「确认按钮写动作、用危险色」 | 见下方 |

- **骨架屏跟着业务组件走**：列表项、详情这类业务组件同时提供自己的骨架（如 `SurveyCard` 配 `SurveyCardSkeleton`），传给 `StateView` 的加载插槽，形状才能接近真实内容。
- **Web 的首屏状态**：Server Component 取数据的页面，首次加载由同目录的 `loading.tsx` 负责（渲染对应的骨架组件），出错由 `error.tsx` 负责（渲染 `ErrorState`，重试调用 Next 传进来的函数：16.3 起是 `retry()`，16.2 是 `unstable_retry()`，16.1 及以前只有 `reset()`，按项目的 Next 版本选）。`StateView` 用在 client 端取数据的地方，以及页面内的局部区块。
- **反馈封装**：
  - Web：toast 直接用 sonner 的 `toast()`（`<Toaster />` 放在根 `layout.tsx`）；危险操作用 `components/common/ConfirmDialog.tsx`，基于 `AlertDialog`，确认按钮固定用 `destructive` 样式。
  - 小程序：`utils/feedback.ts` 导出 `toast()` 和 `confirmDanger()`，内部用 `uni.showToast`（`icon: 'none'`）和 `uni.showModal`（`confirmColor` 用危险色）。不用 wot 的 `useToast` / `useDialog`：它们依赖组件树里的 provide/inject，只能在组件 setup 里调用，请求封装这种非组件代码用不了，混用会出现两种样式的提示。
  - `uni.showModal` 的 `confirmColor` 只接受色值，不能用 CSS 变量，所以 `feedback.ts` 里要写一份危险色，改 `--destructive` 时同步改这里。

## 交互规则

两端都适用。Web 的 PC 端规则（键盘、hover、宽屏）另见 `web/guide.md`「交互底线」。

### 页面状态

每个需要请求数据的页面或列表，都要有下面四种状态，缺一种都不算完成。用 `StateView`（Web 首屏用 `loading.tsx` / `error.tsx`）实现，不在页面里手写判断和样式：

| 状态 | 做法 |
|---|---|
| 首次加载 | 列表、详情用骨架屏（Web `Skeleton`，小程序 `wd-skeleton`），形状接近真实内容；表单页、很短的请求可以用居中的 loading。不要整页白屏等数据。 |
| 空 | 一句话说明为什么是空的，如果用户能做什么（去创建、换个筛选条件），给一个按钮（Web `Empty`，小程序 `wd-empty`）。 |
| 出错 | 首屏加载失败（`50000`、网络错误、超时）显示错误态和「重试」按钮，点了只重新请求，不刷新整页。错误码分流见 `SKILL.md`「响应与错误」。 |
| 正常 | — |

- 刷新、翻页、筛选时，已经显示的内容不要清空换成骨架屏，在原有内容上显示加载提示。
- 下拉刷新或加载更多失败，保留已有数据，用 toast 提示，列表底部给「点击重试」。
- 列表全部加载完显示「没有更多了」，不要让用户以为还在加载。

### 操作反馈

| 场景 | 用什么 |
|---|---|
| 操作成功、不需要用户再做什么 | toast，一句短话（如「已保存」），自动消失 |
| 表单字段校验不通过 | 字段下方的红字（`--destructive`），说明怎么改；不用 toast |
| 接口返回业务错误（非 0 的 `code`，显示 `message`） | toast；如果错误让当前页面无法继续（如无权限），显示为页面状态 |
| 删除、退出、放弃编辑等不可逆操作 | 确认弹窗：标题写后果（「删除后无法恢复」），确认按钮写具体动作（「删除」，不写「确定」），用危险色 |
| 需要用户知道但不打断操作的结果 | toast，不用需要点「知道了」的弹窗 |

- 一律通过「反馈封装」调用（见「组件化」），一个项目只有一套 toast 和确认弹窗。
- 小程序 `uni.showToast` 带图标（`success`、`error`）时最多显示 7 个汉字，文案长的用 `icon: 'none'`；`uni.showModal` 的确认按钮文字最多 4 个字。
- 成功之后要离开页面的，先 toast 再跳转，跳转前留出能看清提示的时间（约 1 秒），或者把结果带到下一个页面上显示。

### 按钮与表单

- 一个页面（或一个弹窗）只有一个主按钮（`--primary` 实心），其余用次级样式。
- 按钮文案用动词：「提交」「保存」「删除」，不写「确定」「OK」。
- 每个异步操作都要有加载态，提交按钮请求期间显示 loading 并禁用，防止重复提交。
- 提交时校验全部字段，第一个出错的字段滚动到可见区域；已经出错的字段在修改后（失焦时）重新校验。
- 提交失败不清空用户已经填的内容。

### 可访问性

- 可点击区域不小于 44×44px（小程序 88×88rpx）。图标按钮看起来小，点击区域也要用 padding 撑够。
- 文字和背景的对比度 ≥ 4.5:1；大字（24px 以上，或 19px 以上的粗体）≥ 3:1。覆盖颜色 token 后用对比度工具检查一遍。
- 控件的边界（输入框边框、复选框外框、开关轨道）和焦点环，与相邻背景的对比度 ≥ 3:1。所以输入框用 `--input`，不用 `--border`；焦点环用 `--primary`。
- 状态不能只靠颜色表达：错误除了变红还要有文字说明，选中除了变色还要有勾或者底色变化。
- 禁用状态要能看出是禁用的，同时文字仍然看得清。

### 文案

- 中文界面用中文全角标点。
- 错误提示写原因和用户能做什么（「网络不太稳定，请重试」），不写技术信息（见 `SKILL.md`「安全」）。
- 同一个动作在整个项目里用同一个词（不要这里叫「删除」那里叫「移除」）。

## 验收命令

在对应项目根目录执行，并在回复里贴出输出。命令扫描整个项目，TS 和 JS 文件都包含；排除依赖、构建产物（`.next`、`out`、`build`、`dist`、`unpackage`）、静态资源目录（`public`、`static`）和组件库生成的代码，所以 `app/` 和 `src/app/` 两种结构都能覆盖。项目的构建输出在别的目录时，自己补一个 `--exclude-dir`。命中的逐条确认：是 token 文件、或者确实必须写死的（如第三方要求的颜色），才可以保留。没有输出表示没有命中。

```bash
# Web：业务代码里写死的色值和调色板类名
grep -rnE "#[0-9a-fA-F]{3,8}\b|rgba?\(|\b(bg|text|border(-[xytrblse])?|ring|fill|stroke|from|via|to|divide|outline|decoration|shadow|accent|caret|placeholder)-(white|black|(slate|gray|zinc|neutral|stone|red|orange|amber|yellow|lime|green|emerald|teal|cyan|sky|blue|indigo|violet|purple|fuchsia|pink|rose)-[0-9]{2,3})\b" . --include='*.tsx' --include='*.ts' --include='*.jsx' --include='*.js' --include='*.css' --include='*.scss' --exclude-dir=node_modules --exclude-dir=.next --exclude-dir=out --exclude-dir=build --exclude-dir=dist --exclude-dir=public | grep -v "app/globals.css" | grep -v "components/ui/"

# 小程序：业务代码里写死的色值
grep -rnE "#[0-9a-fA-F]{3,8}\b|rgba?\(" . --include='*.vue' --include='*.scss' --include='*.css' --include='*.ts' --include='*.js' --exclude-dir=node_modules --exclude-dir=uni_modules --exclude-dir=unpackage --exclude-dir=dist --exclude-dir=static | grep -v "styles/tokens.scss" | grep -vE "utils/feedback\.(ts|js)"
```

`#` 开头的命中也可能是锚点链接之类的非颜色内容，逐条看。

组件化检查（命中的逐条确认，能换成组件的要换）：

```bash
# Web：页面层（app/ 或 src/app/ 下）直接写块外观的地方
grep -rnE "className=\"[^\"]*\b(bg-|rounded|shadow|border)" . --include='*.tsx' --include='*.jsx' --exclude-dir=node_modules --exclude-dir=.next --exclude-dir=out --exclude-dir=build --exclude-dir=dist --exclude-dir=public | grep -E "^\./(src/)?app/"
# Web：绕过反馈封装的原生弹窗，应为空
grep -rnE "\b(window\.)?(alert|confirm)\(" . --include='*.tsx' --include='*.ts' --include='*.jsx' --include='*.js' --exclude-dir=node_modules --exclude-dir=.next --exclude-dir=out --exclude-dir=build --exclude-dir=dist --exclude-dir=public | grep -v "components/ui/"

# 小程序：页面层直接写块外观的地方
grep -rnE "(background|border|border-radius|box-shadow)[a-z-]*\s*:" . --include='*.vue' --exclude-dir=node_modules --exclude-dir=uni_modules --exclude-dir=unpackage --exclude-dir=dist --exclude-dir=static | grep -E "^\./(src/)?(pages|subpkg-[^/]+)/" | grep -v "/components/"
# 小程序：绕过反馈封装直接调用的，应为空
grep -rnE "uni\.(showToast|showModal)|useToast|useDialog" . --include='*.vue' --include='*.ts' --include='*.js' --exclude-dir=node_modules --exclude-dir=uni_modules --exclude-dir=unpackage --exclude-dir=dist --exclude-dir=static | grep -vE "utils/feedback\.(ts|js)"
```

命令无法覆盖、需要在回复里提醒用户确认的：
- 每个数据页面的加载、空、出错三种状态都实际看过（可以临时改接口返回或断网来触发）
- 新写的 UI 先查过有没有可用的已有组件；同一个结构出现两次以上的，已经抽成组件
- 改过颜色 token 的，文字对比度检查过
- 小程序改过品牌色的，`pages.json` 里导航栏和 tabBar 的颜色同步改了，wot 的主色色阶 10 档都覆盖了
