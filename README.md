# 어서오소 Tech Blog

GitHub Pages로 만든 팀 기술 블로그입니다. 주소: https://injipa.github.io/2026_osp_welcome_blog/

## 글 쓰는 방법 (GitHub 웹에서)
1. `_posts` 폴더에서 **내 이름이 author인 파일**을 엽니다.
2. 오른쪽 위 연필(✏️) 아이콘 → 내용 작성
   - 맨 위 `---` 사이(제목, 작성자 등)는 **title만** 필요하면 고치고 나머지는 그대로 두세요.
   - `> ✏️ 작성 중입니다.` 줄은 다 쓰면 지워 주세요.
   - 300단어 이상, 강의 11 "What Makes a Good Post / Common Structure" 참고
3. **Commit changes** → `Commit directly to the main branch` → Commit
4. 1~2분 뒤 블로그에 반영됩니다. (Actions 탭에서 초록 체크 확인)

## 이미지 넣기
- `assets/images/` 폴더에 업로드(Add file → Upload files)
- 글에서: `![설명]({{ "/assets/images/파일이름.png" | relative_url }})`

## Flask/Jinja 코드 넣을 때 (중요)
블로그(Jekyll)도 `{{ }}`, `{% %}`를 자기 문법으로 읽어서, 그냥 넣으면 **그 부분이 사라져요.** 코드 블록을 아래처럼 감싸 주세요.

    {% raw %}
    ```html
    <a href="{{ url_for('login') }}">로그인</a>
    ```
    {% endraw %}

## 주의
- 다른 사람 파일은 수정하지 않기
- 파일 이름(날짜-영문이름.md)은 바꾸지 않기 (바꾸면 주소가 바뀌어요)
