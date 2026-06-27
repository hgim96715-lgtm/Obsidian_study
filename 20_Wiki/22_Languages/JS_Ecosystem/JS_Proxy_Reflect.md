---
aliases:
  - Proxy
  - Reflect
  - 메타프로그래밍
tags:
  - JavaScript
related:
  - "[[00_JS_Ecosystem_HomePage]]"
---

# JS_Proxy_Reflect — Proxy & Reflect

---

---

# ① Proxy — 객체 접근 가로채기 ⭐️

```javascript
// new Proxy(대상객체, 핸들러)
const handler = {
  get(target, key) {
    console.log(`${key} 읽음`);
    return key in target ? target[key] : '없음';
  },
  set(target, key, value) {
    console.log(`${key} = ${value} 저장`);
    target[key] = value;
    return true;   // 반드시 true 반환
  }
};

const user = new Proxy({ name: 'Kim' }, handler);

user.name;          // "name 읽음" → 'Kim'
user.age;           // "age 읽음" → '없음'
user.age = 25;      // "age = 25 저장"
```

```
Proxy 로 할 수 있는 것:
  get    속성 읽기 가로채기
  set    속성 쓰기 가로채기
  has    in 연산자 가로채기
  deleteProperty  delete 가로채기
  apply  함수 호출 가로채기

활용:
  유효성 검사 (음수 금지 등)
  변경 감지 (Vue.js 반응성의 원리)
  기본값 반환
  로깅
```

## 실전 예시 — 유효성 검사

```javascript
const validator = {
  set(target, key, value) {
    if (key === 'age' && typeof value !== 'number') {
      throw new TypeError('age 는 숫자여야 합니다');
    }
    if (key === 'age' && value < 0) {
      throw new RangeError('age 는 0 이상이어야 합니다');
    }
    target[key] = value;
    return true;
  }
};

const person = new Proxy({}, validator);
person.age = 25;   // OK
person.age = -1;   // RangeError
person.age = '25'; // TypeError
```

---

---

# ② Reflect — 기본 동작 호출 ⭐️

```javascript
// Reflect = 객체 기본 동작을 메서드로 호출
const obj = { name: 'Kim' };

Reflect.get(obj, 'name')          // 'Kim'  (obj.name 과 동일)
Reflect.set(obj, 'age', 25)       // true   (obj.age = 25 와 동일)
Reflect.has(obj, 'name')          // true   ('name' in obj 와 동일)
Reflect.deleteProperty(obj, 'age') // true  (delete obj.age 와 동일)
Reflect.ownKeys(obj)              // ['name']
```

## Proxy + Reflect 조합 패턴

```javascript
const handler = {
  get(target, key, receiver) {
    console.log(`get: ${key}`);
    return Reflect.get(target, key, receiver);
    // 기본 get 동작 그대로 수행
  },
  set(target, key, value, receiver) {
    console.log(`set: ${key} = ${value}`);
    return Reflect.set(target, key, value, receiver);
    // 기본 set 동작 그대로 수행
  }
};
```

```
왜 Reflect 를 쓰나:
  Proxy 핸들러에서 기본 동작을 직접 구현하면 복잡
  Reflect 가 기본 동작을 대신 처리
  this 바인딩 등 엣지 케이스도 올바르게 처리
```

---

---

# ③ 메타프로그래밍이란

```
메타프로그래밍 = 코드가 자기 자신 또는 다른 코드를 조작
프로그램이 실행 시 자신의 구조나 동작을 바꿀 수 있음

JS 메타프로그래밍 도구:
  Proxy    = 객체 동작 가로채기 / 재정의
  Reflect  = 기본 동작 명시적 호출
  Symbol   = 내장 동작 커스터마이즈

실무 활용:
  ORM / 반응형 상태관리 (Vue.js) / DI 컨테이너
  NestJS 데코레이터도 메타프로그래밍 개념
```