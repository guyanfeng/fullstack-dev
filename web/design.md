# Web：设计系统落地（Next.js + Tailwind v4 + shadcn/ui）

token 的定义、组件化和交互规则见 `design.md`，本文件只写 Web 端怎么落地，以及 Web 端的验收命令。

## 落地

- 新项目用 `npx shadcn@latest init` 初始化，token 全部写在它生成的 `app/globals.css` 里（用了 `src/` 目录的项目是 `src/app/globals.css`）。现有项目没有用 shadcn 时，不为了本规范引入。
- 把 `:root` 里对应变量的值换成 `design.md` 颜色表的默认值；shadcn 生成的其他变量（`secondary`、`accent`、`popover`、`input`、`ring`、`chart-*`、`sidebar-*`）保留，`input` 取颜色表里 `--input` 的值，`ring` 取 `--primary` 的值。
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

## 验收命令

在 Web 项目根目录执行，并在回复里贴出输出。命令扫描整个项目，TS 和 JS 文件都包含；排除依赖、构建产物（`.next`、`out`、`build`、`dist`）、静态资源目录（`public`）和 shadcn 生成的 `components/ui/`，所以 `app/` 和 `src/app/` 两种结构都能覆盖。构建输出在别的目录时，自己补一个 `--exclude-dir`。命中的逐条确认：是 token 文件、或者确实必须写死的（如第三方要求的颜色），才可以保留；能换成组件的要换。没有输出表示没有命中。

```bash
# Web：业务代码里写死的色值和调色板类名
grep -rnE "#[0-9a-fA-F]{3,8}\b|rgba?\(|\b(bg|text|border(-[xytrblse])?|ring|fill|stroke|from|via|to|divide|outline|decoration|shadow|accent|caret|placeholder)-(white|black|(slate|gray|zinc|neutral|stone|red|orange|amber|yellow|lime|green|emerald|teal|cyan|sky|blue|indigo|violet|purple|fuchsia|pink|rose)-[0-9]{2,3})\b" . --include='*.tsx' --include='*.ts' --include='*.jsx' --include='*.js' --include='*.css' --include='*.scss' --exclude-dir=node_modules --exclude-dir=.next --exclude-dir=out --exclude-dir=build --exclude-dir=dist --exclude-dir=public | grep -v "app/globals.css" | grep -v "components/ui/"

# Web：页面层（app/ 或 src/app/ 下）直接写块外观的地方
grep -rnE "className=\"[^\"]*\b(bg-|rounded|shadow|border)" . --include='*.tsx' --include='*.jsx' --exclude-dir=node_modules --exclude-dir=.next --exclude-dir=out --exclude-dir=build --exclude-dir=dist --exclude-dir=public | grep -E "^\./(src/)?app/"
# Web：绕过反馈封装的原生弹窗，应为空
grep -rnE "\b(window\.)?(alert|confirm)\(" . --include='*.tsx' --include='*.ts' --include='*.jsx' --include='*.js' --exclude-dir=node_modules --exclude-dir=.next --exclude-dir=out --exclude-dir=build --exclude-dir=dist --exclude-dir=public | grep -v "components/ui/"
```

`#` 开头的命中也可能是锚点链接之类的非颜色内容，逐条看。

命令无法覆盖的检查见 `design.md`「验收」。
