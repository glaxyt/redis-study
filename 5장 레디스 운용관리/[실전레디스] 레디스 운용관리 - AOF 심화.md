## 1. AOF 개요

Redis의 AOF(Append Only File)는 Redis 서버에 들어온 **쓰기 명령어들을 순차적으로 파일에 기록**하여 장애 시 데이터를 복구할 수 있도록 해주는 지속성(persistence) 기능이다. AOF는 RDB와는 달리, Redis의 상태를 시간순으로 재현할 수 있는 장점이 있다.

## 2. AOF 동작 방식의 진화

### 2.1 기존 AOF 구조 (단일 파일 기반)

- 클라이언트 명령이 `appendonly.aof`에 지속적으로 기록된다.
- AOF Rewrite 시 자식 프로세스가 새로운 AOF 파일을 작성하고, 부모 프로세스는 재작성 중 발생한 명령어들을 **rewrite buffer**에 임시 저장한 후 자식에 전달하여 최종 AOF 파일을 완성한다.
- 이 과정에서 **부모 → 자식 간 버퍼 전송 비용**이 발생하며, I/O 병목이 생길 수 있다.

### 2.2 멀티 AOF 구조 (Redis 7.0 이후)

- AOF가 base/incr 구조로 분리되고, manifest 파일로 구성 정보를 관리한다.
- **재작성 중 rewrite buffer 전송 로직이 사라져 성능이 개선됨**

### 주요 파일 설명

| 파일명 | 설명 |
| --- | --- |
| appendonly.aof.base | AOF 재작성 시 생성된 스냅샷 명령어 기록 파일 |
| appendonly.aof.incr | base 생성 이후 들어온 명령어를 기록하는 증분 파일 |
| appendonly.aof.manifest | base/incr의 구성 정보를 관리하는 메타 파일 |

## 3. Multi-AOF 흐름 예시

### 1단계: 초기 상태

- 클라이언트 명령은 `appendonly.aof.incr`에만 기록됨
- manifest: `incr-only` 모드

### 2단계: AOF Rewrite 시작

- 자식 프로세스가 `appendonly.aof.base.tmp` 생성 시작
- 부모는 계속 `incr`에 명령을 기록 (rewrite buffer는 사용 안 함)

### 3단계: AOF Rewrite 완료

- base.tmp → base로 rename
- manifest 업데이트 → `hybrid` 모드로 전환 (base + incr 사용)
- Redis는 base + incr 조합으로 복원 가능

### 4단계: 새로운 명령 계속 기록

- `incr`에 명령 계속 추가
- 이후 재작성 시 새로운 base/incr 생성 → 반복

## 4. 운영 시 백업 주의사항

### 기존 방식에서는:

- `appendonly.aof` 단일 파일만 백업하면 됨

### Multi-AOF에서는:

- `appendonly.aof.base`, `appendonly.aof.incr`, `appendonly.aof.manifest` **모두 백업 필요**
- 재시작 시 manifest 기준으로 복원되기 때문에 하나라도 누락되면 복구 불가

### 안전한 백업 방법

```bash
# AOF 비활성화 후 백업
127.0.0.1:6379> CONFIG SET appendonly no
# 백업 수행
# AOF 다시 활성화
127.0.0.1:6379> CONFIG SET appendonly yes
```

## 5. AOF와 휘발성 데이터의 관계

QR 토큰, Refresh Token(RT)과 같은 **짧은 TTL의 휘발성 데이터**만 Redis에 저장할 경우:

- Redis 재시작 시 이 데이터는 없어도 큰 문제가 없음
- 앱이 재발급하거나 복구할 수 있음 (Cache-Aside 패턴)
- 이 경우 **AOF/RDB 비활성화도 고려 가능**

### 실무 예시 설정

```bash
# redis.conf 예시
save ""            # RDB 비활성화
appendonly no       # AOF 비활성화
```

## 6. 결론 및 운영 전략

| 데이터 특성 | 권장 전략 |
| --- | --- |
| 장기 보존, 중요도 높은 상태 정보 | AOF 또는 RDB 사용 |
| 전부 휘발성 (QR, RT 등) | AOF/RDB 비활성화, 복구는 앱에서 담당 |
| 휘발성과 상태 데이터 혼재 | AOF + `appendfsync everysec` 조합 고려 |

---