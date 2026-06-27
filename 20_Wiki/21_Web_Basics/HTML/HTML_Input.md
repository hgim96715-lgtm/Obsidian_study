---
aliases:
  - input 태그
  - type 종류
  - input 속성
tags:
  - HTML
related:
  - "[[00_HTML_HomePage]]"
  - "[[HTML_FormData]]"
---

# HTML_Input — input 태그 / type 종류 / 주요 속성

# 한 줄 요약

```txt
<input> = 사용자에게 값을 입력받는 HTML 폼 요소

type 속성에 따라 텍스트, 숫자, 날짜, 체크박스, 파일 등 입력 방식이 달라지고,
name 속성이 있어야 폼 제출이나 FormData 수집 시 "이름-값" 형태로 전달됨
```

---

---

# 기본 구조 ⭐️⭐️

```html
<input
  type="text"
  id="nickname"
  name="nickname"
  value=""
  placeholder="닉네임을 입력하세요"
/>
```

| 속성 | 역할 |
| --- | --- |
| `type` | 입력 종류 — text, number, email, checkbox 등 |
| `id` | HTML 문서 안에서 요소를 구분하는 식별자 |
| `name` | 폼 제출 시 서버로 전달되는 필드 이름 |
| `value` | 입력 요소가 가지고 있는 실제 값 |
| `placeholder` | 입력 전 표시되는 안내 문구 |

```txt
<input>은 닫는 태그가 없는 빈 요소(void element)

<input></input> ❌
<input />       ⭕
```

`type`을 생략하면 기본값은 `text`이다.

```html
<input name="nickname" />

<!-- 아래와 동일 -->
<input type="text" name="nickname" />
```

---

---

# id와 name의 차이 ⭐️⭐️⭐️

```html
<label for="email">이메일</label>
<input id="email" name="email" type="email" />
```

```txt
id
  → HTML 문서 안에서 요소를 식별
  → <label for="email">과 연결
  → CSS, JavaScript에서 요소를 찾을 때 사용

name
  → 폼을 제출할 때 서버로 전달되는 필드 이름
  → FormData의 key가 됨
```

```javascript
const input = document.querySelector('#email');

console.log(input.id);   // 'email'
console.log(input.name); // 'email'
```

```javascript
const formData = new FormData(form);

formData.get('email');
// name="email"인 input의 값을 가져옴
```

```txt
FormData와 연결되는 속성은 id가 아니라 name

<input id="email">                → FormData에 포함되지 않음
<input id="email" name="email">   → FormData에 포함됨
```

---

---

# label과 input 연결하기 ⭐️⭐️

```html
<label for="nickname">닉네임</label>
<input id="nickname" name="nickname" type="text" />
```

```txt
<label>의 for 값과 <input>의 id 값을 동일하게 지정

<label for="nickname">
<input id="nickname">

→ 라벨을 클릭해도 input에 포커스됨
→ 스크린 리더가 입력 필드의 의미를 알 수 있음
```

## input을 `label` 내부에 넣는 방식도 가능하다.

```html
<label>
  닉네임
  <input name="nickname" type="text" />
</label>
```

```txt
placeholder는 label을 대신할 수 없음

placeholder는 입력을 시작하면 사라지고,
입력 필드가 무엇을 의미하는지 항상 보여주지 못하기 때문
```

---

---

# 자주 사용하는 type 종류 ⭐️⭐️⭐️

| type | 역할 | 입력값 예시 |
| --- | --- | --- |
| `text` | 일반 텍스트 입력 | `홍길동` |
| `password` | 비밀번호 입력 | 화면에 값이 가려짐 |
| `email` | 이메일 형식 입력 | `test@test.com` |
| `number` | 숫자 입력 | `10` |
| `tel` | 전화번호 입력 | `010-1234-5678` |
| `url` | URL 입력 | `https://example.com` |
| `search` | 검색어 입력 | `NestJS` |
| `date` | 날짜 선택 | `2026-06-25` |
| `time` | 시간 선택 | `14:30` |
| `datetime-local` | 날짜와 시간 선택 | `2026-06-25T14:30` |
| `checkbox` | 여러 항목 선택 | 관심사, 약관 동의 |
| `radio` | 여러 항목 중 하나 선택 | 공개 범위 |
| `file` | 파일 선택 | 이미지, 문서 |
| `hidden` | 화면에 보이지 않는 값 | 게시글 ID |
| `submit` | 폼 제출 버튼 | 회원가입 |
| `reset` | 입력값 초기화 버튼 | 초기화 |
| `button` | 일반 버튼 | 별도 JavaScript 동작 |

