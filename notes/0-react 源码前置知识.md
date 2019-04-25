\- htmlnodetype 

```javascript
/*** react-dom/src/shared/HTMLNodeType.js ***/ 

export const ELEMENT_NODE = 1; // 元素节点 

export const TEXT_NODE = 3; // 文本节点 

export const COMMENT_NODE = 8; // 注释节点 

export const DOCUMENT_NODE = 9; // 文档节点 

export const DOCUMENT_FRAGMENT_NODE = 11; // 文档片段节点 
```

每个 DOM 节点都会有一个 nodeType 属性 

[深入理解DOM节点类型第一篇——12种DOM节点类型概述](http://www.cnblogs.com/xiaohuochai/p/5785189.html) 

\- React Fiber 中的 mode 

```javascript
export const NoContext = 0b000; 

export const ConcurrentMode = 0b001; 

export const StrictMode = 0b010; 

export const ProfileMode = 0b100; 
```

翻译过来就是： 

1，普通模式，同步渲染，React15-16的生产环境用， 

2，并发模式，异步渲染，React17的生产环境用， 

3，严格模式，用来检测是否存在废弃API，React16-17开发环境使用， 

4，性能测试模式，用来检测哪里存在性能问题，React16-17开发环境使用 

[React fiber的mode](https://zhuanlan.zhihu.com/p/54037407) 