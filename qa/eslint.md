# ESLint

## 1.安装依赖

```shell
pnpm add -D eslint @eslint/js eslint-plugin-vue eslint-plugin-import eslint-plugin-prettier eslint-config-prettier prettier
```

说明：

- `eslint`：核心
- `@eslint/js`：ESLint 官方推荐规则
- `eslint-plugin-vue`：Vue3 的必备规则
- `eslint-plugin-import`：优化 import/export 语法
- `eslint-plugin-prettier` + `eslint-config-prettier`：解决 ESLint 和 Prettier 的冲突
- `prettier`：格式化工具（推荐和 ESLint 结合使用）

## 2.Prettier 配置

**根目录下新建`.prettierrc.json`**

```json
{
    "semi": false, // 行尾不加分号（更简洁，Vue/React 社区常用）
    "singleQuote": true, // 使用单引号，避免和 JSX 属性双引号冲突
    "printWidth": 100, // 每行最大长度，100 比 80 更宽容，团队常用
    "tabWidth": 2, // 缩进两个空格，几乎所有前端团队默认
    "useTabs": false, // 不用 tab，统一空格缩进
    "trailingComma": "es5", // ES5 允许的地方加尾逗号（对象/数组），函数参数不加
    "bracketSpacing": true, // 对象大括号内加空格 → { foo: bar }
    "arrowParens": "always", // 箭头函数参数总是加括号 → (x) => x
    "endOfLine": 'lf', // 强制 LF 换行符（跨平台一致）
    "htmlWhitespaceSensitivity": "ignore", // HTML 中空格敏感度，避免影响 Vue/SVG 格式
    "vueIndentScriptAndStyle": false, // Vue SFC 中 script/style 不额外缩进
}
```

## 3.ESlint配置文件

**根目录下新建`eslint.config.js`**

```javascript
// eslint.config.js
import js from '@eslint/js'
import pluginVue from 'eslint-plugin-vue'
import prettier from 'eslint-plugin-prettier/recommended'

export default [
  // 忽略目录
  {
    ignores: ['dist/**', 'node_modules/**'],
  },

  // 直接引入推荐规则（注意：这里是顶层，不要用 extends）
  js.configs.recommended,
  ...pluginVue.configs['flat/recommended'],
  prettier,

  // 你的自定义规则
  {
    files: ['**/*.js', '**/*.vue'],
    languageOptions: {
      ecmaVersion: 'latest',
      sourceType: 'module',
      globals: {
        process: 'readonly',
        __dirname: 'readonly',
        module: 'readonly',
        require: 'readonly',
      },
    },
    rules: {
      // JS 规则
      // 'no-console': import.meta.env.PROD ? 'warn' : 'off',
      // 'no-debugger': import.meta.env.PROD ? 'warn' : 'off',

      // Vue 规则
      'vue/multi-word-component-names': 'off',
      'vue/no-mutating-props': 'warn',
      'vue/require-default-prop': 'off',
    },
  },
]

```

## 4.在 `package.json` 添加脚本

```json
{
  "scripts": {
    "dev:lint": "npm run lint && vite", // 启动前先进行eslint检查
    "lint": "eslint . --ext .js,.vue",
    "lint:fix": "eslint . --ext .js,.vue --fix"
  }
}
```

