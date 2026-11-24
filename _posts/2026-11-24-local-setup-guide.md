---
layout: post
title: "Windows에서 Ewha Market 실행하기: Flask와 Pyrebase4 설치부터 Firebase 연결까지"
author: 장예희
type: Guide
order: 5
---

## Context

저희 팀은 이화대동제 축제 부스 판매 웹 **Ewha Market**을 Flask와 Firebase Realtime Database로 만들고 있습니다. 다섯 명이 각자 브랜치에서 기능을 만들고, 저는 저장소 관리와 DB를 맡아 매주 팀원들의 코드를 main에 합쳤습니다.

그런데 매주 코드보다 **"제 컴퓨터에서는 실행이 안 돼요"**에 더 많은 시간을 썼습니다. 누군가는 `ModuleNotFoundError`를 보고, 누군가는 브라우저에서 "사이트에 연결할 수 없음"을 보고, 누군가는 Firebase 키 파일 때문에 서버가 켜지지 않았습니다. 대부분 같은 몇 가지 원인이었습니다.

## Goal

이 글의 목표는 **Windows에서 Ewha Market 저장소를 처음 받은 사람이, 이 글만 보고 서버를 실행하고 Firebase에 데이터가 저장되는 것까지 확인하는 것**입니다.

**준비물**
- Python 3
- Git for Windows (Git Bash 포함)
- 팀에서 공유받은 Firebase 설정 파일 `firebase_auth.json`

## Steps

### 1. 저장소 받기

```bash
mkdir -p /c/dev
cd /c/dev
git clone https://github.com/devleeji/2026_osp_project_welcome.git
cd 2026_osp_project_welcome
```

Git Bash에서 `C:\dev` 폴더를 만들고 그 안에 저장소를 받습니다. **바탕화면이나 문서 폴더처럼 OneDrive가 동기화하는 곳은 피하는 것이 좋습니다.** 저는 처음에 바탕화면에 받았다가 OneDrive 동기화 오류(빨간 X 표시)가 계속 생겼습니다. `.git` 폴더 안의 작은 파일들을 OneDrive가 동기화하려다 충돌하기 때문입니다.

### 2. 가상환경 만들고 켜기

```bash
python -m venv venv
```

프로젝트 전용 Python 환경(`venv` 폴더)을 만드는 명령입니다. 이 환경에 설치한 패키지는 다른 프로젝트와 섞이지 않습니다.

가상환경을 **켜는 명령은 터미널마다 다릅니다.** 저희 팀에서 가장 많이 헷갈린 부분입니다.

| 터미널 | 줄 앞 표시 | 켜는 명령 |
|---|---|---|
| Git Bash | `$` | `source venv/Scripts/activate` |
| PowerShell (VS Code 기본 터미널) | `PS` | `venv\Scripts\activate` |

줄 앞에 `(venv)`가 붙으면 켜진 것입니다. PowerShell에서 "스크립트를 실행할 수 없다"는 오류가 나면 아래 명령을 한 번 실행한 뒤 다시 켭니다.

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

`venv` 폴더는 각자 컴퓨터에서 만드는 것이므로 GitHub에 올리지 않습니다. 저장소의 `.gitignore`에 `venv/`가 이미 들어 있습니다.

### 3. Flask와 Pyrebase4 설치

```bash
pip install flask pyrebase4
```

`flask`는 웹 서버, `pyrebase4`는 Python에서 Firebase를 쓰게 해 주는 라이브러리입니다. 원래 라이브러리 이름은 pyrebase인데 업데이트가 끊겨서, 이어서 관리되는 `pyrebase4`를 설치합니다. 코드에서는 똑같이 `import pyrebase`로 불러옵니다.

**반드시 `(venv)`가 보이는 상태에서 설치해야 합니다.** 가상환경을 끈 채로 설치하면 컴퓨터 전체 Python에 설치되고, 정작 가상환경에서 실행할 때는 패키지를 찾지 못합니다. 설치 로그의 경로가 `...\2026_osp_project_welcome\venv\...`인지 보면 확인할 수 있습니다.

### 4. Firebase 설정 파일 넣기

Ewha Market은 상품, 회원, 리뷰, 좋아요를 Firebase Realtime Database에 저장합니다. 앱이 DB에 접속하려면 Firebase 웹 앱 설정값이 필요한데, 이 값은 **공개 저장소에 올리면 안 되는 정보**라서 저장소에 들어 있지 않습니다.

레포 최상위에 `authentication` 폴더를 만들고, 팀에서 받은 `firebase_auth.json`을 넣습니다.

```
2026_osp_project_welcome/
├── app.py
├── database.py
└── authentication/
    └── firebase_auth.json   ← GitHub에 올라가지 않음
```

`database.py`는 이 파일을 읽어서 Firebase에 연결합니다.

```python
with open('./authentication/firebase_auth.json') as f:
    config = json.load(f)
firebase = pyrebase.initialize_app(config)
self.db = firebase.database()
```

