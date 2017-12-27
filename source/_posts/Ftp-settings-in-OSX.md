---
title: Anonymous FTP settings in OSX
categories:
  - Tech
tags:
  - ftp
  - mac
  - osx
  - settings
abbrlink: 5646
date: 2015-09-03 17:00:00
---

## 익명 FTP 설정
> 설정파일 : /etc/ftpd.conf

1. ftp 서비스 폴더 생성 (/Users/ftp 폴더 생성. 권한 755)
2. ftpd.conf에 한 줄 추가 => chroot GUEST /Users/ftp 추가

## ftp 구동 쉘 스크립트
### 서버 open
{% gist da0867dea84e22eae50f ftpstart.sh %}
### 서버 close
{% gist c17b0acb2e0016653884 ftpstop.sh %}
