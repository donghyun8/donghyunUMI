# EC2 기반 CPU Server 전처리 서버 

# 1. 전체 프로젝트 관점에서 서버의 역할

데이터 수집기의 목표는 Android에서 실시간으로 모든 것을 처리하는 것이 아니다.

Android는 가능한 한 데이터 수집과 Session 및 Episode 확정에 집중하고,
서버는 수집이 끝난 뒤 데이터 정합성을 확인하고 학습 가능한 representation으로 변환한다.

전체 역할은 다음처럼 분리한다.

```text
Android 
    │
    │ Session 단위 upload
    ▼
CPU EC2 Server
    │
    ├─ ingestion
    ├─ integrity validation
    ├─ raw data preservation
    ├─ timestamp alignment
    ├─ ARCore pose 후처리
    ├─ camera → EE transform
    ├─ episode slicing
    └─ training representation 생성
            │
            ▼
        GPU Server
         │
         └─ Dataset/DataLoader + Policy Training
```

> Android에서 학습용 데이터 X
> Android는 최대한 원본에 가까운 데이터를 수집하는 것이 목표
> CPU 서버가 deterministic한 전처리를 담당
> GPU 서버는 가능한 한 학습에만 집중

---

# 2. 왜 Session 단위 전송인가

수집 데이터는 다음처럼 서로 다른 센서와 스트림으로 구성된다.

```text
RGB Video
Frame Timestamp
Accelerometer
Gyroscope
Rotation Vector
ARCore Pose
Episode Marker
Metadata
```

## 2.1 네트워크가 수집 루프에 개입하지 않는다

실시간 upload를 하면 네트워크 지연이나 패킷 손실이 데이터 수집에 영향을 줄 수 있다.

Session 단위 전송 구조에서는 네트워크 상태와 데이터 수집 안정성을 분리할 수 있다.

## 2.2 Continuous timeline을 보존할 수 있다

Episode마다 recording 파일을 끊는 대신,

```text
Session
 ├─ Episode 0
 ├─ Episode 1
 └─ Episode 2
```

형태로 전체 timeline을 유지한다.

Episode는 파일 단위가 아니라 episodes.csv 등의 marker로 표현된다.

이 방식은 이후 서버에서 episode를 다시 자르거나 timestamp 기준으로 센서와 영상을 정렬할 때 훨씬 유연하다고 생각했다

---

# 3. 현재 EC2 서버 환경

현재 사용 중인 EC2 환경

```text
OS: Ubuntu 24.04.4 LTS
Python: 3.12.3
Nginx: 1.24.0
Private IP: 172.26.10.191
Public IP: 43.201.252.170
```

자원

```text
Disk: 약 309 GB
Free: 약 307 GB

RAM: 15 GiB
Available: 약 14 GiB
```

현재 프로젝트 규모의 데이터 수집 및 CPU 전처리 서버로는 충분하다.

---

# 4. 네트워크 구조

현재 서버는 다음처럼 구성되어 있다.

```text
Client
    │
    │ HTTP :80
    ▼
43.201.252.170
    │
    ▼
AWS Network / Security Group
    │
    ▼
UFW
    │
    ▼
Nginx :80
    │
    ▼
FastAPI / Uvicorn
127.0.0.1:8000
```

실제 listen 상태:

```text
0.0.0.0:80        → Nginx
127.0.0.1:8000    → Uvicorn
```

---

# 5. FastAPI 8000 포트를 직접 열지 않은 이유

FastAPI/Uvicorn은 다음처럼 실행한다.

```text
127.0.0.1:8000
```

따라서 EC2 내부에서만 접근 가능하다.

외부 요청은 항상:

```text
Client
 ↓
Nginx
 ↓
FastAPI
```

순서를 거친다.

장점
- 외부 노출 포트를 80/443으로 제한
- 대용량 upload 관련 설정을 Nginx에서 관리 가능
- HTTPS termination을 Nginx에서 처리 가능
- 향후 rate limit, access log, reverse proxy 설정 확장 가능
- FastAPI 자체를 인터넷에 직접 노출하지 않아도 됨

즉, Nginx는 단순 proxy가 아니라 외부와 서버 사이의 경계 역할을 한다.

---

# 6. UFW 구성 의도

기본 서버 UFW는 ssh에 대한 22번 port만 열려 있었고,
nginx로 처리할 80번 port만 추가로 열었다.

