---
layout: post
title: "Ewha Market 로컬에서 실행하기: clone부터 Firebase 연결까지"
author: 장예희
type: Guide
order: 5
---

## 들어가며

팀 프로젝트를 하다 보면 코드보다 **"내 컴퓨터에서 실행이 안 돼요"** 때문에 시간을 더 많이 쓰게 됩니다. 저희 팀도 매주 새 기능을 합칠 때마다 누군가는 `ModuleNotFoundError`를 보고, 누군가는 브라우저에서 "사이트에 연결할 수 없음"을 봤습니다.

이 글은 어서오소 팀의 **Ewha Market**(Flask + Firebase)을 처음 받은 사람이 **자기 컴퓨터에서 실행하고, 테스트 데이터까지 넣는 과정**을 정리한 가이드입니다. 저는 팀에서 저장소 관리와 DB를 맡아 팀원들의 실행 환경 문제를 가장 많이 같이 해결했는데, 그때 실제로 막혔던 지점들도 함께 적었습니다.

**준비물**
- Windows 기준 (macOS도 명령어 일부만 다름)
- Python 3, Git for Windows (Git Bash 포함)
- 팀에서 공유받은 Firebase 키 파일 `firebase_auth.json`

## 1. 저장소 받기

```bash
mkdir -p /c/dev
cd /c/dev
git clone https://github.com/devleeji/2026_osp_project_welcome.git
cd 2026_osp_project_welcome
```

**OneDrive 폴더 안(바탕화면, 문서 등)에 clone하지 않는 것을 추천합니다.** 저는 처음에 바탕화면에 받았다가 OneDrive 동기화 오류(빨간 X)가 계속 떴습니다. `.git` 폴더 안의 작은 파일들이 동기화와 충돌하기 때문입니다. `C:\dev`처럼 동기화되지 않는 곳에 두면 깔끔합니다.

## 2. 가상환경 만들고 켜기

```bash
python -m venv venv
```

가상환경을 켜는 명령어는 **터미널 종류마다 다릅니다.** 제가 가장 많이 헷갈린 부분입니다.

| 터미널 | 켜는 명령어 |
|---|---|
| Git Bash | `source venv/Scripts/activate` |
| PowerShell (VS Code 기본 터미널) | `venv\Scripts\activate` |
| macOS 터미널 | `source venv/bin/activate` |

줄 앞에 `(venv)`가 붙으면 성공입니다. PowerShell에서 "스크립트를 실행할 수 없다"는 오류가 나면 `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`를 한 번 실행한 뒤 다시 켜면 됩니다.

`venv` 폴더는 각자 컴퓨터에서 만드는 것이라 GitHub에 올리지 않습니다. 저장소의 `.gitignore`에 이미 `venv/`가 들어 있습니다.

## 3. 패키지 설치

```bash
pip install flask pyrebase4
```

**반드시 `(venv)`가 켜진 상태에서** 설치해야 합니다. 가상환경을 끈 채로 설치하면 컴퓨터 전체 Python에 설치되고, 정작 실행할 때는 가상환경 안에서 패키지를 찾지 못합니다. 설치 로그에 나오는 경로가 `...\2026_osp_project_welcome\venv\...`인지 보면 확인할 수 있습니다.

## 4. Firebase 키 파일 넣기

Ewha Market은 Firebase Realtime Database에 상품, 회원, 리뷰, 좋아요를 저장합니다. 앱이 DB에 접속하려면 Firebase 웹 앱 설정값이 필요한데, 이 값은 **GitHub에 올리면 안 되는 열쇠**라서 저장소에 들어 있지 않습니다.

1. 레포 최상위에 `authentication` 폴더를 만듭니다.
2. 팀에서 받은 `firebase_auth.json`을 그 안에 넣습니다.

```
2026_osp_project_welcome/
├── app.py
├── database.py
└── authentication/
    └── firebase_auth.json   ← GitHub에 올라가지 않음
```

