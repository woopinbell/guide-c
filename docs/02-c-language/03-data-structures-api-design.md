# 자료구조와 API 작성

자료구조를 구현할 때는 구조체 필드를 정하는 것만으로 충분하지 않습니다. **어떤 상태를 정상으로 볼 것인지, 각 함수가 성공하거나 실패했을 때 상태가 어떻게 달라지는지, 동적 메모리를 누가 소유하고 언제 해제하는지**까지 함께 정해야 합니다.

이 규칙이 먼저 정해져 있어야 구현과 테스트가 같은 기준을 사용할 수 있습니다.

## 먼저 유효한 상태를 적기

자료구조가 항상 만족해야 하는 조건을 먼저 적습니다. 이런 조건을 **불변식(invariant)** 이라고 합니다.

동적 배열을 예로 들면 다음과 같습니다.

```text
size <= capacity

capacity == 0이면:
    data == NULL
    size == 0

capacity > 0이면:
    data != NULL

읽을 수 있는 원소의 index:
    0 <= index < size
```

예를 들어 다음 상태는 잘못된 상태입니다.

```text
size = 10
capacity = 4
```

`size <= capacity`를 위반하기 때문입니다.

다음 상태도 위 규칙에서는 허용하지 않습니다.

```text
capacity = 0
data != NULL
```

물론 다른 규칙을 선택할 수도 있습니다. 중요한 점은 특정 표현 방식 자체가 아니라 **한 가지 규칙을 정하고 모든 함수가 그 규칙을 일관되게 유지하는 것**입니다.

문자열 자료구조라면 추가 조건이 필요합니다.

```text
length < capacity
data[length] == '\0'
```

예를 들어:

```c
struct owned_string {
    char *data;
    size_t length;
    size_t capacity;
};
```

에서 `length`가 문자열 내용의 길이라면 `data[length]`에는 마지막 NUL 문자를 저장할 수 있어야 합니다.

따라서 내용이 5바이트라면 최소한:

```text
length = 5
capacity >= 6
```

이어야 합니다.

이런 조건을 구현 전에 적어두면 다음을 판단하기 쉬워집니다.

* 어떤 입력을 잘못된 상태로 볼 것인가
* 함수 실행 전에 무엇을 검사해야 하는가
* 어떤 값들을 함께 변경해야 하는가
* 실패했을 때 어디까지 원래 상태를 복구해야 하는가
* 테스트에서 무엇을 확인해야 하는가

## 공개 구조체와 불투명 타입

작은 C 라이브러리에서는 구조체 정의를 헤더에 그대로 공개할 수 있습니다.

```c
struct int_vector {
    int *data;
    size_t size;
    size_t capacity;
};
```

장점은 단순합니다.

* 호출자가 스택이나 다른 구조체 내부에 직접 배치할 수 있습니다.
* 별도 생성 함수가 없어도 됩니다.
* 필드를 직접 읽을 수 있습니다.
* 구현이 비교적 단순합니다.

하지만 호출자가 다음처럼 직접 값을 바꿀 수도 있습니다.

```c
vector.size = 1000;
vector.capacity = 4;
```

그러면 자료구조의 유효한 상태가 깨집니다.

따라서 필드를 공개하더라도 다음과 같은 사용 규칙을 둘 수 있습니다.

> 초기화 이후 `size`, `capacity`, `data`는 라이브러리가 제공하는 함수만 변경합니다. 호출자는 필요한 경우 읽기만 합니다.

더 강하게 제한하려면 구조체의 실제 정의를 `.c` 파일에 숨길 수 있습니다.

헤더:

```c
struct int_vector;

struct int_vector *int_vector_create(void);
void int_vector_destroy(struct int_vector *vector);

size_t int_vector_size(const struct int_vector *vector);
int int_vector_push(struct int_vector *vector, int value);
```

구현 파일:

```c
struct int_vector {
    int *data;
    size_t size;
    size_t capacity;
};
```

호출자는 구조체 내부 필드를 알 수 없으므로 직접 변경할 수 없습니다. 이런 형태를 보통 **불투명 타입(opaque type)** 이라고 합니다.

대신 다음 비용이 생길 수 있습니다.

* 구조체 자체를 동적으로 할당해야 할 수 있음
* 생성/정리 함수가 필요함
* 간단한 필드 조회에도 함수를 호출해야 할 수 있음

