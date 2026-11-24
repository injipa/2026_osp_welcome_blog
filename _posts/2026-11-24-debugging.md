---
layout: post
title: "Debugging: pyrebase 경로 초기화 동작으로 인한 Firebase DB 전체 덮어쓰기 버그 해결"
author: 황보희
type: Debugging
order: 4
---

## Context

상품 상세 페이지에 좋아요 기능을 붙이던 중에 겪은 일입니다. 하트 버튼을 누르면 페이지가 새로고침되지 않고(AJAX) 바로 하트가 채워지면서 좋아요 숫자가 올라가는 기능이었습니다.

기능 구현을 마치고 `main` 브랜치에 합친 뒤, 화면에서 하트를 눌러 보았습니다. 그런데 하트는 분명히 채워졌는데 옆에 뜨는 숫자는 `0` 그대로였습니다.

이상해서 Firebase 콘솔에 들어가 보았습니다. 그런데 이때까지의 데이터가 전부 사라지고, 아래 한 줄만 덩그러니 남아 있었습니다.

```text
[https://ewha-market-....firebasedatabase.app/](https://ewha-market-....firebasedatabase.app/): true
```

팀 전체가 공용으로 쓰던 데이터베이스가 `true`라는 값 하나로 통째로 덮어써진 것입니다.

---

## Problem or goal

### 문제가 된 코드

좋아요 데이터는 Firebase에 `like / 상품ID / 사용자ID : true` 구조로 저장하고, 이미 좋아요를 누른 상태라면 해당 데이터를 지워서(취소) 토글하는 방식으로 작성했습니다.

```python
def toggle_like(self, item_id, user_id):
    ref = self.db.child("like").child(item_id).child(user_id)
    if ref.get().val():   # 1) 이미 눌렀는지 확인
        ref.remove()      # 2-a) 눌렀으면 삭제(취소)
        return False
    ref.set(True)         # 2-b) 안 눌렀으면 True 저장
    return True
```

똑같은 긴 경로(`like` → `item_id` → `user_id`)를 세 번이나 쓰기 번거로우니 `ref`라는 변수에 한 번 담아두고 재사용했습니다. 중복 코드를 줄이려고 한 건데, 바로 이 부분이 문제였습니다.

### 목표

1. `ref` 변수를 재사용했을 때 왜 DB 맨 꼭대기(루트)가 `true`로 덮어써졌는지 원인을 정확히 찾아냅니다.
2. 다른 데이터는 절대 건드리지 않고 `like/상품ID/사용자ID` 경로에만 안전하게 저장·삭제되도록 `toggle_like` 코드를 수정합니다.
3. 날아간 팀 공용 Firebase 데이터를 복구하고, 다시는 이런 일이 생기지 않도록 테스트 방법을 바꿉니다.

---

## Steps

### Step 1. 원인 분석 — `pyrebase`는 요청 한 번에 경로를 초기화한다

문제는 우리가 사용하던 `pyrebase` 라이브러리의 동작 방식에 있었습니다. 라이브러리 내부 코드를 확인해 보니, `child()` 메서드는 새로운 경로 객체를 복사해서 만들어 주는 것이 아니라 **단 하나의 DB 객체 안에 있는 `path` 문자열에 경로를 이어 붙인 뒤 자기 자신을 그대로 반환**하고 있었습니다. 게다가 `get()`, `set()`, `remove()` 같은 요청을 한 번이라도 보내고 나면 내부의 `path`를 빈 문자열(`""`)로 초기화해 버립니다.

즉, `ref` 변수 자체는 계속 DB 객체이지만 `ref`가 가리키는 DB 객체 속의 `path`만 비어버립니다. 그래서 위의 코드는 실제로 다음과 같은 순서로 동작했습니다.

| 순서 | 실행 코드 | 실제 요청 경로 |
| :--- | :--- | :--- |
| 1 | `ref.get()` | `like/상품ID/사용자ID` (정상 조회 후 내부 경로 초기화) |
| 2-b | `ref.set(True)` | 경로가 비어 있음 → **DB 맨 꼭대기(루트)** |

첫 번째 클릭에서는 직전의 `ref.get()`으로 인해 경로가 초기화되어 DB 루트 전체가 `true`로 덮어써졌고, 같은 원리로 두 번째 클릭에서는 `ref.remove()`가 실행되면서 루트 전체를 삭제해 버린 것입니다. 화면의 하트가 채워졌던 이유는 서버 입장에서 `set` 요청 자체가 에러 없이 성공했기 때문이고, 숫자가 `0`이었던 이유는 `like` 노드 자체가 통째로 날아가서 개수를 셀 데이터가 없었기 때문입니다.

### Step 2. 사전 테스트에서 놓친 지점 점검

