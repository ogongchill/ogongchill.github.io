# 로컬에서 블로그를 빌드하기

### 1. Ruby 설치 확인

```bash
ruby -v
gem -v
```
Ruby와 RubyGems가 정상적으로 설치되어 있어야 함

### 2. bundler 설치
```
gem install bundler
```

### 3. dependency 설치
```
cd your-jekyll-project
bundle install 
```

### 4. 로컬에서 Jekyll 실행
```
bundle exec jekyll serve
```
기본적으로 localhost:4000에서 실행됨  
_config.yml 파일을 수정했을 경우 다시 실행해야 함  


# 게시글을 작성하기

### 머리말

게시글 파일은 _posts 폴더 내에 위치  
게시글은 markdown 형식을 기본으로 지원  
게시글의 파일 이름은 **https://github.com/gzcxadfzcYYYY-MM-DD-TITLE.md** 형태를 지켜야 함  
게시글의 상단에는 아래와 같은 머리말을 작성해야 함  

```
---
title: "Chirpy Theme로 Jekyll 블로그 시작하기"
date: 2023-11-12 17:00:00 +/-TTTT
categories: [Programming, Blogging]
tags: [jekyll, chirpy, blog]
author: {작성자자}
math: true
toc: true
pin: true
image:
  path: thumbnail.png
  alt: image alternative text
---
```
- **categories:** 최대 2개의 항목이 포함  
- **toc:** 는 게시글의 오른쪽에 표시되는 Table of Content  
- **math:**  수학 기능 활성화를 위한 옵션  
- **pin:** 옵션을 통해 한개 또는 여러개의 게시글을 블로그 상단에 고정할 수 있음  
- **image:** path: 에 썸네일 이미지의 경로를 넣어 썸네일을 추가할 수 있음음. 1200 x 630 해상도와 1.91 : 1 비율을 권장
- **author / authors[]** 생략시 orginization계정으로 생성됨.  *_data/authors.yml* 에 작성자 정보 추가 후 설정해야함  

### 수식 추가하기
수학 기능을 활성화 한 후 아래와 같이 수학식을 추가할 수 있음
```
$$ math $$ 와 같은 형식으로 활용합니다.
$$ y = \frac{1}{x} $$
```
 
     

### 이미지 삽입
이미지를 추가하고 너비와 높이를 지정할 수 있으며, 이미지 아래 캡션도 아래와 같이 추가할 수 있음  

이미지의 위치를 지정할 수 있으며, 다크/라이트 모드와 그림자 효과 설정이 가능합니다.

크기 및 캡션 설정

```
![img-description](/path/to/image){: width="700" height="400" }
_Image Caption_
위치 설정
![Desktop View](/assets/img/sample/mockup.png){: .normal }
![Desktop View](/assets/img/sample/mockup.png){: .left }
![Desktop View](/assets/img/sample/mockup.png){: .right }
다크/라이트 모드
![Light mode only](/path/to/light-mode.png){: .light }
![Dark mode only](/path/to/dark-mode.png){: .dark }
그림자 효과
![Desktop View](/assets/img/sample/mockup.png){: .shadow }
```

### 구문 강조  
아래의 문법으로 파일 경로를 강조할 수 있음  
```
`/path/to/a/file.extend`{: .filepath}
```
프롬프트를 추가하여 문구를 강조할 수 있습니다.
tip 대신 info, warning, danger 옵션도 사용 가능  
```
> An example showing the `tip` type prompt.
{: .prompt-tip }
 ```    

### 마크다운 사용하기  
그 외의 사항은 마크다운 문법을 사용하여 작성하면 됨  

# githubpages에 게시하기  
 
*.github/workflows/pages-deploy.yml* 설정에 의해 **main** 브랜치에 푸시하면 자동으로 배포됨  
글을 쓰기 위해서는  
1. _post내부에 YYYY-MM-DD-{제목}.md형식으로 파일 생성
2. 헤더 설정
    ```
    ---
    title: "제목"
    date: 2023-11-12 17:00:00 +/-TTTT
    categories: [Programming, Blogging]
    tags: [jekyll, chirpy, blog]
    ...
    ---
    {본문내용 작성}
    ```
3. main으로 푸시