따라서 공개 구조체와 불투명 타입 중 하나가 항상 더 좋은 것은 아닙니다. 라이브러리 규모와 호출자가 내부 표현을 알아야 하는지에 따라 선택합니다.

## 함수별 성공과 실패를 정하기

공개 함수는 반환값뿐 아니라 **호출 전후 상태**까지 정의하는 것이 좋습니다.

예를 들어:

```c
int int_vector_push(struct int_vector *vector, int value);
```

다음처럼 규칙을 정할 수 있습니다.

```text
입력:
    vector는 유효한 int_vector를 가리켜야 함

성공:
    0 반환
    value가 마지막 원소로 추가됨
    size가 1 증가함
    필요하면 capacity와 data가 변경될 수 있음

실패:
    -1 반환
    기존 data, size, capacity와 모든 기존 원소의 값이 유지됨
```

마지막 실패 조건이 중요합니다.

예를 들어 메모리 할당 실패 후에도 기존 벡터가 그대로 남아 있다면 호출자는 다음처럼 처리할 수 있습니다.

```c
if (int_vector_push(&vector, value) != 0) {
    /* vector는 여전히 정상 상태이므로 정리하거나 재사용 가능 */
}
```

반대로 실패 후 일부 상태가 변경될 수 있다면 호출자는 어떤 값이 믿을 수 있는지 별도로 알아야 합니다.

## 실패 후 상태 보존

자료구조 API를 설계할 때 유용한 방식 중 하나는:

> 함수가 실패하면 호출 전 상태를 가능한 한 그대로 유지합니다.

예를 들어:

```text
before:
    data = P
    size = 8
    capacity = 8

push 실패

after:
    data = P
    size = 8
    capacity = 8
    기존 8개 원소도 그대로
```

이 규칙이 있으면 호출자가 실패 복구를 쉽게 할 수 있습니다.

하지만 모든 함수가 반드시 이 규칙을 가져야 하는 것은 아닙니다. 어떤 작업은 부분 진행 상태를 허용할 수도 있습니다.

중요한 것은 **실패 시 상태를 문서에서 명확하게 정의하는 것**입니다.

## 실패하기 쉬운 작업을 먼저 수행하기

동적 배열의 `push`는 공간이 부족하면 메모리 재할당이 필요합니다.

안전한 순서는 대략 다음과 같습니다.

```text
입자와 현재 상태 검사
→ 새로운 capacity 계산
→ capacity 계산 overflow 검사
→ 필요한 바이트 수 계산 전 overflow 검사
→ realloc 시도
→ 성공한 새 포인터 반영
→ 새 원소 기록
→ size 증가
```

예를 들어 다음 순서는 위험합니다.

```c
vector->capacity *= 2;

int *resized = realloc(
    vector->data,
    vector->capacity * sizeof *vector->data
);

if (resized == NULL) {
    return -1;
}
```

`realloc`이 실패하면:

```text
capacity = 새 값
data = 예전 allocation
```

이 되어 실제 할당 크기와 `capacity`가 맞지 않게 됩니다.

대신 계산 결과를 지역 변수에 보관합니다.

```c
size_t new_capacity = vector->capacity * 2;

int *resized = realloc(
    vector->data,
    new_capacity * sizeof *vector->data
);

if (resized == NULL) {
    return -1;
}

vector->data = resized;
vector->capacity = new_capacity;
```

즉 **실패할 수 있는 작업의 결과를 먼저 확보한 뒤 성공이 확인되었을 때 공개 상태를 변경합니다.**

## 상태 변경 순서도 중요합니다

할당이 성공했다고 해서 이후 상태를 아무 순서로나 변경해도 되는 것은 아닙니다.

예를 들어:

```c
vector->size++;
vector->data[vector->size - 1] = value;
```

보다:

```c
vector->data[vector->size] = value;
vector->size++;
```

가 의도를 더 명확하게 나타냅니다.

후자는 새 원소 기록을 완료한 뒤 `size`를 증가시킵니다.

즉 `size`는 항상 **이미 유효하게 저장된 원소 개수**를 나타내도록 유지할 수 있습니다.

## 출력 매개변수는 성공 후 변경하기

C에서는 함수가 여러 결과를 반환하기 위해 출력 매개변수를 자주 사용합니다.

```c
int int_vector_get(
    const struct int_vector *vector,
    size_t index,
    int *out_value
)
{
    if (vector == NULL ||
        out_value == NULL ||
        index >= vector->size) {
        return -1;
    }

    *out_value = vector->data[index];
    return 0;
}
```