---

---

# 텍스트 계열 input

## text

```html
<input
  type="text"
  name="nickname"
  placeholder="닉네임"
  minlength="2"
  maxlength="20"
  required
/>
```

```txt
일반 문자열을 입력받을 때 사용

minlength  최소 문자열 길이
maxlength  최대 문자열 길이
required   반드시 입력해야 함
```

## password

```html
<input
  type="password"
  name="password"
  minlength="8"
  autocomplete="new-password"
  required
/>
```

```txt
입력한 문자열을 화면에서 가려서 보여줌

하지만 password 타입 자체가 값을 암호화하는 것은 아님

→ 서버 전송 시 HTTPS 필요
→ 서버에서는 bcrypt, Argon2 등으로 해시해서 저장해야 함
```

## email

```html
<input
  type="email"
  name="email"
  autocomplete="email"
  required
/>
```

```txt
브라우저가 기본적인 이메일 형식을 검사함

test@test.com  ⭕
test           ❌

하지만 완벽한 이메일 검증은 아님
→ 서버에서도 다시 검증해야 함
```

여러 이메일 입력을 허용하려면 `multiple`을 사용한다.

```html
<input type="email" name="emails" multiple />
```

## tel

```html
<input
  type="tel"
  name="phone"
  inputmode="tel"
  placeholder="010-1234-5678"
/>
```

```txt
전화번호는 국가마다 형식이 달라서
type="tel"만으로 특정 형식을 자동 검증하지 않음

필요하다면 pattern 또는 JavaScript/서버 검증을 함께 사용
```

## url

```html
<input
  type="url"
  name="homepage"
  placeholder="https://example.com"
/>
```

```txt
URL 형식의 문자열을 입력받을 때 사용
브라우저가 기본적인 URL 형식을 검사함
```

---

---

# number — 숫자 입력 ⭐️⭐️

```html
<input
  type="number"
  name="age"
  min="1"
  max="120"
  step="1"
/>
```

| 속성 | 역할 |
| --- | --- |
| `min` | 입력 가능한 최솟값 |
| `max` | 입력 가능한 최댓값 |
| `step` | 숫자의 증가 단위 |

```html
<input type="number" name="quantity" min="1" max="10" step="1" />

<input type="number" name="price" min="0" step="100" />

<input type="number" name="rating" min="0" max="5" step="0.5" />
```

## JavaScript에서 number 값 읽기

```html
<input id="age" type="number" value="20" />
```

```javascript
const ageInput = document.querySelector('#age');

console.log(ageInput.value);         // '20'
console.log(typeof ageInput.value);  // 'string'

console.log(Number(ageInput.value)); // 20
console.log(ageInput.valueAsNumber); // 20
```

```txt
type="number"라도 input.value는 기본적으로 문자열

숫자 계산에 사용하려면:

Number(input.value)
input.valueAsNumber

등으로 숫자로 변환해야 함
```

```txt
주민등록번호, 전화번호, 우편번호처럼
숫자처럼 보여도 계산하지 않는 값에는 type="number"를 사용하지 않음

이런 값은 0으로 시작할 수 있고,
+, -, 소수점 같은 숫자 입력 기능이 필요하지 않기 때문

전화번호 → type="tel"
우편번호 → type="text" + inputmode="numeric"
```

---

---

# 날짜와 시간

```html
<input type="date" name="startDate" />
<input type="time" name="startTime" />
<input type="datetime-local" name="reservationAt" />
```

## date 범위 제한

