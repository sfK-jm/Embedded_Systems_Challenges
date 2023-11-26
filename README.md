# Embedded_Systems_Challenges

> 2023 2학기 임베디드 시스템 과제 2
>
> > redis 분석 및 새로운 명령어 추가

## 과제 설명

![challenges](/image.png)

## 1. redis-server

### 과제 1-1 aeMain()를 중심으로 ㄹ레디스 서버의 전반적인 동작 과정 분석

Redis 서버의 주요 동작은 이벤트 루프 기반으로 처리되며, 이 과정은 ae.c 파일의 aeMain() 함수를 중심으로 이루어집니다.
Redis에서 사용하는 이벤트 루프는 비동기 이벤트 처리 모델을 기반으로 하며, 네트워크 요청, 타이머 이벤트, 파일 입출력 이벤트 등을 효율적으로 관리합니다. aeMain() 함수의 작동 과정은 다음과 같습니다:

1. aeMain()함수는 aeEventLoop 포인터를 인자로 받습니다. 이 구조체는 파일 이벤트, 시간 이벤트, 발생한 이벤트 등 이벤트 루프에 대한 모든 정보를 담고 있습니다.

2. eventLoop->stop을 0으로 설정하여 이벤트 루프를 실행 상태로 만듭니다.

3. eventLoop->stop이 0이 아닐 때까지(즉, 이벤트 루프가 중지되지 않을 때까지) 계속해서 이벤트를 처리하는 while 루프에 들어갑니다.

4. while 루프 내에서 aeProcessEvents() 함수를 AE_ALL_EVENTS, AE_CALL_BEFORE_SLEEP, AE_CALL_AFTER_SLEEP 플래그와 함께 호출합니다. 이 함수는 처리할 준비가 된 모든 이벤트를 처리하고 잠자기 전후에 호출되는 콜백을 호출합니다.

5. aeMain() 함수가 반환되면, 이벤트 루프가 중지되고, Redis 서버는 더 이상 클라이언트의 요청을 처리하지 않습니다.

요약하면, aeMain() 함수는 Redis 서버가 실행되고 클라이언트 요청을 처리하도록 하는 역할을 합니다. 이는 이벤트 루프를 계속 실행하여 이벤트를 처리하도록 함으로써 이루어집니다.

```
1. 이벤트 루프 시작
    |
    v
2. 이벤트 루프 초기화
    |
    v
3. 네트워크 이벤트 감지
    |
    v
4. 클라이언트 요청 처리
    |
    v
5. 타이머/기타 이벤트 처리
    |
    v
6. 이벤트 루프 종료

```

### 과제 1-2 명령어 처리

Redis의 SET 명령어가 클라이언트에서 서버로 전달되고, 최종적으로 In-Memory DB에 저장되는 과정은 다음과 같습니다:

1. 클라이언트에서 SET x 1 명령어를 입력하면, 이 명령어는 Redis 프로토콜에 따라 서버로 전송됩니다.

2. 서버는 네트워크 소켓을 통해 명령어를 받아들이고, 이를 처리하기 위해 이벤트 루프에 등록합니다.

3. 이벤트 루프는 등록된 이벤트를 순차적으로 처리합니다. SET x 1 명령어가 처리되면, 이는 Redis 서버의 명령 테이블에서 해당하는 함수를 찾아 실행합니다.

4. SET 명령어에 해당하는 함수는 x라는 키에 1이라는 값을 설정하는 작업을 수행합니다. 이 값은 Redis의 In-Memory DB에 저장됩니다.

이 과정을 그림으로 나타내면 다음과 같습니다:

```
1. 클라이언트 ---- `SET x 1` ----> 서버
    |
    v
2. 이벤트 루프 등록
    |
    v
3. 이벤트 처리 (명령어 실행)
    |
    v
4. In-Memory DB에 `x: 1` 저장
```

> redis에서는 주로 다음과 같은 자료 구조를 사용합니다:

> > 버퍼: 클라이언트에서 서버로 전송되는 명령어를 임시로 저장하는데 사용됩니다.

> > 명령 테이블: 각 Redis 명령어에 해당하는 함수를 저장하고 있습니다.

> > In-Memory DB: 키-값 쌍을 저장하는 메인 데이터 저장소입니다. 이는 해시 테이블 형태로 구현되어 있습니다.

### pang 명령어 구현

- pang이라는 명령어는 ping이랑 같은 역활을 수행함
- 차이점으로는 pang이라는 명령어는 PUNG을 출력함

server.h에 pingCommand 명령어 선언

```h
void pangCommand(client *c);
```

server.c에 pingCommand 명령어 생성

```c
/*=================== PANG COMMAND ===================*/
void pangCommand(client *c) {
    /* The command takes zero or one arguments. */
    if (c->argc > 2) {
        addReplyErrorArity(c);
        return;
    }

    if (c->flags & CLIENT_PUBSUB && c->resp == 2) {
        addReply(c,shared.mbulkhdr[2]);
        addReplyBulkCBuffer(c,"PUNG",4);
        if (c->argc == 1)
            addReplyBulkCBuffer(c,"",0);
        else
            addReplyBulk(c,c->argv[1]);
    } else {
        if (c->argc == 1)
            addReply(c,shared.pung);
        else
            addReplyBulk(c,c->argv[1]);
    }
}
```

server.h의 shareObjectsStruct에 pung 추가

```h
    struct sharedObjectsStruct {
    robj *ok, *err, *emptybulk, *czero, *cone, *pong, *pung, // 생략
    }
```

