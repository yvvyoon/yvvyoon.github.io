---
title: 'Django Template Engine'
date: 2020-01-03 16:55:00 -0400
categories: Django
---

곧 투입될 프로젝트에서 Django 프레임워크 기반 프론트엔드 서버를 구축하게 되었다. 백엔드 개발이 아니다보니 당연하게 비즈니스 로직 쪽은 건들 것이 거의 없고 아마 퍼블리셔 측에서 전달해주는 정적 파일들을 Django 템플릿 언어로 컨버전하는 것이 주 업무가 될 것 같다.

Django로 게시판을 만들어 봤기 때문에 템플릿 언어가 완전히 낯선 아이가 아니다. 그러나 프로젝트 투입에 앞서 더 깊은 내용을 공부하고자 이번 포스트를 작성하고자 한다.

Django 공식 문서를 참조하여 공부한 내용을 바탕으로 작성하려 한다.

## Django Template Language (DTL)

Django로 웹 프로젝트를 제작하는 경우 여러 템플릿 엔진 중에서 원하는 엔진을 선택해서 진행할 수 있다. 기본적으로 Django가 빌트인으로 제공하는, `Django 템플릿 언어(DTL, Django Template Language)`가 있고 Flask 앱을 만들면서 사용해봤던 `Jinja2`, 그리고 써드파티 템플릿 언어도 사용이 가능하다.

DTL에 일부 특이한 점과 





> *<https://docs.djangoproject.com/en/3.0/topics/templates/#support-for-template-engines>*