```html
<input
  type="date"
  name="exhibitionDate"
  min="2026-01-01"
  max="2026-12-31"
/>
```

```txt
date의 value 형식:

YYYY-MM-DD

예:
2026-06-25
```

## datetime-local 주의점

```html
<input type="datetime-local" name="reservationAt" />
```

```txt
datetime-local은 날짜와 시간을 입력받지만
시간대(timezone) 정보는 포함하지 않음

예:
2026-06-25T17:30

이 값만으로는 서울 시간인지, 뉴욕 시간인지 알 수 없음
→ 서버에서 사용자의 시간대 정책을 별도로 정해야 함
```

---

---

# checkbox — 여러 값 선택 ⭐️⭐️⭐️

```html
<label>
  <input type="checkbox" name="hobby" value="music" />
  음악
</label>

<label>
  <input type="checkbox" name="hobby" value="running" />
  러닝
</label>

<label>
  <input type="checkbox" name="hobby" value="exhibition" />
  전시
</label>
```

```txt
체크박스 여러 개에 같은 name을 지정하고
각각 다른 value를 지정함

선택된 항목만 FormData에 포함됨
```

음악과 전시를 선택했다면:

```javascript
formData.get('hobby');
// 'music'

formData.getAll('hobby');
// ['music', 'exhibition']
```

```txt
같은 name으로 여러 값이 들어올 수 있으므로
checkbox 그룹은 get()보다 getAll()을 사용하는 것이 안전
```

## 체크 여부 확인

```javascript
const checkbox = document.querySelector(
  'input[name="agree"]',
);

console.log(checkbox.checked);
// true 또는 false
```

## 단일 체크박스

```html
<label>
  <input
    type="checkbox"
    name="agreeTerms"
    value="yes"
    required
  />
  이용약관에 동의합니다.
</label>
```

```txt
체크됨:
  agreeTerms = 'yes'

체크되지 않음:
  FormData에 agreeTerms 자체가 포함되지 않음
```

```javascript
const agreed = formData.get('agreeTerms') === 'yes';
```

## checked

```html
<input
  type="checkbox"
  name="notifications"
  value="email"
  checked
/>
```

```txt
checked는 처음 화면이 열렸을 때
기본적으로 선택된 상태로 만드는 속성
```

---

---

# radio — 하나만 선택 ⭐️⭐️⭐️

```html
<label>
  <input
    type="radio"
    name="visibility"
    value="public"
    checked
  />
  공개
</label>

<label>
  <input
    type="radio"
    name="visibility"
    value="private"
  />
  비공개
</label>
```

```txt
radio는 같은 name을 가진 항목끼리 하나의 그룹이 됨

같은 그룹에서는 한 항목만 선택할 수 있음
```

공개를 선택했다면:

```javascript
formData.get('visibility');
// 'public'
```

```txt
checkbox
  → 같은 그룹에서 여러 개 선택 가능
  → getAll() 사용 가능

radio
  → 같은 그룹에서 하나만 선택 가능
  → get() 사용
```

---

---

# file — 파일 입력 ⭐️⭐️⭐️

```html
<input
  type="file"
  name="profileImage"
  accept="image/*"
/>
```

```javascript
const fileInput = document.querySelector(
  'input[type="file"]',
);

const file = fileInput.files[0];

console.log(file.name);
console.log(file.type);
console.log(file.size);
```

```txt
fileInput.files는 FileList

files[0]으로 첫 번째 파일을 가져옴
```

## 여러 파일 선택

```html
<input
  type="file"
  name="photos"
  accept="image/*"
  multiple
/>
```

```javascript
const files = fileInput.files;

for (const file of files) {
  console.log(file.name);
}
```

```txt
multiple 속성이 있으면 사용자가 여러 파일을 선택할 수 있음
```

## accept

```html
<input type="file" accept="image/png, image/jpeg" />

<input type="file" accept="image/*" />

<input type="file" accept=".pdf" />
```

```txt
accept는 사용자가 선택할 파일 종류를 안내하고 제한하는 속성

하지만 보안 검증이 아님

→ 파일 확장자
→ MIME 타입
→ 실제 파일 내용

은 서버에서도 검사해야 함
```

