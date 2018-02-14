---
title: vue
---

v-on
v-bind
v-for
v-once
v-pre
v-cloak
priority: v-for > v-if for문을 우선함.
v-if
v-else
v-else-if
v-show
v-model (2way-binding)
new Vue({
  el
  data
  filters 파이프.
  methods
  computed 계산된 함수를 지정하여 처리. 
   - 메소드와 동일하지만, 계산된 속성은 종속성에 따라 캐시된다
   - 메소드 호출은 재 렌더링 할 때마다 항상 메소드를 호출합니다.
   - get, set 모두 지정가능
  watch : 임의의 변수에 바인딩을 함. (async 반응. 이것보다는 computed를 써라)
  components: 컴포넌트 {
    tempate:
    props:
    data: function
    methods: {}
  }
})


### issues
##### vscode에서 typescript사용할때 우선 decorator할때 에러남.
```
Experimental support for decorators is a feature that is subject to change in a future release. Set the 'experimentalDecorators' option to remove this warning.
```
multi project인 경우에 발생함.
single project를 사용하는 괜찮음.