파일 내용은 Firebase 콘솔의 웹 앱 설정을 **JSON 형식**으로 옮긴 것입니다.

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

Firebase 콘솔이 보여 주는 코드는 자바스크립트라서 그대로 붙여넣으면 안 됩니다. **키 이름에 큰따옴표**를 붙이고, `const firebaseConfig =` 같은 줄과 마지막 세미콜론을 지워야 합니다. 저는 이걸 놓쳐서 `JSONDecodeError: Expecting property name enclosed in double quotes`를 만났습니다.

`.gitignore`에 `authentication/`이 들어 있는지도 꼭 확인하세요. `git status`를 했을 때 `authentication/`이 목록에 보이면 아직 막히지 않은 것이니 `git add .`를 하면 안 됩니다.

## 5. 실행

```bash
python app.py
```

`Running on http://127.0.0.1:5000`이 나오면 브라우저에서 그 주소를 엽니다. 메인 화면에서 바로 상품 목록이 보입니다.

이 페이지는 **터미널에서 서버가 켜져 있는 동안에만** 열립니다. `Ctrl+C`를 누르거나 터미널 창을 닫으면 "사이트에 연결할 수 없음"이 뜨는데, 고장이 아니라 서버가 꺼진 것입니다.

## 6. 테스트 데이터 넣기 (선택)

DB가 비어 있으면 목록이 빈 화면이라 기능을 확인하기 어렵습니다. 저장소의 스크립트로 테스트 상품을 한 번에 넣을 수 있습니다.

```bash
python tools/seed_items.py
```

이미 같은 이름의 상품이 있으면 건너뛰기 때문에 여러 번 실행해도 중복되지 않습니다. 상품 이미지는 저장소의 `static/illustrations/`에 있는 그림을 쓰기 때문에 **누구 컴퓨터에서 실행해도 사진이 깨지지 않습니다.** 반대로 상품 등록 페이지에서 직접 사진을 올리면 그 사진은 올린 사람 컴퓨터의 `static/images/`에만 저장되어, 다른 팀원 화면에서는 깨져 보입니다.

## 자주 만나는 오류

| 증상 | 원인 | 해결 |
|---|---|---|
| `ModuleNotFoundError: No module named 'flask'` | 가상환경을 안 켜고 실행 | 2단계의 켜는 명령어 실행 후 다시 실행 |
| `cd /c/dev/...` 경로를 찾을 수 없음 | PowerShell에서 Git Bash 명령어 사용 | 줄 앞이 `PS`면 PowerShell, `$`면 Git Bash. 터미널에 맞는 명령어 사용 |
| `JSONDecodeError` | 키 파일이 JSON 형식이 아님 | 4단계처럼 큰따옴표, 세미콜론 확인 |
| 브라우저 "사이트에 연결할 수 없음" | 서버가 꺼져 있음 | `python app.py` 다시 실행, 터미널 닫지 않기 |
| 수정했는데 화면이 그대로 | 브라우저가 예전 CSS를 기억 | `Ctrl + F5`로 강력 새로고침 |

## 정리

실행 전에 이 다섯 가지만 확인하면 대부분 해결됩니다.

- [ ] OneDrive 밖에 clone했는가
- [ ] 줄 앞에 `(venv)`가 보이는가
- [ ] `(venv)` 상태에서 `flask`, `pyrebase4`를 설치했는가
- [ ] `authentication/firebase_auth.json`이 JSON 형식으로 있는가
- [ ] `git status`에 `authentication/`과 `venv/`가 안 보이는가

팀 프로젝트에서 실행 환경 문제는 한 번 겪은 사람이 정리해 두면 다음 사람의 한 시간을 아껴 줍니다. 저희 팀도 이 순서를 공유한 뒤로는 새 기능을 받을 때마다 "안 돼요" 메시지가 확실히 줄었습니다.
