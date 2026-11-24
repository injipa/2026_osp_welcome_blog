---
layout: post
title: "Flask의 session은 어떻게 로그인 상태를 기억할까?"
author: 이지원
type: Concept
order: 3
---

## Context

우리 팀 **어서오소**는 이화대동제 부스 메뉴를 찾고, 사고, 후기를 남기는 마켓 사이트 **Ewha Market**을 만들고 있다. 나는 그중 **회원 관리**(회원가입, 로그인/로그아웃, 마이페이지)를 맡았다.

아이디와 비밀번호가 맞는지 확인하는 기능은 금방 만들었다. 그런데 로그인에 성공해서 메인 페이지로 넘어가면, 메인 페이지는 **방금 누가 로그인했는지 어떻게 알까?** 페이지를 옮길 때마다 아이디와 비밀번호를 다시 보낼 수는 없었다. 수업에서 `session`을 쓰면 된다고 배웠지만 처음에는 "서버 어딘가에 저장되는 변수" 정도로만 알고 있었다. 이 글에서는 직접 구현하면서 알게 된 session의 동작 방식을 정리한다.

## Goal

- 로그인한 사용자를 다른 페이지에서도 알아보기
- 로그인한 사람만 마이페이지, 상품 등록, 리뷰 작성에 들어갈 수 있게 하기
- 로그인 화면의 **"로그인 상태 유지"** 체크박스 만들기

## 1. 왜 session이 필요할까?

웹 브라우저와 서버는 HTTP로 대화하는데, HTTP는 **무상태(stateless)** 다. 서버는 요청 하나에 응답하고 나면 그 요청을 기억하지 않는다. 그래서 서버 입장에서는 로그인 요청을 보낸 사람과 1초 뒤 마이페이지를 요청한 사람이 같은 사람인지 알 수 없다.

이 문제를 해결하려면 브라우저가 요청할 때마다 "나 아까 로그인한 그 사람이야"라는 표시를 같이 보내야 한다. 이 표시를 **쿠키**라고 한다. 서버가 쿠키를 건네주면 브라우저가 저장해 두었다가 다음 요청마다 자동으로 같이 보낸다.

## 2. Flask session = 서명된 쿠키

그런데 쿠키에 `id=jiwon`처럼 그냥 적어 두면, 사용자가 개발자 도구에서 값을 다른 아이디로 바꿔 버릴 수 있다. Flask의 `session`은 이 문제를 **서명**으로 해결한다. 데이터를 쿠키에 담되, `SECRET_KEY`로 만든 서명을 붙여서 보내는 것이다.

우리 프로젝트 `app.py`에는 이렇게 설정되어 있다.

```python
app.config["SECRET_KEY"] = "helloosp"
app.permanent_session_lifetime = timedelta(days=7)  # 로그인 상태 유지 기간
```

- `SECRET_KEY`: session 쿠키에 서명할 때 쓰는 비밀 열쇠다. 이 값이 없으면 session을 쓸 수 없다.
- `permanent_session_lifetime`: "로그인 상태 유지"를 체크했을 때 로그인을 얼마나 유지할지 정한다. 우리는 7일로 했다.

실제로 우리 사이트에 로그인한 뒤 개발자 도구를 열어 보면 이런 쿠키가 생겨 있다.

```
session=eyJfcGVybWFuZW50Ijp0cnVlLCJpZCI6Imppd29uIiwibmlja25hbWUiOiJqaiJ9.awTlkg.-fCmmtDylGqjsxnilQiwfOJsRqk
```

점(`.`)을 기준으로 **데이터 / 서명한 시각 / 서명** 세 부분으로 나뉜다. 신기했던 점은 첫 번째 데이터 부분이 암호화된 게 아니라 Base64로 바꿔 놓은 것뿐이라는 것이다. 디코딩해 보면 내용이 그대로 보인다.

```
{"_permanent":true,"id":"jiwon","nickname":"jj"}
```

즉 Flask session은 **누구나 읽을 수는 있지만, 고칠 수는 없는** 쿠키다. 사용자가 `id`를 바꾸면 서명이 맞지 않게 되고, Flask는 그 쿠키를 무시한다. 그래서 session에는 **비밀번호처럼 남이 보면 안 되는 값을 넣으면 안 된다.** 우리 코드도 session에는 아이디와 닉네임만 넣고, 비밀번호는 해시값으로 DB에만 저장한다.

## 3. 로그인할 때 session에 저장하기

```python
@app.route("/login_confirm", methods=['POST'])
def login_user():
    id_ = request.form['id']
    pw = request.form['pw']
    pw_hash = hashlib.sha256(pw.encode('utf-8')).hexdigest()
    if DB.find_user(id_, pw_hash):
        session['id'] = id_
        user = DB.get_user(id_)
        session['nickname'] = user.get('nickname') if user else id_
        session.permanent = bool(request.form.get('remember'))
        return redirect(url_for('index'))
    else:
        flash("Wrong ID or PW!")
        return render_template("login.html")
```