```text
Default incoming: deny
-> 인바운드를 기본적으로 전부 차단하고, 명시적으로 허용된 것만 받는다

22/tcp  ALLOW
80/tcp  ALLOW
```
FastAPI는 localhost에만 bind되어 있으므로 외부 요청은 반드시 Nginx를 거친다.

향후 HTTPS를 적용하면 최종적으로,

```text
22   SSH
80   HTTP → HTTPS redirect 
443  HTTPS
```

---

# 7. proxy_request_buffering off

Session에는 MP4 파일이 포함되기 때문에 upload body가 매우 커질 수 있다

기본적인 reverse proxy buffering 구조에서는

```text
Client
 ↓
Nginx가 전체 request body 수신
 ↓
임시 저장
 ↓
FastAPI 전달
```

처럼 동작할 수 있다.

현재는,

```nginx
proxy_request_buffering off;
```

를 사용하여 가능한 한

```text
Client
 ↓
Nginx
 ↓
FastAPI
 ↓
Disk
```

불필요한 buffering을 줄이기 위해서 streaming하도록 구성했다.

또한 Session 단위 대용량 upload를 위해 `client_max_body_size`도 충분히 크게 설정한다.

---

# 8. 저장 구조

현재 서버에서는 raw 데이터를 다음처럼 분리한다.

```text
/srv/
├── incoming/
└── sessions/
```

## /srv/incoming

업로드 중이며 아직 검증되지 않은 임시 데이터

## /srv/sessions

모든 validation을 통과한 최종 raw Session 데이터

```text
/srv/sessions/<session_id>/
```

### 왜 바로 `/srv/sessions`에 쓰지 않는가

업로드 도중 네트워크가 끊겼을 때 불완전한 Session이 정상 dataset처럼 보이는 것을 막기 위해서다.

정상 흐름:

```text
Upload
  ↓
/srv/incoming
  ↓
Validation
  ↓
모두 정상
  ↓
Atomic Rename
  ↓
/srv/sessions/<session_id>
```

`/srv/incoming`과 `/srv/sessions`는 당연히 같은 filesystem 위에 존재한다.

같은 filesystem 내부에서는 `rename()`을 이용해 데이터 전체를 다시 복사하지 않고,
filesystem namespace 상의 디렉터리 엔트리 관계를 변경하는 방식으로 경로를 옮길 수 있다.

개념적으로는 다음과 비슷하다.

```text
이 디렉터리 엔트리는 이제
/srv/incoming 아래가 아니라
/srv/sessions 아래에 속한다.
```

즉, 파일 데이터 블록 자체를 다시 쓰는 것이 아니라,
해당 inode 또는 디렉터리를 어떤 pathname 아래에서 참조할지를 바꾸는 쪽에 가깝다.

그래서 Session 전체 크기가 5GB, 10GB라고 해도
같은 filesystem 내부에서 rename을 사용하면 5GB 또는 10GB 데이터를 복사하지 않고도 빠른 작업 속도가 가능하다.

오랜만에 시스템 프로그래밍 영역의 개념을 접한 김에 기존 개념과 rename이 기존에 알던 개념에서 어떻게 해석되는지 정리해보려 한다.
---

## 어느 레벨의 개념인가

현재 짠 애플리케이션 코드에서는 다음처럼 보일 수 있다.

```python
stage.rename(final)
```

하지만 `rename()`은 FastAPI의 개념도 아니고, Python의 기능도 아니다.

본질적으로는 **OS 커널의 filesystem / VFS 레벨 연산**이다.

계층을 단순화하면 다음과 같다.

```text
Application
  ↓
Python Path.rename()
  ↓
OS / libc interface
  ↓
rename / renameat / renameat2 syscall
  ↓
Linux VFS
  ↓
ext4 / XFS / Btrfs 등의 실제 filesystem 구현
  ↓
inode / directory entry / filesystem metadata 갱신
```

즉 Python의 `Path.rename()`은 사용자에게 제공되는 인터페이스이고,
실제 동작은 syscall을 통해 Linux 커널의 VFS/filesystem rename 연산을 이용한다.

---

## inode와 directory entry 관점

Linux/Unix filesystem 관련 아키텍처를 되새겨보면,

```text
파일 데이터 블록
      ↑
    inode
      ↑
directory entry
      ↑
filename / pathname
```

예를 들어:

```text
/srv/incoming/abc/main_rgb.mp4
```

라는 경로는 결국 특정 inode를 찾아가는 이름이다.