이 함수의 실패 규칙을 다음처럼 정할 수 있습니다.

```text
성공:
    0 반환
    *out_value에 요청한 값을 기록

실패:
    -1 반환
    *out_value는 변경하지 않음
```

그러면:

```c
int value = 123;

if (int_vector_get(&vector, 99, &value) != 0) {
    /* value는 여전히 123 */
}
```

처럼 호출자가 실패 여부와 출력값 변경 여부를 별도로 추적할 필요가 없습니다.

출력 매개변수를 여러 개 사용하는 함수라면 특히 중요합니다. 일부만 변경한 뒤 실패하면 호출자가 어떤 결과가 유효한지 판단하기 어려워집니다.

## 초기화와 정리

자료구조에는 일반적으로 사용 가능한 상태가 되는 시점과 자원을 반납하는 시점이 있습니다.

```text
초기화
→ 0회 이상의 작업
→ 정리
```

예:

```c
struct int_vector vector;

if (int_vector_init(&vector) != 0) {
    return -1;
}

/* 사용 */

int_vector_destroy(&vector);
```

초기화 함수는 성공한 객체가 자료구조의 불변식을 만족하도록 만들어야 합니다.

예를 들어 빈 벡터의 초기 상태를:

```text
data == NULL
size == 0
capacity == 0
```

으로 정할 수 있습니다.

## 초기화되지 않은 객체는 별개의 문제입니다

다음 코드는 안전하다고 볼 수 없습니다.

```c
struct int_vector vector;

int_vector_destroy(&vector);
```

`vector`의 필드가 초기화되지 않았기 때문입니다.

`destroy`가:

```c
free(vector->data);
```

를 수행하면 임의의 값을 포인터로 사용하게 될 수 있습니다.

따라서 API가 특별히 지원한다고 명시하지 않는 이상:

> `destroy`는 성공적으로 초기화된 객체에만 호출합니다.

라는 전제 조건이 필요합니다.

## 다시 초기화할 때의 문제

다음도 위험할 수 있습니다.

```c
int_vector_init(&vector);

/* 사용 */

int_vector_init(&vector);
```

두 번째 `init`이 기존 `data`를 단순히 `NULL`로 덮어쓰면 기존 동적 메모리를 잃어버려 메모리 누수가 발생할 수 있습니다.

따라서 보통은 다음 순서를 사용합니다.

```text
init
→ 사용
→ destroy
→ 필요하면 다시 init
```

또는 API가 재초기화를 명시적으로 지원한다면 그 동작을 별도로 정의해야 합니다.

## 반복 가능한 정리 함수

정리 후 다음 상태로 돌려놓을 수 있습니다.

```text
data == NULL
size == 0
capacity == 0
```

예:

```c
void int_vector_destroy(struct int_vector *vector)
{
    if (vector == NULL) {
        return;
    }

    free(vector->data);

    vector->data = NULL;
    vector->size = 0;
    vector->capacity = 0;
}
```

그러면 이미 정상적으로 초기화됐던 객체라면:

```c
int_vector_destroy(&vector);
int_vector_destroy(&vector);
```

처럼 반복 호출하는 것을 안전하게 만들 수 있습니다.

이는 `free(NULL)`이 허용되기 때문입니다.

다만 이것은 **초기화되지 않은 객체에 대한 `destroy`까지 안전하게 만든다는 뜻은 아닙니다.**

## 할당 함수 주입

일반적인 구현은 직접 `realloc`과 `free`를 호출할 수 있습니다.

하지만 메모리 할당 실패를 재현하며 테스트하려면 할당 함수를 교체할 수 있게 만들 수도 있습니다.

예:

```c
struct allocator {
    void *context;

    void *(*resize)(
        void *context,
        void *pointer,
        size_t size
    );

    void (*release)(
        void *context,
        void *pointer
    );
};
```

`context`는 콜백이 사용할 상태를 전달합니다.

예를 들어 테스트용 allocator는 다음 정보를 저장할 수 있습니다.

```text
현재까지의 할당 호출 횟수
몇 번째 호출에서 실패시킬지
release가 몇 번 호출되었는지
```

그러면 다음 상황을 결정적으로 만들 수 있습니다.

