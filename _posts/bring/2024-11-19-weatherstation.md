---
title: WeatherStation 구조 변경 (2) observer 정보 삭제
author: qoskdud0928
date: 2024-11-19
category: bring
layout: post
---

DB에 저장하지 않고 WeatherStation을 두려던 이유는 장소의 크기, 하루의 길이는 정해져 있기 때문이었다.
옵저버 패턴이 내가 구현하려던 프로젝트에 잘 맞는다는 생각이 들어서 도입했는데 
클래스 구조를 만들고 나니 사용자 수에 따라 메모리에 올라와 있는 객체가 급격히 증가하는 구조였다. 

엑셀의 장소 개수는 3800개였기 때문에 장소마다 24시간의 날씨 정보를 저장하고 
날씨 정보마다 observer를 설정해주니 observer 객체의 수가 최대 3800*24*(사용자수)로 늘어날 수도 있었다.

observer를 등록하지 않고 스케줄 정보를 바탕으로 장소, 시간의 날씨 정보를 가져오는 방식으로 변경했다.

### 수정 전 클래스 다이어그램

[수정 전 클래스 다이어그램](/assets/post-img/ver02-3_UML.jpg)

### 수정 후 클래스 다이어그램

[수정 후 클래스 다이어그램](/assets/post-img/ver02-4_UML.jpg)

