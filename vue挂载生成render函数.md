如果当前是template 模板或者document.selectorQuery(el) 获取的html节点。

入口 ：  src\platforms\web\entry-runtime-with-compiler.js

*Vue*.prototype.$mount 中：

```
      const { render, staticRenderFns } = compileToFunctions(template, {
        outputSourceRange: process.env.NODE_ENV !== 'production',
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
```

compileToFunctions 函数

入口 ：  src\platforms\web\compiler\index.js

```
import { baseOptions } from './options'
import { createCompiler } from 'compiler/index'

const { compile, compileToFunctions } = createCompiler(baseOptions)

export { compile, compileToFunctions }
```

createCompiler 函数   ： 

入口 ：src\compiler\index.js

```
export const createCompiler = createCompilerCreator(function baseCompile (
  template: string,
  options: CompilerOptions
): CompiledResult {
  const ast = parse(template.trim(), options)
  if (options.optimize !== false) {
    optimize(ast, options)
  }
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
})

```

 createCompiler  返回的是函数，返回的这个函数的值为如下格式

原理： 将传入的html,  转换成ast,   generate再转换一下格式生成如下格式

```
code = {
render: "with(this){return _c('div',{attrs:{"id":"app"}},[_v("\n        "+_s(msg)+"\n        "),_c('button',{on:{"click":function($event){return changeMsg()}}},[_v("改变值")])])}",
staticRenderFns: []
}

```



createCompilerCreator 函数 （高阶函数，函数柯理化，将函数的参数分多次传入，跨平台）

入口 ： src\compiler\create-compiler.js

描述：传入一个baseCompile函数返回一个函数createCompiler， 延迟执行	baseCompile

createCompiler 函数返回一个 compile ， compile  执行 baseCompile

```javascript
export function createCompilerCreator (baseCompile: Function): Function {
  return function createCompiler (baseOptions: CompilerOptions) {
    function compile (
      template: string,
      options?: CompilerOptions
    ): CompiledResult {
      const finalOptions = Object.create(baseOptions)
      const errors = []
      const tips = []
		......
      const compiled = baseCompile(template.trim(), finalOptions)
      if (process.env.NODE_ENV !== 'production') {
        detectErrors(compiled.ast, warn)
      }
      compiled.errors = errors
      compiled.tips = tips
      return compiled
    }

    return {
      compile,
      compileToFunctions: createCompileToFunctionFn(compile)
    }
  }
}

```

compileToFunctions 调用最后返回 一个 返回一个函数包裹 传入的baseCompile 函数的返回值