---

---

# hidden — 화면에 보이지 않는 값

```html
<input type="hidden" name="postId" value="123" />
```

```txt
사용자에게 입력창은 보이지 않지만
폼 제출 시 name-value 값은 서버로 전달됨

예:
  게시글 ID
  수정 대상 ID
  CSRF 토큰
```

```txt
hidden은 보이지 않을 뿐, 안전하게 숨겨진 값이 아님

개발자 도구에서 값 확인과 수정이 가능함

→ 권한이나 가격처럼 중요한 값은 서버에서 반드시 다시 검증해야 함
```

---

---

# submit / reset / button

```html
<input type="submit" value="회원가입" />
<input type="reset" value="초기화" />
<input type="button" value="중복 확인" />
```

```txt
submit
  → form을 제출

reset
  → form 입력값을 초기 상태로 되돌림

button
  → 기본 동작 없음
  → JavaScript 이벤트를 연결해서 사용
```

일반적으로는 `<input type="button">`보다 `<button>`을 더 많이 사용한다.

```html
<button type="submit">회원가입</button>
<button type="reset">초기화</button>
<button type="button">중복 확인</button>
```

```txt
<button>은 내부에 아이콘, span 등의 HTML 요소를 넣을 수 있어서
스타일과 구조를 구성하기 더 편리함
```

## button의 기본 type 주의

```html
<form>
  <button>버튼</button>
</form>
```

```txt
<form> 내부의 <button>은 type을 생략하면
기본적으로 submit으로 동작함

폼 제출 목적이 아니라면 명확하게 작성:

<button type="button">일반 버튼</button>
```

---

---

# 주요 공통 속성 ⭐️⭐️⭐️

| 속성 | 역할 |
| --- | --- |
| `name` | 폼 제출 시 전달되는 필드 이름 |
| `value` | 입력 요소의 값 |
| `placeholder` | 입력 전 표시되는 안내 문구 |
| `required` | 필수 입력 |
| `disabled` | 입력과 클릭을 비활성화 |
| `readonly` | 값을 수정하지 못하게 함 |
| `autocomplete` | 브라우저 자동 완성 설정 |
| `autofocus` | 페이지가 열릴 때 자동 포커스 |
| `minlength` | 문자열 최소 길이 |
| `maxlength` | 문자열 최대 길이 |
| `min` | 숫자·날짜 최솟값 |
| `max` | 숫자·날짜 최댓값 |
| `step` | 숫자·시간 증가 단위 |
| `pattern` | 입력값이 만족해야 하는 정규 표현식 |
| `multiple` | 여러 이메일 또는 파일 선택 허용 |
| `accept` | 선택 가능한 파일 종류 안내 |
| `checked` | checkbox·radio의 기본 선택 상태 |
| `inputmode` | 모바일에서 표시할 키보드 종류 힌트 |

---

---

# required — 필수 입력

```html
<input type="text" name="nickname" required />
```

```txt
required가 있는 입력값이 비어 있으면
브라우저가 기본적으로 폼 제출을 막음
```

```html
<input type="checkbox" name="agree" required />
```

```txt
checkbox에 required를 사용하면
해당 체크박스를 선택해야 제출할 수 있음
```

---

---

# disabled vs readonly ⭐️⭐️⭐️

```html
<input
  name="nickname"
  value="홍길동"
  disabled
/>

<input
  name="email"
  value="test@test.com"
  readonly
/>
```

| 속성 | 사용자 수정 | 포커스 | FormData·폼 제출 |
| --- | --- | --- | --- |
| `disabled` | 불가능 | 불가능 | 포함되지 않음 |
| `readonly` | 불가능 | 가능 | 포함됨 |

```txt
disabled
  → 입력 자체가 완전히 비활성화
  → name이 있어도 FormData에 포함되지 않음

readonly
  → 사용자가 수정할 수 없지만 값은 존재
  → FormData에 포함됨
```