입력받은 비밀번호를 해시로 바꾼 뒤 DB의 회원 정보와 비교하고, 맞으면 `session['id']`와 `session['nickname']`에 값을 넣는다. session은 **파이썬 딕셔너리처럼** 쓰면 되고, 값을 넣으면 Flask가 알아서 서명된 쿠키를 만들어 응답에 붙여 준다.

마지막 줄의 `session.permanent`는 "로그인 상태 유지" 체크 여부로 정한다.

| 체크 여부 | `session.permanent` | 결과 |
|---|---|---|
| 체크 안 함 | `False` | 브라우저를 닫으면 로그아웃 |
| 체크함 | `True` | 7일 동안 로그인 유지 |

위에서 디코딩한 쿠키에 `"_permanent": true`가 들어 있던 것도, 체크하고 로그인해서 Flask가 이 값을 같이 저장해 둔 것이다.

## 4. 다른 페이지에서 session 꺼내 쓰기

Flask는 템플릿에서도 `session`을 바로 쓸 수 있게 해 준다. 그래서 모든 페이지가 상속하는 공통 헤더(`index.html`)에서 로그인 여부에 따라 메뉴를 바꿨다.

{% raw %}
```html
{% if session['id'] %}
    <span class="nav-link">Hello, {{ session['nickname'] or session['id'] }}!</span>
    <a href="{{ url_for('logout') }}" class="btn btn-primary btn-sm">로그아웃</a>
    <a href="{{ url_for('mypage') }}" class="nav-link">마이페이지</a>
{% else %}
    <a href="{{ url_for('login') }}" class="btn btn-primary btn-sm">로그인</a>
    <a href="{{ url_for('signup') }}" class="nav-link">회원가입</a>
{% endif %}
```
{% endraw %}

로그인한 사람만 써야 하는 페이지는 함수 맨 앞에서 session에 `id`가 있는지 확인하고, 없으면 로그인 페이지로 보낸다.

```python
@app.route("/mypage")
def mypage():
    if 'id' not in session:
        flash("로그인이 필요합니다.")
        return redirect(url_for('login'))

    my_id = session['id']
    ...
```

이 검사를 통과하면 `session['id']`로 **내가 등록한 상품, 내가 쓴 리뷰, 내가 좋아요한 상품**을 골라서 보여 준다. 상품을 등록할 때도 판매자 아이디를 `session['id']`로 자동으로 넣기 때문에, 남의 아이디로 상품을 올릴 수 없다.

## 5. 로그아웃

```python
@app.route("/logout")
def logout():
    session.clear()
    return redirect(url_for('index'))
```

`session.clear()`로 session에 든 값을 모두 지우면, 다음 요청부터는 `'id' not in session`이 참이 되어 로그아웃 상태가 된다.

## Result

![로그인 전 헤더]({{ "/assets/images/session-header-before.png" | relative_url }})

로그인하기 전에는 헤더에 로그인과 회원가입 버튼이 보인다.

![로그인 후 헤더]({{ "/assets/images/session-header-after.png" | relative_url }})

로그인하면 인사말, 로그아웃, 마이페이지가 나타나고, 어느 페이지로 이동해도 이 상태가 유지된다.

![개발자 도구에서 본 session 쿠키]({{ "/assets/images/session-devtools-cookie.png" | relative_url }})

개발자 도구(F12 → Application → Cookies)에서 `session` 쿠키를 확인할 수 있다. "로그인 상태 유지"를 체크하고 로그인했더니 만료일(Expires)이 정확히 7일 뒤로 찍혔다. (끝에 붙은 `Z`는 세계 표준시라는 뜻이라 한국 시간보다 9시간 느리게 표시된다.)

![로그인 없이 마이페이지 접근 시]({{ "/assets/images/session-login-required.png" | relative_url }})

로그인하지 않고 마이페이지 주소를 직접 입력하면 "로그인이 필요합니다." 메시지와 함께 로그인 페이지로 이동한다.

## What I learned

처음에는 `session['id'] = id_` 한 줄이면 끝나는 기능이라고 생각했는데, 그 뒤에서는 쿠키와 서명, 만료 시간이 함께 움직이고 있었다. 직접 쿠키를 열어 보고 나서야 두 가지를 제대로 이해할 수 있었다.

- session 쿠키는 고칠 수는 없어도 **누구나 읽을 수 있기 때문에** 비밀번호 같은 값은 넣으면 안 된다.
- 서명을 지켜 주는 것은 `SECRET_KEY` 하나뿐이다. 이 값이 알려지면 누구든 다른 사람인 척하는 쿠키를 만들 수 있으니, 실제 서비스라면 코드에 직접 쓰지 말고 환경변수 등으로 숨겨야 한다.

## References

- [Flask 공식 문서 - Quickstart: Sessions](https://flask.palletsprojects.com/en/stable/quickstart/#sessions)
- [Flask 공식 문서 - Configuration: SECRET_KEY](https://flask.palletsprojects.com/en/stable/config/#SECRET_KEY)
- [MDN - HTTP 쿠키](https://developer.mozilla.org/ko/docs/Web/HTTP/Cookies)
- [우리 팀 프로젝트 저장소 (2026_osp_project_welcome)](https://github.com/devleeji/2026_osp_project_welcome)