* 첫 번째 할당 실패
* 두 번째 크기 증가에서 실패
* 특정 `realloc` 호출만 실패
* 실패 후 기존 포인터 유지 여부 확인
* 실패 후 기존 원소 값 유지 여부 확인
* `destroy`가 올바른 `release` 콜백을 호출하는지 확인

실제 기본 allocator는 내부에서 다음 함수를 사용할 수 있습니다.

```c
realloc(pointer, size);
free(pointer);
```

이 설계의 목적은 단순히 함수 포인터를 사용하는 데 있지 않습니다. **운영 환경에서는 드물게 발생하는 메모리 부족 경로를 테스트에서 정확하게 재현하기 위한 것**입니다.

작은 자료구조에 항상 필요한 설계는 아닙니다. 실패 경로를 직접 검증해야 할 이유가 있을 때 사용합니다.

## 이름은 실제 동작을 드러내기

함수 이름은 무엇을 대상으로 어떤 작업을 수행하는지 드러내는 것이 좋습니다.

예:

```text
record_reader_next
account_transfer
int_vector_push
owned_string_append
```

이 이름들은 각각 다음 정보를 어느 정도 포함합니다.

```text
record_reader_next
→ reader에서 다음 record를 읽음

int_vector_push
→ int vector의 끝에 원소를 추가함

owned_string_append
→ owned string 뒤에 내용을 덧붙임
```

반면:

```text
process
handle
manage
run
```

같은 이름은 주변 문맥이 없으면 어떤 데이터를 무엇으로 바꾸는지 알기 어렵습니다.

물론 `process` 같은 단어 자체가 항상 잘못된 것은 아닙니다. 실제 동작이 하나로 명확한 좁은 범위 안에서는 사용할 수 있습니다.

중요한 것은 함수 이름만 보고도 가능한 한 **대상과 핵심 동작을 추측할 수 있게 하는 것**입니다.

## 오류 코드

간단한 함수는 성공과 실패만 구분하면 충분할 수 있습니다.

```c
return 0;   /* 성공 */
return -1;  /* 실패 */
```

호출자가 실패 이유까지 구분해야 한다면 더 구체적인 값을 사용할 수 있습니다.

```c
enum parse_result {
    PARSE_OK,
    PARSE_INVALID,
    PARSE_RANGE_ERROR
};
```

예를 들어:

```c
enum parse_result parse_integer(
    const char *text,
    long *out_value
);
```

호출자는 다음을 구분할 수 있습니다.

```c
switch (parse_integer(text, &value)) {
case PARSE_OK:
    break;

case PARSE_INVALID:
    /* 숫자 형식이 아님 */
    break;

case PARSE_RANGE_ERROR:
    /* 표현 가능한 범위를 벗어남 */
    break;
}
```

하지만 오류 코드는 많을수록 좋은 것이 아닙니다.

예를 들어 호출자가 모든 실패를 동일하게 처리한다면:

```text
ALLOC_FAILED
CAPACITY_FAILED
COPY_FAILED
INTERNAL_FAILED
```

처럼 지나치게 세분화해도 실제 효용은 없습니다.

오류 종류를 나누려면 호출자 또는 테스트가 그 차이에 따라 실제로 다른 행동을 할 이유가 있어야 합니다.

## 정수와 크기 검사

동적 자료구조에서 `capacity`를 두 배로 늘린다고 가정합니다.

```c
new_capacity = capacity * 2;
```

먼저 원소 개수 계산 자체가 overflow하지 않는지 확인합니다.

```c
if (capacity > SIZE_MAX / 2) {
    return -1;
}

size_t new_capacity = capacity * 2;
```

그다음 실제 메모리 크기를 계산할 수 있는지 확인합니다.

```c
if (new_capacity > SIZE_MAX / sizeof *vector->data) {
    return -1;
}

size_t byte_size =
    new_capacity * sizeof *vector->data;
```

두 검사는 서로 다른 연산을 보호합니다.

```text
capacity * 2
→ 원소 개수 계산

new_capacity * sizeof(element)
→ 바이트 수 계산
```

첫 번째가 안전하다고 두 번째까지 자동으로 안전한 것은 아닙니다.

## 초기 capacity가 0인 경우

단순히:

```c
new_capacity = capacity * 2;
```

라고 하면 `capacity == 0`일 때 결과도 계속 `0`입니다.

따라서 실제 동적 배열은 보통 최초 할당 크기를 별도로 정합니다.

예:

```c
if (vector->capacity == 0) {
    new_capacity = 8;
} else {
    if (vector->capacity > SIZE_MAX / 2) {
        return -1;
    }

    new_capacity = vector->capacity * 2;
}
```

즉 **첫 삽입과 기존 배열 확장은 서로 다른 계산이 필요할 수 있습니다.**

## `size + 1`도 overflow를 고려해야 합니다

다음 코드도 정수 연산입니다.

```c
vector->size + 1
```

실제로 정상적인 자료구조라면 할당 가능한 메모리 크기 때문에 `SIZE_MAX`개의 원소를 가지는 상황은 현실적으로 제한되지만, 자료구조의 크기 계산을 일반적으로 작성한다면 덧셈 자체도 표현 가능한지 생각해야 합니다.

예:

```c
if (vector->size == SIZE_MAX) {
    return -1;
}
```

그 후:

```c
size_t required = vector->size + 1;
```

처럼 계산할 수 있습니다.

핵심은 모든 연산에 무조건 검사를 붙이는 것이 아니라 **새로운 크기를 만드는 정수 연산이 overflow할 가능성이 있는지 확인하는 것**입니다.

## 별칭과 자기 참조

다음과 같은 문자열 API를 생각해 봅니다.

```c
int owned_string_append(
    struct owned_string *string,
    const char *source
);
```

일반적인 외부 문자열을 전달할 수도 있습니다.

```c
owned_string_append(&string, "abc");
```

하지만 현재 문자열 자신의 버퍼를 다시 전달할 수도 있습니다.

```c
owned_string_append(&string, string.data);
```

이를 자기 참조 입력이라고 볼 수 있습니다.

또는 현재 문자열 일부를 전달할 수도 있습니다.

```c
owned_string_append(&string, string.data + 2);
```

이 경우 `source`는 `string->data`와 같은 allocation을 가리키는 **별칭 포인터**입니다.

## 별칭 입력을 지원하지 않는 경우

API가 이런 호출을 지원하지 않는다면 문서에 명시해야 합니다.

예:

```text
source는 string 내부 버퍼와 겹치는 메모리를 가리켜서는 안 됩니다.
```

그러면 구현은 더 단순해질 수 있습니다.

하지만 호출자가 이 조건을 위반하면 결과가 잘못될 수 있으므로 전제 조건을 명확히 알려야 합니다.

## 별칭 입력을 지원하는 경우

지원하려면 최소한 두 문제를 생각해야 합니다.

첫 번째는 `realloc`입니다.

```text
source → 현재 data 내부

realloc(data, ...)
→ 메모리가 다른 주소로 이동

기존 source → 무효
```

따라서 재할당 전에 필요한 상대 위치를 보존하거나, 입력을 임시 공간에 복사하는 등의 처리가 필요합니다.

두 번째는 메모리 영역 겹침입니다.

같은 버퍼 안의 데이터를 이동하거나 복사하는 과정에서 원본과 대상이 겹칠 수 있다면 `memcpy`가 아니라 `memmove`가 필요할 수 있습니다.

따라서 별칭 지원 여부는 단순한 문서상의 선택이 아니라 구현 방식과 테스트 항목을 실제로 바꿉니다.

## 입력 검증과 손상된 객체

공개 구조체라면 호출자가 필드를 직접 바꿀 수 있기 때문에 다음과 같은 손상된 상태가 함수에 들어올 가능성도 있습니다.

```text
size > capacity
capacity > 0인데 data == NULL
```

여기서 중요한 설계 결정이 하나 있습니다.

### 방법 1: 유효한 객체만 받는다고 규정

```text
모든 API는 정상적으로 초기화되고 손상되지 않은 객체만 입력으로 받습니다.
```

이 경우 모든 함수가 내부 상태 전체를 반복해서 검사할 필요는 없습니다.

### 방법 2: 공개 함수에서 내부 상태도 검사

```c
if (vector->size > vector->capacity) {
    return -1;
}

if (vector->capacity > 0 &&
    vector->data == NULL) {
    return -1;
}
```

이 방식은 잘못된 상태를 더 빨리 발견할 수 있지만 모든 함수의 코드가 복잡해질 수 있습니다.

따라서 `"손상된 size/capacity 조합을 테스트한다"`는 항목을 넣으려면 먼저 **API가 손상된 객체를 감지해야 하는지, 아니면 그런 입력 자체를 허용하지 않는지**를 정해야 합니다.

이 부분은 원문의 테스트 목록에서 특히 명시할 필요가 있습니다.