```javascript
const formData = new FormData(form);

formData.get('nickname');
// null — disabled이므로 제외됨

formData.get('email');
// 'test@test.com' — readonly이므로 포함됨
```

```txt
값을 보여주되 서버에도 전송해야 한다면 readonly 고려

단, readonly 값도 개발자 도구에서 조작할 수 있으므로
서버에서는 전달된 값을 그대로 신뢰하면 안 됨
```

---

---

# autocomplete 

```txt
autocomplete는 자동완성 기능 자체를 구현하는 속성이라기보다,
브라우저에게 "이 칸에는 이메일, 비밀번호, 전화번호 같은 값이 들어간다"고 알려주는 힌트
```

```html
<input
  type="email"
  name="email"
  autocomplete="email"
/>

<input
  type="password"
  name="password"
  autocomplete="current-password"
/>
```

| 값 | 용도 |
| --- | --- |
| `name` | 사용자 이름 |
| `username` | 로그인 아이디 |
| `email` | 이메일 |
| `tel` | 전화번호 |
| `current-password` | 현재 비밀번호 |
| `new-password` | 새 비밀번호 |
| `one-time-code` | 일회용 인증 코드 |
| `off` | 자동 완성을 사용하지 않도록 요청 |

```html
<!-- 로그인 -->
<input
  type="email"
  name="email"
  autocomplete="username"
/>

<input
  type="password"
  name="password"
  autocomplete="current-password"
/>
```

```html
<!-- 회원가입 또는 비밀번호 변경 -->
<input
  type="password"
  name="password"
  autocomplete="new-password"
/>
```

---

---

# inputmode — 모바일 키보드 힌트

```html
<input
  type="text"
  name="verificationCode"
  inputmode="numeric"
/>
```

| 값 | 예상 키보드 |
| --- | --- |
| `text` | 일반 텍스트 키보드 |
| `numeric` | 숫자 중심 키보드 |
| `decimal` | 소수점 입력 키보드 |
| `tel` | 전화번호 키보드 |
| `email` | 이메일 입력 키보드 |
| `url` | URL 입력 키보드 |

```txt
inputmode는 어떤 키보드를 보여줄지 브라우저에 알려주는 힌트

입력값 자체를 검증하거나 제한하지는 않음

→ 실제 값 검증은 HTML 속성, JavaScript, 서버에서 처리
```

우편번호나 인증번호처럼 계산하지 않는 숫자 문자열에 유용하다.

```html
<input
  type="text"
  name="verificationCode"
  inputmode="numeric"
  maxlength="6"
/>
```

---

---

# pattern — 입력 형식 검사

```html
<input
  type="text"
  name="verificationCode"
  pattern="[0-9]{6}"
  title="숫자 6자리를 입력해주세요"
  required
/>
```

```txt
pattern="[0-9]{6}"
  → 숫자 6자리 형식만 허용

title
  → 형식이 맞지 않을 때 사용자에게 보여줄 설명
```

전화번호 예시:

```html
<input
  type="tel"
  name="phone"
  pattern="010-[0-9]{4}-[0-9]{4}"
  placeholder="010-1234-5678"
/>
```

```txt
브라우저 검증은 사용자 편의를 위한 1차 검증

개발자 도구나 직접 HTTP 요청으로 우회할 수 있으므로
서버 검증을 대체할 수 없음
```

---

---

# value와 placeholder 차이

```html
<input
  type="text"
  name="nickname"
  value="홍길동"
  placeholder="닉네임을 입력하세요"
/>
```

```txt
value
  → input이 실제로 가지고 있는 값
  → 폼 제출 시 서버로 전달됨

placeholder
  → 값이 비어 있을 때만 보이는 안내 문구
  → 실제 입력값이 아님
  → 폼 제출 시 전달되지 않음
```

```javascript
input.value;
// '홍길동'

input.placeholder;
// '닉네임을 입력하세요'
```

---

---

# JavaScript에서 값 읽고 변경하기

```html
<input
  id="nickname"
  name="nickname"
  type="text"
/>
```

