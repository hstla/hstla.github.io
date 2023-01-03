---
title: ubuntu conda 명령어 오류
categories:
- Error
# layout: single
date: 2023-01-03
toc: true
toc_label: CATEGORY
toc_sticky: true
# author_profile: true
---

서버에 부하가 얼마나 걸리는지 테스트하기 aws EC2를 사용하여 서버를 만들었습니다. 
    
nginx와 Anaconda를 설치하여 테스트를 성공적으로 진행 한 후 
다음날 인스턴스에 접속하여 conda명령어를 쳤더니 다음과 같은 오류가 났습니다.
    
<p align = "center"><img src='/assets/images/posts/2023-01-03/1.png' width="300"/></p>
    
git clone으로 받은 프로젝트 파일들은 그대로지만 신기하게 conda만 사라졌습니다.
    
1~2시간의 삽질 끝에 원인을 찾지못해 인스턴스를 삭제했고, 다른 인스턴스를 새로 생성하여 테스트를 진행했습니다.
    
새로운 인스턴스에 conda를 설치하면서 찾아보니 conda환경변수를 설정하지 않아서 생기는 오류였습니다.
    
설치할 때만 설정하면 되는지 알았는데 원격서버에 접속할 때마다 일일이 밑의 명령어를 쳐야 conda가 실행되었습니다. 
    
```bash
sudo su    // 루트계정으로 접속한 후 
source ~/anaconda3/etc/profile.d/conda.sh    //conda환경변수설정
```
    
문제는 해결되었지만 일일이 명령어를 작성하기 힘들어 자동으로 환경변수설정해주는 방법을 보면 올리겠습니다.