## 테스트할 내용

동적 배열이라면 최소한 다음을 확인할 수 있습니다.

* 빈 상태에서 첫 원소 삽입
* 최초 capacity 할당
* 남은 capacity가 있을 때 삽입
* capacity가 가득 찬 상태에서 크기 증가
* 여러 번 연속된 크기 증가
* 첫 번째 원소 조회
* 마지막 원소 조회
* `index == size`인 범위 밖 조회
* 매우 큰 index 조회
* `NULL` 객체 포인터
* `NULL` 출력 매개변수
* capacity 증가 계산 overflow
* 바이트 수 계산 overflow
* 최초 할당 실패
* 이후 `realloc` 실패
* 할당 실패 후 `data` 포인터 보존
* 할당 실패 후 `size`와 `capacity` 보존
* 할당 실패 후 기존 원소 값 보존
* 실패한 조회 후 출력 매개변수 보존
* `destroy` 후 빈 상태 확인
* 반복 `destroy`를 지원한다면 두 번 호출
* 별칭 입력을 지원한다면 자기 자신 전체 append
* 별칭 입력을 지원한다면 내부 부분 문자열 append

손상된 내부 상태를 검사하도록 API를 설계했다면 다음도 추가합니다.

* `size > capacity`
* `capacity > 0 && data == NULL`
* 정의한 다른 불변식을 위반한 상태

반대로 유효한 객체만 입력으로 받는 API라면 이런 테스트는 공개 함수의 정상 동작 테스트와 분리하는 편이 낫습니다.

## API를 작성할 때 확인할 질문

공개 함수 하나를 작성할 때 다음 질문에 답할 수 있어야 합니다.

```text
입력:
    NULL을 허용하는가?
    객체는 어떤 상태여야 하는가?
    입력 메모리를 읽기만 하는가, 수정하는가?
    별칭 입력을 허용하는가?

성공:
    무엇을 반환하는가?
    어떤 필드가 변경되는가?
    새 메모리가 생긴다면 누가 소유하는가?

실패:
    무엇을 반환하는가?
    기존 객체는 그대로 유지되는가?
    출력 매개변수는 변경되는가?
    새로 확보한 임시 자원은 정리되는가?

수명:
    반환된 포인터는 언제까지 유효한가?
    어떤 작업이 기존 포인터를 무효화할 수 있는가?
```

이 질문들에 답이 있으면 함수 구현뿐 아니라 헤더 주석과 테스트도 훨씬 구체적으로 작성할 수 있습니다.

## 완료 기준

1. 자료구조가 항상 만족해야 하는 유효 상태를 식으로 적습니다.
2. 불변식과 단순한 필드 값의 차이를 설명합니다.
3. 공개 구조체와 불투명 타입의 장단점을 설명합니다.
4. 각 공개 함수의 입력 조건, 성공 후 상태, 실패 후 상태를 설명합니다.
5. 실패 시 기존 상태를 유지해야 하는 함수에서는 성공이 확인되기 전에 공개 상태를 변경하지 않습니다.
6. 출력 매개변수는 성공이 확인된 뒤 변경합니다.
7. 초기화되지 않은 객체와 정상적으로 초기화된 빈 객체를 구분합니다.
8. 재초기화 전에 기존 자원을 정리해야 하는 이유를 설명합니다.
9. 동적 메모리의 소유자와 정리 함수를 명확하게 정합니다.
10. 할당 함수 주입이 메모리 부족 경로를 결정적으로 테스트하기 위한 방법임을 설명합니다.
11. capacity 증가와 실제 바이트 수 계산을 별도로 overflow 검사합니다.
12. `capacity == 0`인 최초 확장 경로를 별도로 처리합니다.
13. 함수 이름이 대상과 실제 동작을 드러내도록 작성합니다.
14. 오류 코드를 세분화할 때 호출자가 그 차이를 실제로 사용할 이유가 있는지 확인합니다.
15. 별칭 및 자기 참조 입력을 지원할지 명시합니다.
16. 별칭 입력을 지원한다면 `realloc` 이후 기존 포인터가 무효화될 수 있음을 고려합니다.
17. 공개 함수가 손상된 내부 상태를 검사할지, 아니면 유효한 객체만 입력으로 받을지 명확하게 정합니다.
18. 할당 실패 후 기존 포인터, 크기 정보, 기존 원소가 보존되는지 테스트합니다.
