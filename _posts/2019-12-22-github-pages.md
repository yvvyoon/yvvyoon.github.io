---
title: 'GitHub Pages 제작기'
date: 2019-12-22 16:26:00 -0400
categories: Essay
---

미루고 미루다가 드디어 블로그를 만들어보려고 한다. 항상 GitHub 1일 0.5커밋(?)을 유지하기 위해 노력은 했다만 점점 사용하다보니 소스코드를 남기기보다는 에세이 성격을 뿜뿜하고 있는 것 같아서 GitHub에서의 제약을 느끼고 있다. 항상 모든 내용을 README에 올려놓을 순 없으니...

학원 커리큘럼 내에서 AWS S3에 배포하여 간단하게 만들어보긴 했으나 사실 없다고 해도 무방하다. 이번에는 그 유명하다는 `Jekyll` 테마로 `깐지나게` 만들어 보자ㅏㅏㅏ.

옷이나 여러 아이템에서 약간의 유니크함을 추구하지만 테마 같은 경우에는 대세를 따르는 게 덜 피곤한 방법이다. `minimal-mistakes`가 좋아요 수가 제일 많더라. 소개글을 보니 그 안에서도 여러 스킨으로 나뉘어져 있다.

<br>

> _https://github.com/mmistakes/minimal-mistakes_

<br>

### 기본 설정

`Jekyll` 테마에 공통적으로 사용되는 설정 파일인 `_config.yml` 파일을 그대로 가져온다. 위 링크에 소스 코드가 있고, 가져 왔다면 `remote_theme` 속성의 주석을 해제해주자. GitHub repository 이름의 형식 그대로이다.

```yml
remote_theme: 'mmistakes/minimal-mistakes'
```

<br>

스킨도 골라보자. 무조건 깔끔해야 하기 때문에 `contrast`로 정했다.

```yaml
minimal_mistakes_skin: 'contrast'
# "air", "aqua", "contrast", "dark", "dirt", "neon", "mint", "plum", "sunrise"
```

<br>

그 다음 테마를 적용하기 위해 내 블로그와 연결해준다. 프로젝트 별로 구분할 것도 아니므로 `baseurl`은 비운다.

```yaml
url: 'https://yvvyoon.github.io'
baseurl: ''
```

<br>

그 밖에도 많은 설정 옵션들이 있지만, 일단 구축해서 띄워보고 하나씩 입맛대로 바꿔볼 것이다.

<br>

### Index

각 테마마다 필요로 하는 index 파일들이 다르다고 한다. 이 테마에서는 `index.html`을 사용하는데 열어보니 전혀 HTML이 아닌 HTML이다. 당황 :)

일단 똑같이 만들자.

```html
---
layout: home
author_profile: true
---
```

<br>

### \_posts

Jekyll 테마들은 `_posts`라는 디렉토리 아래의 마크다운 파일들을 자동으로 블로그 포스트로 인식한다고 한다. 그래서 업로드 할 파일명 앞에 `/_posts/`를 붙여준다. 일반적으로 파일 이름에 날짜 형식도 붙여준다고 한다.

지금 쓰고 있는 이 글을 바로 테스트로 올릴 거다. 하 벌써 연말이라니...

`_posts/2019-12-22-github-pages.md`

<br>

### front-matter

Jekyll로 올리는 각 블로그 포스트 파일 맨 위에 입력하는 코드이다. Jekyll에서 확인하는 메타 데이터라고 한다.

```
---
title: "Welcome to Jekyll!"
date: 2019-12-22 16:26:00 -0400
categories: jekyll update
---
```

<br>

<img width="778" alt="스크린샷 2019-12-22 오후 4 31 26" src="https://user-images.githubusercontent.com/12066892/71318752-81f3dd00-24d8-11ea-9626-bf08df604029.png">

실제로 블로그 화면에 보이는 제목 같은 메타데이터였구나.

<br>

### 블로그 테스트

이제 커밋하고 `https://yvvyoon.github.io`로 접속해 테스트해보자.