같은 filesystem 내부 rename은 보통 파일 데이터 블록 자체를 새 위치로 복사하는 것이 아니라,
어떤 directory entry가 어떤 inode를 가리키는지,
또 그 entry가 어느 parent directory 아래 존재하는지를 갱신하는 쪽에 가깝다.

ex:

```text
Before

/srv/incoming
    │
    └── "session_A"
            │
            ▼
      directory inode 5001
            │
            ├── main_rgb.mp4
            ├── imu.csv
            └── ...

/srv/sessions
    └── 없음
```

rename 후:

```text
After

/srv/incoming
    └── 없음

/srv/sessions
    │
    └── "session_A"
            │
            ▼
      directory inode 5001
            │
            ├── main_rgb.mp4
            ├── imu.csv
            └── ...
```

핵심은,

```text
파일 자체는 그대로
경로 / namespace상의 연결 관계만 변경
```

---

## rename이 빠른 이유

예를 들어 5GB짜리 디렉터리를

```text
/srv/incoming/a
```

에서:

```text
/srv/sessions/a
```

로 옮긴다고 하자.

같은 filesystem 내부라면 내부의 5GB 파일 데이터를 다시 읽고 쓰지 않는다.

즉 비용이,

```text
파일 크기에 비례하는 대용량 copy
```

가 아니라

```text
filesystem metadata 갱신
```

쪽에 가깝다.

그래서 같은 filesystem 내부 rename은
대형 Session을 final 위치로 publish하기에 매우 적합하다.

---

## Atomic

Filesystem의 `rename()`은 atomic operation이다.

```text
/srv/incoming/A
        ↓ rename()
/srv/sessions/A
```

다른 프로세스는 원칙적으로 rename 전 상태 또는 rename 후 상태만 보게 된다.

---

## 왜 rename은 atomic한가

metadata 변경이 작기 때문에 자동으로 atomic해지는 것은 아니고,

`rename()` 자체를 Linux kernel/VFS/filesystem이 atomic operation으로 보이도록 구현한다.

---

## Atomicity

여기서 말하는 atomicity는 CPU instruction 수준의 atomicity와 다르다.

```text
CPU atomic instruction
- compare-and-swap
- atomic increment
```

와

```text
Filesystem atomic operation
- rename
```

은 다른 층의 개념이다.

Filesystem atomicity는 대략:

```text
Application
   ↓
rename()
   ↓
Linux VFS
   ↓
Filesystem implementation
   ↓
namespace / inode / directory metadata
```

레벨에서 제공된다.

---

## Lock

동시에 여러 프로세스가 같은 directory나 inode를 수정할 수 있기 때문에,

Atomic한 처리를 위해서 lock이 사용될 수 있다.

```text
Process A
rename("/incoming/A", "/sessions/A")

Process B
rename("/incoming/B", "/sessions/B")

Process C
lookup("/sessions/A")
```

이런 요청이 동시에 들어오면 filesystem metadata가 충돌하지 않도록 동기화해야 한다.

그래서 kernel/VFS/filesystem은 관련 directory와 inode에 대한 lock을 사용한다.

개념적으로:

```text
rename 시작
   ↓
관련 source/destination directory lock
   ↓
현재 상태 확인
   ↓
namespace 변경
   ↓
metadata 갱신
   ↓
lock 해제
```

정도로 이해할 수 있다.

실제 구현은 filesystem과 kernel 버전에 따라 더 복잡할 것이다.

Lock은 다른 concurrent operation이 rename 도중 관련 metadata를 동시에 변경하지 못하도록 막는다.

예를 들어 한 프로세스가 바꾸는 동안 다른 프로세스가 같은 entry를 동시에 수정하면 consistency가 깨질 수 있다.

Lock을 이용하면, 

```text
현재 rename이 관련 구조를 수정하는 동안
다른 conflicting operation은 기다리게 함
```

으로써 atomic operation 동작을 돕는다.

다만,

> rename atomicity = lock 하나만 걸면 끝

은 아니다.

실제로는 다음이 함께 사용될 수 있다.

```text
VFS synchronization
inode/directory locking
filesystem-specific transaction
journal
```

---

## Journaling과 Atomicity

ext4 같은 journaling filesystem은 filesystem metadata 변경을 journal transaction으로 관리할 수 있다.

개념적으로,

```text
rename()
   ↓
VFS synchronization / locking
   ↓
filesystem metadata 변경
   ↓
journal transaction
```

형태다.