```javascript
const nicknameInput = document.querySelector('#nickname');

// 현재 값 읽기
console.log(nicknameInput.value);

// 값 변경
nicknameInput.value = '홍길동';

// 비활성화
nicknameInput.disabled = true;

// 읽기 전용
nicknameInput.readOnly = true;

// 필수 입력 여부
nicknameInput.required = true;
```

## checkbox와 radio

```javascript
const checkbox = document.querySelector(
  'input[name="agree"]',
);

console.log(checkbox.checked);

checkbox.checked = true;
```

## 선택된 radio 찾기

```javascript
const selected = document.querySelector(
  'input[name="visibility"]:checked',
);

console.log(selected?.value);
```

---

---

# form과 연결하기 ⭐️⭐️

```html
<form id="signupForm">
  <label for="email">이메일</label>

  <input
    id="email"
    name="email"
    type="email"
    required
  />

  <button type="submit">회원가입</button>
</form>
```

```txt
form 안에서 submit 버튼을 누르면
name이 있는 입력 요소들이 name-value 형태로 제출됨
```

```javascript
const form = document.querySelector('#signupForm');

form.addEventListener('submit', (event) => {
  event.preventDefault();

  const formData = new FormData(form);

  console.log(formData.get('email'));
});
```

## form 밖의 input 연결하기

```html
<form id="signupForm">
  <button type="submit">회원가입</button>
</form>

<input
  form="signupForm"
  type="email"
  name="email"
/>
```

```txt
input의 form 속성에 form의 id를 지정하면
input이 form 밖에 있어도 해당 form에 연결됨
```

---

---

# FormData에 포함되는 조건 ⭐️⭐️⭐️

```html
<form id="exampleForm">
  <input name="nickname" value="홍길동" />

  <input id="email" value="test@test.com" />

  <input name="role" value="USER" disabled />

  <input name="status" value="ACTIVE" readonly />

  <input
    type="checkbox"
    name="hobby"
    value="music"
  />
</form>
```

```javascript
const form = document.querySelector('#exampleForm');
const formData = new FormData(form);
```

| input | FormData 포함 여부 |
| --- | --- |
| `name="nickname"` | 포함 |
| `id="email"`만 있고 name 없음 | 제외 |
| `name="role"` + `disabled` | 제외 |
| `name="status"` + `readonly` | 포함 |
| 선택되지 않은 checkbox | 제외 |

```txt
FormData에 들어가기 위한 핵심 조건:

1. name 속성이 있어야 함
2. disabled 상태가 아니어야 함
3. checkbox/radio라면 선택된 상태여야 함
4. form에 소속된 입력 요소여야 함
```

자세한 내용은 [[HTML_FormData]] 참고.

---

---

# 브라우저 검증과 서버 검증 ⭐️⭐️⭐️

```html
<input
  type="email"
  name="email"
  minlength="5"
  maxlength="100"
  required
/>
```

```txt
HTML 속성을 사용하면 브라우저가 제출 전에 기본 검증을 수행함

하지만 브라우저 검증은 우회할 수 있음:

  개발자 도구로 속성 제거
  JavaScript로 직접 요청
  Postman, curl 등으로 API 직접 호출

따라서 서버에서도 반드시 같은 조건을 다시 검증해야 함
```

```typescript
// NestJS DTO
import {
  IsEmail,
  IsString,
  Length,
} from 'class-validator';

export class SignupDto {
  @IsEmail()
  email: string;

  @IsString()
  @Length(2, 20)
  nickname: string;
}
```

```txt
HTML 검증
  → 사용자가 잘못 입력했을 때 빠르게 알려주는 UX 역할

서버 검증
  → 잘못된 데이터가 시스템에 저장되는 것을 막는
    보안·데이터 무결성 역할
```

---

---

# 자주 하는 실수

## 1. name 없이 id만 작성

```html
<input id="email" type="email" />
```

```javascript
formData.get('email');
// null
```

수정:

```html
<input
  id="email"
  name="email"
  type="email"
/>
```

## 2. number의 value를 숫자라고 생각

