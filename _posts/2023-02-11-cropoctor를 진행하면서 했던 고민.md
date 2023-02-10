---
title: cropdoctor프로젝트를 하며 했던 고민
categories:
- project
# layout: single
date: 2023-02-11
toc: true
toc_label: CATEGORY
toc_sticky: true
# author_profile: true
---

그냥 프로젝트하면서 이런 문제가 있었고 어떻게 해결했는지 가볍게 작성해봅니다!!

---

## 문제

내가 맡은 api는 사용자가 올린 작물 사진을 ai에 입력 후 결과(결과사진, 병명)을 response하는 것이었다.

<p align = "center">결과사진은 작물과 질병 위치를 잡아준다.<img src='/assets/images/posts/2023-02-11/1.png' width="500"/></p>



가장 큰 고민은 위의 결과사진 처리 방법이었는데 ai담당 팀원과 같이 ai 모델의 설정들을 만져가며 2가지 방법을 생각했다.

첫번째는 작물과 질병을 잡아주는 네모박스 x,y 좌표값을 받아  기존의 사진에 표시하는 방법,

두번째는 결과사진을 s3에 올려 s3 url을 보내는 방법이다.

우리는 두번째 방법을 선택했는데 이유는 결과사진이 게속 로컬 `runs/detect/exp`위치에 저장되었는데 그때는 아무리 찾아도 결과사진을 저장되지 않게 하는 법을 찾지 못했다…

첫번째 방법을 사용하면 s3에 사용자가 업로드한 사진만 올려도 됐었기에 첫번째 방법을 사용하고 싶었지만 두번쨰 방법은 자동으로 저장되는 사진을 s3에 올리기만 하고 로컬 경로로 삭제만하면 됐었기에 두번째 방법으로 구현했다.

두번째 방법으로 기능구현 후 s3에도 잘 올라가고 1차 배포 후에도 정상작동이 되는 걸 확인하고 안심하고 있었다.

하지만 2차 배포 후에 자꾸 진단이 안되는 문제가 발생했다.

aws ec2서버로 직접 들어가 확인해보니 `runs/detect/exp`에 저장된 결과사진이 지워지기도 전에 다른 요청이 들어와 `runs/detect/exp1,2,3` 이렇게 폴더가 생성되어 결과사진이 저장되고 exp폴더로 설정한 경로에 지울 사진을 찾지 못해 오류가 나고 있었다.(이게 동시성 문제 그런건가??)

문제를 exp1, exp2, .. 폴더가 추가로 생성되어 저장되는 것으로 생각했고 ai 모델 설정을 찾아봤는데 의외로 빨리 해결할 수 있었다. 바로 ai모델의 `exist_ok`설정을 False에서 True로 바꾸었더니 같은 상황에서도 exp1,2, .. 폴더가 생성되지 않고 exp폴더 안에서 여러 결과사진들이 만들어졌다.

이렇게만 변경해줘도 문제가 해결되지만 경로와 파일이름으로 삭제하는 방법은 또 다른 변수가 있을까봐 불안했었고, 밑의 함수를 이용하여 exp폴더에 생성된지 5분이 지난 파일을 지워주는 방법으로 수정했다.

프로젝트를 발표하고 서버를 내릴 때까지 문제없이 동작되었다.

```python
def delete_old_files(path_target, days_elapsed):
    for f in os.listdir(path_target): 
        f = os.path.join(path_target, f)
        if os.path.isfile(f): 
            timestamp_now = datetime.now().timestamp()
            is_old = os.stat(f).st_mtime < timestamp_now - (days_elapsed * 60)
            if is_old:
                try:
                    os.remove(f)
                    print(f, 'is deleted')
                except OSError: 
                    print(f, 'can not delete')

delete_old_files(path_target=Path.joinpath({폴더 경로}), days_elapsed=5)
```