Journaling은 특히 crash 이후 filesystem metadata consistency를 유지하는 데 도움을 준다.

---

## 서버에 적용하면

```text
/srv/incoming
= 작업 중인 영역
```

```text
/srv/sessions
= 검증 완료 영역
```

흐름:

```text
Upload
   ↓
/srv/incoming/stage
   ↓
Validation
   ↓
모두 정상
   ↓
rename(stage, final)
   ↓
/srv/sessions/<session_id>
```

이때,

```text
왜 빠른가?
→ 대용량 데이터를 다시 복사하지 않고
  filesystem metadata를 변경하기 때문
```

```text
왜 atomic한가?
→ kernel/filesystem이 rename을
  atomic namespace operation으로 구현하기 때문
```

```text
atomicity 구현에 무엇이 사용될 수 있는가?
→ directory/inode lock
  VFS synchronization
  filesystem transaction/journaling
```

---

## mv 명령어도 rename인가?

> 같은 filesystem 안에서는 대체로 그렇다고 보면 된다.

```bash
mv /srv/incoming/a /srv/sessions/a
```

두 경로가 같은 filesystem 위에 있다면,
`mv`는 실제 데이터 복사보다 kernel의 rename 계열 연산을 이용해 처리할 수 있다.

그래서 빠르다.

하지만,

```bash
mv /mnt/disk1/a /mnt/disk2/a
```

처럼 서로 다른 filesystem 사이를 이동하면 상황이 달라진다.

이 경우 rename syscall 하나로 처리할 수 없기 때문에
사용자에게는 여전히 이동처럼 보이지만 내부적으로는

```text
copy
 ↓
원본 delete
```

에 가까운 동작이 필요하다.

즉 동일한 `mv` 명령이라도
source와 destination의 filesystem 관계에 따라 실제 내부 동작이 달라질 수 있다.

---

## 왜 다른 filesystem에서는 rename이 안 되는가

```text
/dev/nvme0n1p1
    └── /srv/incoming

/dev/nvme1n1p1
    └── /data/sessions
```

라고 하자.

두 filesystem은 각각 자신의,

```text
inode namespace
directory entries
allocation metadata
journaling metadata
```

등을 독립적으로 관리한다.

한 filesystem 내부의 inode를
다른 filesystem의 directory entry가 그대로 참조하도록 연결할 수 없다.

그래서:

```python
Path("/disk1/a").rename("/disk2/a")
```

같은 요청은 실패할 수 있다.

Linux에서는 이런 상황에서 대표적으로

```text
EXDEV
Invalid cross-device link
```

계열 오류가 발생할 수 있다.

---

## 실제로 같은 filesystem인지 확인

다음 명령으로 확인할 수 있다.

```bash
df /srv/incoming /srv/sessions
```

둘 다 같은 filesystem device를 가리키면 된다.

예:

```text
Filesystem   Mounted on
/dev/root    /
/dev/root    /
```

그러면 두 디렉터리는 동일 filesystem 위에 있다.

반대로:

```text
/dev/nvme0n1p1
/dev/nvme1n1p1
```

처럼 다르게 나오면 cross-filesystem 이동이다.

---

## inode가 실제로 유지되는지 확인

같은 filesystem 내부의 rename에서
파일 inode가 그대로 유지되는지 직접 볼 수도 있다.

```bash
stat /srv/incoming/test.txt
```

출력에서 inode 번호를 확인한다.

그다음,

```bash
mv /srv/incoming/test.txt /srv/sessions/test.txt
```

그리고 다시,

```bash
stat /srv/sessions/test.txt
```

를 확인한다.

같은 filesystem 내부 이동이라면 보통 동일 inode를 확인할 수 있다.

즉,

```text
파일 자체가 새로 생성된 것이 아니라
기존 파일의 pathname 관계가 변경되었다.
```

는 것을 직접 확인할 수 있다.

---

## 파일을 열어둔 상태에서 rename하면

Linux에서는 프로세스가 이미 파일을 열어둔 상태라면
rename 이후에도 기존 file descriptor가 계속 유효하다.

```text
Process A:

open("/srv/incoming/a/video.mp4")
```

그 이후 다른 프로세스가:

```text
/srv/incoming/a
→
/srv/sessions/a
```

로 rename해도,
Process A가 이미 열어둔 file descriptor는 기존 inode를 계속 가리킨다.

즉 Linux/Unix에서