server.c에 서버초기화 영역에 공유 문자열 객체를 생성하는 함수인
createShareObjects에 pung을 추가

```c

void createSharedObjects(void) {
    int j;

    /* Shared command responses */
    // 생략
    shared.pung = createObject(OBJ_STRING,sdsnew("+PUNG\r\n"));
    // 생략
}
```

commands.def에 ping명령어처럼 다음과 같이 구현

```def
/********** PANG ********************/

#ifndef SKIP_CMD_HISTORY_TABLE
/* PING history */
#define PANG_History NULL
#endif

#ifndef SKIP_CMD_TIPS_TABLE
/* PING tips */
const char *PANG_Tips[] = {
"request_policy:all_shards",
"response_policy:all_succeeded",
};
#endif

#ifndef SKIP_CMD_KEY_SPECS_TABLE
/* PING key specs */
#define PANG_Keyspecs NULL
#endif

/* PING argument table */

struct COMMAND_ARG PANG_Args[] = {
{MAKE_ARG("message",ARG_TYPE_STRING,-1,NULL,NULL,NULL,CMD_ARG_OPTIONAL,0,NULL)},
};


// (중략)

{MAKE_CMD("pang","Description of the pang command.","O(1)","1.0.0",CMD_DOC_NONE,NULL,NULL,"connection",COMMAND_GROUP_CONNECTION,PANG_History,0,PANG_Tips,0,pangCommand,-1,CMD_FAST,ACL_CATEGORY_CONNECTION,PANG_Keyspecs,0,NULL,1),.args=PANG_Args},


```

![pangCommand](/pang_command.png)

## 2. In-Memory Database

### 과제 2-1 Key-value store

- 레디스 database(key-value-store)가 메모리에 저장되는 자료 구조를 분석하라

Redis의 데이터베이스는 메모리에 저장되는 key-value store로, 주로 dict라는 자료 구조를 통해 관리됩니다. dict는 해시 테이블을 구현한 것으로, 키를 해시 함수를 통해 인덱스로 변환하고, 이 인덱스를 사용해 값을 저장하거나 검색합니다.

redis에서 사용자가 입력한 데이터가 데이터베이스에 저장되는 과정

1. Client에서 Command 수신: Redis 클라이언트는 사용자로부터 명령을 받습니다. 예를 들어, SET key value와 같은 명령을 받을 수 있습니다.

2. Command Parsing: Redis 서버는 클라이언트로부터 받은 명령을 파싱합니다. 이 과정에서 명령의 유효성을 검사하고, 명령의 이름과 인자를 추출합니다.

3. Command Lookup and Execution: Redis 서버는 파싱된 명령의 이름을 기반으로 명령 테이블에서 해당 명령의 함수를 찾습니다. 찾은 함수를 실행하여 명령을 처리합니다.

4. Data Structure Operation: 명령의 처리 과정에서 Redis 서버는 적절한 데이터 구조에 대한 연산을 수행합니다. 예를 들어, SET 명령의 경우, Hashtable에 key-value 쌍을 추가하는 연산을 수행합니다.

5. Response to Client: 연산이 완료되면, Redis 서버는 결과를 클라이언트에게 응답합니다. 예를 들어, SET 명령의 경우, 성공적으로 값을 설정하면 "OK"를 응답합니다.

---

Redis는 주로 해시 테이블을 사용하여 메모리에 데이터를 저장합니다. 이 해시 테이블은 dict 구조체로 표현되며, 각 키-값 쌍은 dictEntry 구조체로 표현됩니다.

dictEntry 구조체는 다음과 같은 필드를 가집니다:

- void \*key: 키를 저장하는 포인터입니다.
- union { void \*val; uint64_t u64; int64_t s64; double d; } v: 값(value)을 저장하는 유니온입니다. 이 유니온은 여러 가지 데이터 타입을 저장할 수 있도록 설계되어 있습니다.
- struct dictEntry \*next: 같은 해시 버킷의 다음 dictEntry를 가리키는 포인터입니다. 이 포인터는 체이닝 방식으로 해시 충돌을 해결하는 데 사용됩니다.
- void \*metadata[]: 메타데이터를 저장하는 배열입니다. 이 배열의 크기는 dictType의 dictEntryMetadataBytes() 함수에 의해 결정됩니다.

### 과제 2-2 Data types

- Data Type 중 String, List가 어떻게 구현되어 있는지 분석하라

Redis의 String과 List 데이터 타입은 다음과 같이 구현되어 있습니다:

- String: Redis의 String 데이터 타입은 robj라는 구조체를 통해 표현됩니다. 이 구조체는 object.c 파일에서 정의되며, 값의 타입, 인코딩 방식, 실제 데이터 등을 저장합니다. String 데이터 타입은 일반적으로 REDIS_ENCODING_RAW 또는 REDIS_ENCODING_INT 인코딩 방식을 사용합니다. getDecodedObject 함수는 인코딩된 객체를 디코딩하여 원래의 String 객체를 반환합니다.

- List: Redis의 List 데이터 타입은 **adlist.c**와 **adlist.h** 파일에서 정의되고 구현됩니다. List는 연결 리스트를 기반으로 하며, 각 노드는 listNode라는 구조체를 통해 표현됩니다. 또한, List 전체는 list라는 구조체를 통해 표현되며, 이는 헤드 노드, 테일 노드, 노드의 수 등을 저장합니다. List 데이터 타입은 REDIS_ENCODING_LINKEDLIST 또는 REDIS_ENCODING_ZIPLIST 인코딩 방식을 사용할 수 있습니다.

###### redis 다운로드 링크 [REDIS](https://redis.io/download/)