```javascript
const quantity = input.value;

console.log(quantity + 1);
// '101' — input.value가 '10'이었다면 문자열 연결
```

수정:

```javascript
const quantity = Number(input.value);

console.log(quantity + 1);
// 11
```

## 3. 선택되지 않은 checkbox도 false로 전달된다고 생각

```html
<input
  type="checkbox"
  name="notifications"
  value="enabled"
/>
```

```txt
체크하지 않으면:

notifications = false로 전달되는 것이 아님
→ notifications 필드 자체가 FormData에 없음
```

```javascript
const enabled =
  formData.get('notifications') === 'enabled';
```

## 4. disabled 값을 서버에서도 받을 수 있다고 생각

```html
<input
  name="email"
  value="test@test.com"
  disabled
/>
```

```txt
disabled input은 FormData와 폼 제출에서 제외됨
```

필요하다면 `readonly`를 사용한다.

```html
<input
  name="email"
  value="test@test.com"
  readonly
/>
```

또는 별도 `hidden` input을 사용한다.

```html
<input value="test@test.com" disabled />

<input
  type="hidden"
  name="email"
  value="test@test.com"
/>
```

단, 서버에서는 전달된 값을 다시 검증해야 한다.

## 5. placeholder를 label 대신 사용

```html
<input
  type="email"
  placeholder="이메일"
/>
```

수정:

```html
<label for="email">이메일</label>

<input
  id="email"
  name="email"
  type="email"
  placeholder="test@test.com"
/>
```

## 6. 클라이언트 검증만 믿음

```html
<input
  type="email"
  name="email"
  required
/>
```

```txt
required와 type="email"이 있어도
API 요청을 직접 보내면 브라우저 검증을 거치지 않을 수 있음

→ 서버 DTO 검증 필수
```

---

---

# 실무 예시 — 회원가입 폼

```html
<form id="signupForm">
  <div>
    <label for="email">이메일</label>

    <input
      id="email"
      name="email"
      type="email"
      autocomplete="email"
      required
    />
  </div>

  <div>
    <label for="password">비밀번호</label>

    <input
      id="password"
      name="password"
      type="password"
      minlength="8"
      autocomplete="new-password"
      required
    />
  </div>

  <div>
    <label for="nickname">닉네임</label>

    <input
      id="nickname"
      name="nickname"
      type="text"
      minlength="2"
      maxlength="20"
      required
    />
  </div>

  <label>
    <input
      type="checkbox"
      name="agreeTerms"
      value="yes"
      required
    />

    이용약관에 동의합니다.
  </label>

  <button type="submit">회원가입</button>
</form>
```

```javascript
const form = document.querySelector('#signupForm');

form.addEventListener('submit', async (event) => {
  event.preventDefault();

  const formData = new FormData(form);

  const email = formData.get('email');
  const password = formData.get('password');
  const nickname = formData.get('nickname');

  const agreed =
    formData.get('agreeTerms') === 'yes';

  console.log({
    email,
    password,
    nickname,
    agreed,
  });
});
```

---

---

# 한눈에

```txt
<input>은 사용자에게 값을 입력받는 HTML 요소
type을 생략하면 기본값은 text

id
  → label, CSS, JavaScript에서 요소 식별

name
  → 폼 제출과 FormData에서 사용되는 필드 이름
  → name이 없으면 서버로 전달되지 않음

value
  → 실제 입력값

placeholder
  → 값이 비어 있을 때 표시되는 안내 문구
  → 실제 값이 아니며 label을 대체할 수 없음

checkbox
  → 여러 개 선택 가능
  → 선택된 항목만 전달
  → 같은 name의 여러 값은 FormData.getAll()

radio
  → 같은 name 그룹 중 하나만 선택
  → FormData.get()

disabled
  → 수정 불가 + FormData에서 제외

readonly
  → 수정 불가 + FormData에는 포함

number 타입이어도 input.value는 문자열
  → Number(value) 또는 valueAsNumber로 변환

required, pattern, min, max 등의 브라우저 검증은
사용자 편의를 위한 1차 검증

→ 서버에서도 반드시 다시 검증해야 함
```