코드를 `main`에 합치기 전에 테스트를 아예 안 한 것은 아니었습니다. 다만 실제 Firebase DB를 건드리지 않으려고 가짜 DB 객체로 테스트를 진행했습니다. 그 가짜 객체는 `child()`를 부를 때마다 새 경로 객체를 만들어 주는, "당연히 그렇겠지" 하고 생각한 방식으로 동작했습니다. 실제 `pyrebase`의 특이한 동작을 반영하지 못했기 때문에 테스트는 전부 통과해 버렸고, 결국 실제 DB에 붙자마자 사고가 터진 것입니다.

### Step 3. 코드 수정 및 실제 라이브러리 기반 네트워크 검증

원인을 파악한 뒤에는 경로를 변수에 담아 재사용하지 않고, 요청을 보낼 때마다 `child()`로 처음부터 경로를 다시 지정하도록 수정했습니다.

```python
def toggle_like(self, item_id, user_id):
    if self.db.child("like").child(item_id).child(user_id).get().val():
        self.db.child("like").child(item_id).child(user_id).remove()
        return False
    self.db.child("like").child(item_id).child(user_id).set(True)
    return True
```

수정 후에는 가짜 DB 대신 실제 `pyrebase` 라이브러리를 사용하되, 네트워크 요청만 가로채서 어떤 경로로 요청이 나가는지 기록하며 검증했습니다. 모든 요청이 `like/상품ID/사용자ID` 경로로만 정확히 나가는 것을 확인한 뒤에야 다시 `main`에 병합했습니다.

### Step 4. 소실된 데이터 복구

안타깝게도 Firebase 무료 플랜은 자동 백업을 지원하지 않아서 이미 날아간 데이터는 되살릴 수 없었습니다. 그래서 다음과 같이 데이터를 복구하였습니다.

- **상품 데이터:** 저장소에 있던 일러스트 이미지를 활용하는 테스트 상품 등록 스크립트(`tools/seed_items.py`)를 실행해 다시 채워 넣었습니다. 같은 상품명이 있으면 건너뛰도록 만들어 두어서 여러 번 실행해도 안전하게 복구할 수 있었습니다.
- **회원 데이터:** 비밀번호를 해시값으로만 저장하고 있어서 복구가 불가능했기 때문에, 팀원 모두가 다시 회원가입을 진행했습니다.
- **기타 데이터:** 페이지네이션 동작을 확인하려고 넣어 두었던 테스트용 상품 데이터들도 다시 채워 넣었습니다.

---

## Result

코드를 수정한 뒤 상품 상세 페이지에서 좋아요 버튼을 누르면 새로고침 없이 하트 모양과 숫자가 정상적으로 바뀌고, Firebase 콘솔에서도 다른 상품·회원 데이터는 그대로 유지된 채 `like/상품ID/사용자ID: true`만 깔끔하게 생겼다가 지워지는 것을 확인했습니다.

![좋아요 기능 정상 작동 화면]({{ "/assets/images/debugging-like-working.png" | relative_url }})

![Firebase 콘솔 like 노드 정상 반영 화면]({{ "/assets/images/debugging-firebase-like-result.png" | relative_url }})

---

## What I learned

1. **라이브러리 동작을 내 마음대로 추측하지 말자:** 함수가 새 객체를 만들어 주는지, 기존 객체의 값을 직접 바꾸는지에 따라 변수 재사용이 위험할 수 있다는 걸 느꼈습니다. 헷갈릴 때는 문서나 라이브러리 내부 코드를 직접 열어보는 습관을 들여야겠습니다.
2. **가짜 테스트만 믿지 말자:** 내가 직접 만든 가짜 객체로 테스트하면 결국 내가 생각한 범위 안에서만 검증이 됩니다. 외부 라이브러리와 연결되는 부분은 실제 라이브러리를 써서 요청이 제대로 나가는지 꼭 확인해야 한다는 걸 배웠습니다.
3. **공용 DB를 건드릴 때는 백업부터 하자:** 팀원 모두가 같은 DB를 쓰고 있어서 버그 하나가 모두의 데이터를 날리는 큰 사고로 이어졌습니다. 앞으로 DB를 건드리는 큰 작업을 합치기 전에는 Firebase 콘솔에서 'JSON 내보내기'로 미리 백업해 두기로 했습니다.

---

## References

- [Pyrebase GitHub 저장소 (Database `child` 및 `path` 구현 코드)](https://github.com/thisbejim/Pyrebase)
- [Firebase Realtime Database 공식 문서 - 데이터 읽기 및 쓰기](https://firebase.google.com/docs/database/web/read-and-write)
- [Ewha Market 저장소](https://github.com/devleeji/2026_osp_project_welcome)