`json.load`로 읽기 때문에 파일은 **반드시 JSON 형식**이어야 합니다. Firebase 콘솔이 보여 주는 설정 코드는 자바스크립트라서 그대로 붙여넣으면 안 됩니다.

```json
{
  "apiKey": "...",
  "authDomain": "...",
  "databaseURL": "...",
  "projectId": "...",
  "storageBucket": "...",
  "messagingSenderId": "...",
  "appId": "..."
}
```

키 이름(`apiKey` 등)에도 **큰따옴표**를 붙이고, `const firebaseConfig =` 줄과 마지막 세미콜론은 지워야 합니다. 저는 이걸 놓쳐서 `JSONDecodeError: Expecting property name enclosed in double quotes`를 만났습니다.

마지막으로 `git status`를 실행했을 때 `authentication/`이 목록에 **보이지 않는지** 확인합니다. 보인다면 `.gitignore`가 아직 이 폴더를 막지 못하는 것이므로 `git add .`를 하면 안 됩니다.

### 5. 서버 실행

```bash
python app.py
```

`Running on http://127.0.0.1:5000`이 나오면 브라우저에서 그 주소를 엽니다. 이 페이지는 **터미널에서 서버가 켜져 있는 동안에만** 열립니다. `Ctrl+C`를 누르거나 터미널을 닫으면 "사이트에 연결할 수 없음"이 뜨는데, 고장이 아니라 서버가 꺼진 것입니다.

### 6. 테스트 데이터 넣기 (선택)

DB가 비어 있으면 상품 목록이 빈 화면이라 기능을 확인하기 어렵습니다. 저장소의 스크립트로 테스트 상품을 한 번에 넣을 수 있습니다.

```bash
python tools/seed_items.py
```

같은 이름의 상품이 이미 있으면 건너뛰므로 여러 번 실행해도 중복되지 않습니다. 이 스크립트는 저장소에 포함된 `static/illustrations/` 그림을 쓰기 때문에 **누구 컴퓨터에서 실행해도 사진이 보입니다.** 반대로 상품 등록 페이지에서 직접 올린 사진은 올린 사람 컴퓨터에만 저장되어 다른 팀원 화면에서는 깨져 보입니다.

### 7. 팀원 코드를 받은 뒤 다시 실행할 때

```bash
git pull origin main
source venv/Scripts/activate
python app.py
```

새 터미널을 열 때마다 가상환경은 꺼진 상태이므로 **켜는 것부터** 다시 해야 합니다. 팀원이 새 패키지를 추가했다면 3단계의 설치 명령도 다시 실행합니다.

## Result

회원가입 후 로그인하면 헤더에 닉네임이 표시되고, 메인 화면에서 상품 목록이 보입니다.

![로컬에서 실행한 Ewha Market 상품 목록]({{ "/assets/images/guide-result-list.png" | relative_url }})

상품을 등록하거나 좋아요를 누르면 Firebase 콘솔의 데이터 탭에서 바로 값이 바뀌는 것을 확인할 수 있습니다.

![Firebase 콘솔에 저장된 상품 데이터]({{ "/assets/images/guide-result-firebase.png" | relative_url }})

**자주 만나는 오류 정리**

| 증상 | 원인 | 해결 |
|---|---|---|
| `ModuleNotFoundError: No module named 'flask'` | 가상환경을 켜지 않고 실행 | 2단계로 켠 뒤 다시 실행 |
| `cd /c/dev/...` 경로를 찾을 수 없음 | PowerShell에서 Git Bash 명령 사용 | 줄 앞 `PS`/`$`로 터미널을 확인하고 맞는 명령 사용 |
| `JSONDecodeError` | 설정 파일이 JSON 형식이 아님 | 4단계처럼 큰따옴표·세미콜론 확인 |
| 브라우저 "사이트에 연결할 수 없음" | 서버가 꺼져 있음 | `python app.py` 다시 실행, 터미널 닫지 않기 |
| 코드를 고쳤는데 화면이 그대로 | 브라우저가 예전 CSS를 기억 | `Ctrl + F5`로 강력 새로고침 |

## What I learned

실행 환경 문제는 대부분 **"지금 어느 터미널에서, 어느 Python으로, 어느 폴더에서 실행하고 있는가"**를 확인하면 풀렸습니다. 그리고 한 번 겪은 사람이 순서를 정리해 두면 다음 사람의 한 시간을 아껴 줍니다. 이 순서를 팀에 공유한 뒤로는 새 기능을 합칠 때마다 오던 "안 돼요" 메시지가 확실히 줄었습니다.

## References

- [Flask Installation (공식 문서)](https://flask.palletsprojects.com/en/stable/installation/)
- [Python venv (공식 문서)](https://docs.python.org/3/library/venv.html)
- [Pyrebase4 (PyPI)](https://pypi.org/project/Pyrebase4/)
- [Firebase: 웹 앱에 Firebase 추가](https://firebase.google.com/docs/web/setup)
- [Ewha Market 저장소](https://github.com/devleeji/2026_osp_project_welcome)
