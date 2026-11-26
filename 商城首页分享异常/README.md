# 商城首页分享异常 · V3.0


## 一、**任务描述**

租户：项目 A

搭建对接人：搭建对接人 A

需求：示例商城首页分享代码出现bug，凡是在代码片段下面的动作都不执行

```javascript
// 代码片段中使用【控件变量】和【组件上下文】的正确方式： // 1.如果需要获取【控件变量】，请先在代码片段开头执行：const topThis = getSlotRealThis(self)，再通过 topThis.state.xxx 来获取控件变量 // 2.如果需要获取【组件上下文】，则直接从self上获取即可，比如：self.record const getSlotRealThis = (self) => { while (self.$this$) { self = self.$this$; } return self } const domain = this.state.domain; window.wx.ready(function () { //需在用户可能点击分享按钮前就先调用 window.wx.updateAppMessageShareData({ title: '示例商城', // 分享标题 desc: '商城分享说明', // 分享描述 link: domain + '/share-page', // 分享链接，该链接域名或路径必须与当前页面对应的服务号JS安全域名一致 imgUrl: domain + '/share-icon', // 分享图标 success: function () { // 设置成功 ，可以加调试信息 console.log("设置分享成功") } }) });：
```



## 二、问题定位

1、影响页面：  

商城首页，

从 MySQL 菜单表已确认：

影响页面 = 商城首页，PAGE_ID = `已隐去`，fileName = `homePage`

2、问题代码涉及 3 个动作组

页面共有 14 个动作组，其中 **动作组11、13、14** 包含自定义代码片段，全部存在 `this` 上下文引用错误：

| 动作组 | 原始代码                                              | 问题                                      |
| ------ | ----------------------------------------------------- | ----------------------------------------- |
| 11     | `console.log("总记录数", this, state.total)`          | `state` 不是全局变量 → `ReferenceError`   |
| 13     | `const domain = this.state.domain;` + `wx.ready(...)` | `this.state` 为 `undefined` → `TypeError` |
| 14     | `location.replace()`                                  | 缺 URL 参数，无效调用（空操作）           |

2. 根因分析——真正的崩溃点

用户看到的现象是"分享代码出bug"，但**第一个崩溃的不是分享代码**：

```
动作组11: console.log("总记录数", this, state.total)  ←  这里先崩
    ↓ ReferenceError: state is not defined
动作组12: 清掉搜索的缓存                              ← ❌ 跳过
动作组13: 微信分享代码                                ← ❌ 跳过（根本没执行到）
动作组14: location.replace()                          ← ❌ 跳过
```

**根因**：低代码平台自定义代码片段中，`this` 不是页面根组件。代码注释已明确提示需使用 `getSlotRealThis(self)` 获取 `topThis`，但三处代码片段都没有正确使用。



## 三、解决方案

### （一）解决方案一

直接修改**动作组11、13、14** 包含自定义代码片段

参考地址：（内部链接已隐去） A&appCode=mix&id=已隐去

#### 动作组11

```javascript
const getSlotRealThis = (self) => {
 while (self.$this$) {
  self = self.$this$;
 }
 return self
}
const topThis = getSlotRealThis(self);
console.log("总记录数", topThis.state.total)
```

改动：`this, state.total` → `topThis.state.total`

#### 动作组13

```javascript
const getSlotRealThis = (self) => {
 while (self.$this$) {
  self = self.$this$;
 }
 return self
}
const topThis = getSlotRealThis(self);
const domain = topThis.state.domain;

if (window.wx) {
  window.wx.ready(function () {
    window.wx.updateAppMessageShareData({
      title: '示例商城',
      desc: '商城分享说明',
      link: domain + '/share-page',
      imgUrl: domain + '/share-icon',
      success: function () {
        console.log("设置分享成功")
      }
    })
  });
}
```

改动：

- `this.state.domain` → `topThis.state.domain`
- 新增 `if (window.wx)` 守卫，避免浏览器预览环境因微信 JS-SDK 不存在而报错

#### 动作组14

```javascript
const getSlotRealThis = (self) => {
 while (self.$this$) {
  self = self.$this$;
 }
 return self
}
```

改动：原始 `location.replace()` 缺参数本为空操作，保持原有"什么都不做"的行为。

## 四、验证

| 序号 | 验证点                  | 预期结果                          | 修复前                 |
| ---- | ----------------------- | --------------------------------- | ---------------------- |
| ①    | 控制台无红色报错        | 无 `TypeError` / `ReferenceError` | `state is not defined` |
| ②    | `总记录数` 日志正常打印 | `总记录数 {total: XX}`            | 不出现                 |
| ③    | `wx.ready` 不报错       | 浏览器预览环境静默跳过            | —                      |
| ④    | 页面无死循环刷新        | 请求数正常（~24 个）              | 413 个请求             |
