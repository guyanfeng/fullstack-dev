# 小程序：设计系统落地（uni-app + wot UI）

token 的定义、组件化和交互规则见 `design.md`，本文件只写小程序端怎么落地，以及小程序端的验收命令。

## 落地

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

## 验收命令

在小程序项目根目录执行，并在回复里贴出输出。命令扫描整个项目，TS 和 JS 文件都包含；排除依赖（`node_modules`、`uni_modules`）、构建产物（`unpackage`、`dist`）和静态资源目录（`static`），uni-app CLI 的 `src/` 结构和 HBuilderX 的根目录结构都能覆盖。命中的逐条确认：是 token 文件、反馈封装，或者确实必须写死的，才可以保留；能换成组件的要换。没有输出表示没有命中。

```bash
# 小程序：业务代码里写死的色值
grep -rnE "#[0-9a-fA-F]{3,8}\b|rgba?\(" . --include='*.vue' --include='*.scss' --include='*.css' --include='*.ts' --include='*.js' --exclude-dir=node_modules --exclude-dir=uni_modules --exclude-dir=unpackage --exclude-dir=dist --exclude-dir=static | grep -v "styles/tokens.scss" | grep -vE "utils/feedback\.(ts|js)"

# 小程序：页面层直接写块外观的地方
grep -rnE "(background|border|border-radius|box-shadow)[a-z-]*\s*:" . --include='*.vue' --exclude-dir=node_modules --exclude-dir=uni_modules --exclude-dir=unpackage --exclude-dir=dist --exclude-dir=static | grep -E "^\./(src/)?(pages|subpkg-[^/]+)/" | grep -v "/components/"
# 小程序：绕过反馈封装直接调用的，应为空
grep -rnE "uni\.(showToast|showModal)|useToast|useDialog" . --include='*.vue' --include='*.ts' --include='*.js' --exclude-dir=node_modules --exclude-dir=uni_modules --exclude-dir=unpackage --exclude-dir=dist --exclude-dir=static | grep -vE "utils/feedback\.(ts|js)"
```

`#` 开头的命中也可能是锚点链接之类的非颜色内容，逐条看。

命令无法覆盖、需要在回复里提醒用户确认的（另见 `design.md`「验收」）：
- 小程序改过品牌色的，`pages.json` 里导航栏和 tabBar 的颜色同步改了，wot 的主色色阶 10 档都覆盖了
