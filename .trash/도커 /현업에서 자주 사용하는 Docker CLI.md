# 컨테이너 생성/실행

### ✅ 컨테이너 생성 + 실행

> [!info]
> 이미지로 컨테이너 만들고 바로 실행하는 명령. 처음 셋업할 때 제일 자주 씀.

```bash
# 형식
docker run <이미지>[:태그]

# 예시: 포그라운드 실행
docker run nginx  # 현재 터미널에서 실행됨 (Ctrl+C로 종료)
```

#### 백그라운드에서 실행하기

> [!note]
> 포그라운드: 화면에 계속 출력돼서 다른 작업하기 빡셈
> 
> 백그라운드: 터미널이랑 분리돼서 뒤에서 돈다. 계속 다른 명령도 입력 가능

```bash
# 형식
docker run -d [옵션들] <이미지>[:태그]

# 예시
docker run -d --name my-web-server -p 4000:80 nginx
```

- **-d**: 백그라운드(detached)로 돌리기
- **--name**: 컨테이너 이름 붙이기
- **-p 호스트:컨테이너**: 포트 매핑 (예: 호스트 4000 → 컨테이너 80)

---

# 컨테이너 조회/중지/삭제

### ✅ 컨테이너 조회

```bash
# 실행 중인 컨테이너만
docker ps

# 모든 컨테이너 (중지 포함)
docker ps -a
```

- **ps**: 실행 중만
- **-a**: 전부(all)

### ✅ 컨테이너 중지

```bash
docker stop <컨테이너ID|이름>   # 정상 종료
docker kill <컨테이너ID|이름>   # 즉시 강제 종료
```

> [!warning]
> kill은 그냥 바로 죽이는 거라, 웬만하면 stop 먼저 쓰자.

### ✅ 컨테이너 삭제

```bash
# 중지된 특정 컨테이너 삭제
docker rm <컨테이너ID|이름>

# 실행 중인 컨테이너 강제 삭제
docker rm -f <컨테이너ID|이름>

# 중지된 모든 컨테이너 일괄 삭제
docker rm $(docker ps -qa)

# 모든 컨테이너(실행 중 포함) 일괄 강제 삭제
docker rm -f $(docker ps -qa)
```

> [!warning]
> 일괄 삭제는 되돌릴 수 없다. 진짜 다 지워도 되는지 확인하고 실행하자.

---

# 컨테이너 로그 조회

> [!info]
> 문제 생기면 일단 로그부터 본다. 실행 상태 체크/에러 확인은 로그가 답.

### ✅ 로그 확인 기본

```bash
# 모든 로그 출력
docker logs <컨테이너ID|이름>

# 실시간으로 로그 팔로우
docker logs -f <컨테이너ID|이름>
```

### ✅ 필요한 양만 보기

```bash
# 최근 10줄만
docker logs --tail 10 <컨테이너ID|이름>

# 기존 로그는 제외하고 새로 생기는 로그만 실시간 보기
docker logs --tail 0 -f <컨테이너ID|이름>
```

- **-f**: follow (실시간)
- **--tail N**: 끝에서 N줄만 보기

---

# 실행 중인 컨테이너 내부 접속

### ✅ 쉘 접속

```bash
# bash가 있는 경우
docker exec -it <컨테이너ID|이름> bash

# bash가 없으면 sh 사용
docker exec -it <컨테이너ID|이름> sh
```

```bash
# 예시: nginx 컨테이너 접근 후 설정 확인
docker run -d --name web -p 80:80 nginx
docker exec -it web bash
ls
cd /etc/nginx
cat nginx.conf
```

- 나가려면 `exit` 또는 `Ctrl + D`
- **-it**: 대화형 TTY 붙여서 계속 입력 가능