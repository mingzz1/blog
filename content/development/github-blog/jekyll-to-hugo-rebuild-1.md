---
title: "[Hugo] Jekyll to Hugo 블로그 이전기"
date: 2021-07-12
draft: false
category: [development]
subcategories: [github-blog]
tags: [Hugo, Github Blog]
---

이 전에도 여러 번 거론했다시피, 내 블로그는 원래 `Jekyll`로 만들었다.
그런데 블로그를 개설한지 어언 3년...
글이 점점 많이지다보니 새로운 포스팅을 썼을 때와, 디자인을 변경했을 때 커밋 후 블로그에 실제로 반영될때까지 속도가 너무 느렸다.
그래서 고민하다가 블로그를 대대적으로 리뉴얼 하면서 `Hugo`로 이전까지 하게 되었다. 

<!--more-->

날짜를 미래의 날짜로 하면 글이 안뜸
draft true 로 해도 글이 안뜸

directory 구조를 만들기 위해서는 content 아래에 폴더를 만들고, 각 폴더에 _index.md를 넣어야 함.
_index.md는 비어있어도 되지만, 내 블로그에서 각 카테고리 눌렀을 때 나오는 `Development/Github Blog` 같은걸 넣으려면 최소한의 information 란은 적어주는 것이 좋음

자동으로 빌드되지 않기 때문에 repository가 2개 필요함

테마는 직접 만들 수 있음 -> 변수는 이런식으로 사용, 변수 호출하는 법 등

image 경로는 변수로 생성해서 넣을 수 있음
permalink는 content 아래의 폴더 별로 지정할 수 있음

scss도 사용 가능함 -> 단, 기본 테마 틀에서는 불가능하고 assets/sass를 만들어야 가능함

마지막에 올리기 위해서는 hugo build -t 테마명  등 submodule 먼저 지정하고 해야 함

로컬에서 돌리기 위해서는 이렇게 해야함. -> 로컬 세팅 편...


