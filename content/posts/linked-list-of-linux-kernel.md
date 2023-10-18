---
title: 리눅스 커널의 연결 리스트
date: 2023-10-18 17:10:57 +0900
tags:
  - Linux
  - kernel
  - data_structure
categories:
  - 리눅스 커널
comments: true
---
리눅스 커널에서는 태스크 목록 등을 관리하기 위해 이중 연결 리스트가 사용되는데, 이 이중 연결 리스트는 중복 코드를 최대한 줄이기 위해 독특한 형태로 구성되어 있다.

# 일반적인 이중 연결 리스트
```c
struct list_head{
	data_t payload;
	struct list_head *next, *prev;
};
```
이중 연결 리스트는 일반적으로 payload와 이전, 이후 노드를 가리키는 포인터들로 구성되어 있다. 이러한 구조체가 몇 개 밖에 없다면 큰 문제가 되지 않는다. 하지만 리눅스 커널처럼 복잡한 코드의 경우 여러 타입의 이중 연결 리스트가 매우 많이 필요하고 타입의 개수 만큼의 insert, delete, 순회 함수 등을 구현해야 한다.

# 리눅스 커널의 이중 연결 리스트
이를 해결하기 위해 리눅스 커널에서는 이중 연결 리스트 구조체를 다음과 같이 정의하고 있다.
```c
struct list_head{
	struct list_head *next, *prev;
};
```

이 리스트를 위한 초기화, 삽입, 순회 함수/매크로는 다음과 같다.

(실제 커널에서 사용되는 코드와 다릅니다)
```c
// 초기화
#define LIST_HEAD_INIT(name) { &(name), &(name) }

// 삽입
void list_add(struct list_head *new, struct list_head *head){  
    new->next = head->next;  
    head->next->prev = new;  
    head->next = new;  
}

// 순회
#define LIST_FOR_EACH(pos, head) for (pos = head->next; pos!=head; pos = pos->next)
```

일반적인 이중 연결 리스트와는 다르게, 이 구조체에는 페이로드 필드가 빠져 있다. 그렇다면 이 구조체를 어떻게 사용할 수 있을까?

# 리스트 필드를 포함하는 구조체에 접근하기
리눅스 커널은 `list_head` 구조체를 포함하고 있는 구조체의 주소를 계산하는 접근 방식을 취하고 있다.

한 구조체 내에서 필드의 위치를 구하는 방법은 다음과 같다.
```c
#define offsetof(TYPE, MEMBER) ((size_t)&((TYPE *)0)->MEMBER)
```
이 매크로는 TYPE 구조체가 0번지에서 시작된다고 가정하고, MEMBER 필드의 주소를 가져온다. 이 계산 결과는 결국 필드의 오프셋 값과 같다.

오프셋 값을 이용하면 해당 필드를 담고 있는 구조체의 주소 값을 역산할 수 있다.
```c
#define container_of(ptr, type, member) ({ \  
    const typeof(((type *)0)->member) *__mptr = (ptr); \  
    (type *)((char *)__mptr - offsetof(type, member)); })
```
이 매크로의 두 번째 줄은 컴파일 시간에 타입을 검사하고 경고를 발생시키기 위해 사용된다.

# 테스트 프로그램
이 매크로들을 사용하면 다음과 같이 리스트를 만들고 순회하는 테스트 프로그램을 작성할 수 있다.
```c
#include <stdio.h>  
  
#define offsetof(TYPE, MEMBER) ((size_t)&((TYPE *)0)->MEMBER)  
  
#define container_of(ptr, type, member) ({ \  
    const typeof(((type *)0)->member) *__mptr = (ptr); \  
    (type *)((char *)__mptr - offsetof(type, member)); })  
  
#define LIST_HEAD_INIT(name) { &(name), &(name) }  
#define LIST_FOR_EACH(pos, head) for (pos = head->next; pos!=head; pos = pos->next)  
  
struct list_head{  
    struct list_head *next, *prev;  
};  
  
void list_add(struct list_head *new, struct list_head *head){  
    new->next = head->next;  
    head->next->prev = new;  
    head->next = new;  
}  
  
struct list_entry{  
    int key;  
    struct list_head list;  
};  
  
int main(void){  
    struct list_entry e0 = {  
            .key = 0,  
            .list = LIST_HEAD_INIT(e0.list)  
    };  
  
    struct list_entry e1 = {  
            .key = 1  
    };  
    list_add(&(e1.list), &(e0.list));  
  
    struct list_entry e2 = {  
            .key = 2  
    };  
    list_add(&(e2.list), &(e1.list));  
  
    struct list_head* list;  
    struct list_head* current = &(e0.list);  
    printf("%p key=%d\n", current, e0.key);  
  
    struct list_entry* entry;  
    LIST_FOR_EACH(list, current){  
        entry = container_of(list, struct list_entry, list);  
        printf("%p key=%d\n", list, entry->key);  
    }  
}
```

## 실행 결과
```
0x7ffc9f409128 key=0
0x7ffc9f409148 key=1
0x7ffc9f409168 key=2
```
의도한 것과 같이 잘 실행됨을 확인할 수 있다.