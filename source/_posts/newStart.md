---
title: 关于vue单页面应用的动态菜单和角色权限的实践
date: 2019-08-27 22:39:35
tags:
---

> 新接手一个vue后台项目，技术栈使用vue全家桶，因每个角色登录的菜单和权限各不一样。经过方案调研决定后台返回菜单和权限数据来显示，具体思路就是 每一个登录的用户生成一个token,添加到axios请求头，再请求菜单和权限接口。前端组装菜单数组进行渲染。

### addRouters方法动态添加路由

`this.$router.addRoutes('已组装好的路由') 同时存到sessionStorage  `

> 用户权限 通过后台返回的权限表 添加一个自定义方法

```
Vue.directive('has', {
 bind: function (el,val){
  if (!Vue.prototype.$_has(val)) {
   el.parentNode.removeChild(el);
  }
 }
});

// 权限检查方法
Vue.prototype.$_has = function (value) {
 let isExist = false;
 let btnPermissionsStr = sessionStorage.getItem("permissions");
 if (btnPermissionsStr == undefined || btnPermissionsStr == null) {
  return false;
 }
 if (value.includes(btnPermissionsStr)) {
  isExist = true;
 }
 return isExist;
};
<button v-has="'edit'">编辑</button> 
```