파일 이름이 바뀌거나 경로가 이동해도
이미 열린 fd가 곧 바로 깨지는 것은 아니라는 것이다. 

---

# 9. Android → Server API 

현재 upload 방식은 ZIP 단일 파일이 아니다

```text
POST /sessions
Content-Type: multipart/form-data
```

형태로 Session의 각 파일을 multipart로 보낸다.

## Header

```text
Idempotency-Key: <session_id>
```

`session_id`는 UUID 형식이다.

그리고 반드시:

```text
Idempotency-Key
==
metadata.json의 session_id
```

이어야 한다.

---

# 10. Multipart Part

Main camera Session 기준:

| Part Name | Filename |
|---|---|
| metadata | metadata.json |
| main_video | main_rgb.mp4 |
| main_frame_timestamps | main_frame_timestamps.csv |
| accelerometer | accelerometer.csv |
| gyroscope | gyroscope.csv |
| rotation_vector | rotation_vector.csv |
| arcore_poses | arcore_poses.csv |
| episodes | episodes.csv |

Ultra-wide 활성화 시 추가,

| Part Name | Filename |
|---|---|
| ultrawide_video | ultrawide_rgb.mp4 |
| ultrawide_frame_timestamps | ultrawide_frame_timestamps.csv |

서버는 part name뿐 아니라 실제 filename도 검증한다.

---

# 11. Metadata Manifest와 Integrity Validation

`metadata.json` 안에는 각 파일에 대해 최소한 다음 정보가 들어간다.

```json
{
  "path": "main_rgb.mp4",
  "sizeBytes": 123456789,
  "sha256": "..."
}
```

서버는 파일을 받은 뒤 실제로 다시 file size와 SHA-256을 계산한다.

```text
Android Finalize
       │
       ├─ size
       └─ sha256
              │
              ▼
        metadata.json
              │
           Upload
              │
              ▼
          CPU Server
              │
       실제 파일 재계산
              │
        ┌─────┴─────┐
        │           │
      Match       Mismatch
        │           │
      Accept       Reject
```

목적은 단순히 HTTP request가 성공했는지가 아니라 전송된 파일이 Android가 finalize한 파일과 byte-level에서 동일한지 확인하는 것

---

# 12. Idempotency

네트워크 upload에서는 다음 상황이 가능하다.

```text
Android
 ↓
Upload
 ↓
Server 저장 완료
 ↓
응답이 Android에 도착하기 전 Wi-Fi 끊김
```

Android 입장에서는 성공 여부를 알 수 없다.

따라서 동일한 Session을 다시 전송한다.

### 최초 upload

```text
201 Created

{
  "session_id": "...",
  "result": "created"
}
```

### 동일 Session 재전송

```text
200 OK

{
  "session_id": "...",
  "result": "duplicate"
}
```

즉 retry가 발생해도 동일 데이터를 여러 Session으로 생성하지 않는다.

현재 서버에서 실제 확인된 결과,

```text
최초 요청       → 201 created
동일 요청       → 200 duplicate
다른 session ID → 409 conflict
```

---

# 13. 현재까지 실제 검증 완료된 항목

```text
[완료] Nginx :80 external listen
[완료] FastAPI 127.0.0.1:8000
[완료] 외부 PC → Public IP /health
[완료] POST /sessions
[완료] multipart upload
[완료] metadata parse
[완료] file size validation
[완료] SHA-256 validation
[완료] CSV header validation
[완료] /srv/incoming → /srv/sessions publish
[완료] 최초 upload → 201 created
[완료] retry → 200 duplicate
[완료] Idempotency-Key mismatch → 409
```

외부 PC에서도

```text
curl http://43.201.252.170/health
```

요청이 성공했다.

따라서 네트워크 경로 자체는 정상이다.

---

# 14. systemd 서비스

Uvicorn을 SSH shell에서 직접 실행하지 않고 systemd가 관리하도록 하는 것

```text
systemd
   ↓
uvicorn
   ↓
FastAPI
```

이를 통해
- SSH 종료 후에도 서버 유지
- EC2 reboot 후 자동 시작
- crash 시 자동 restart
- `journalctl`을 통한 로그 관리

가 가능해진다.

---

CPU 서버 기본 구현은 완료하였고,

1. 실제 안드로이드와 통신 확인
2. 데이터 전처리 작업 추가

를 진행해야 한다. 

안드로이드 쪽 진행 상황과, 데이터 수집기가 만들어지고 camera <-> EE 확정을 기다리고 있다.
