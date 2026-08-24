# 함수, 배열, 문자열

함수는 코드를 단순히 잘게 나누는 수단이 아닙니다. 입력으로 무엇을 받고, 성공하면 무엇을 반환하며, 실패하면 호출자의 값과 자원을 어떻게 처리할지 정하는 단위입니다.

## 함수 선언과 호출

```c
int parse_count(const char *text, size_t *out_count);
```

위의 함수 선언에서 확인할 내용은 다음과 같습니다.

- `text`는 read only 문자열입니다.
- `out_count`는 함수 스코프가 끝나도 외부에서 지속적으로 값이 유지됩니다.
- 반환값은 성공 여부를 알립니다.

함수 내부에서 `out_count`처럼 외부에 연결된 값을 변경해야 한다면 모든 검사가 끝난 뒤에 변경하는 것이 안전합니다.

```c
int parse_count(const char *text, size_t *out_count) {
    size_t value;

    if (text == NULL || out_count == NULL) {
        return -1;
    }
    if (parse_size(text, &value) != 0) {
        return -1;
    }
    *out_count = value;
    return 0;
}
```

실패했을 때 `*out_count`가 유지되므로 호출자는 부분 결과를 해석할 필요가 없으며 반환값만 확인합니다.

## 값 전달과 포인터 전달

C 함수의 인자는 값으로 전달됩니다.

```c
void increment(int value) {
    value++;
}
```

이 함수는 호출자의 정수를 바꾸지 않습니다. 호출자의 객체를 수정하려면 포인터를 통해 주소를 전달합니다.

```c
int increment(int *value) {
    if (value == NULL) {
        return -1;
    }
    (*value)++;
    return 0;
}
```

포인터를 전달했다고 해서 함수가 객체를 소유하게 되는 것은 아닙니다. 함수가 단순히 빌려 쓰는지, 메모리를 해제할 수 있는지 별도로 정해야 합니다.

## 배열과 길이

함수 인자의 배열 표기는 실제로 포인터로 조정됩니다.

```c
long sum_values(const long values[], size_t length);
```

함수는 `values`가 가리키는 배열의 길이를 자동으로 알 수 없기 때문에 길이를 별도 인자로 전달해야하며, 이는 우리가 많은 함수에서 `length`라는 매개변수를 마주하는 이유입니다.

```c
long sum_values(const long *values, size_t length) {
    long total = 0;

    for (size_t index = 0; index < length; index++) {
        total += values[index];
    }
    return total;
}
```

`values == NULL`을 허용할지, `length == 0`일 때 `NULL`을 허용할지는 함수가 정합니다. 단지 호출자와 구현이 같은 규칙을 따르면 됩니다.

## 문자열은 `NUL`로 끝나는 바이트 배열

```c
char text[] = {'h', 'i', '\0'};
```

`strlen`은 `\0`을 찾을 때까지 읽습니다. 배열 안에 종료 문자가 없으면 유효한 범위를 넘어 읽게 되며 이는 UB를 유발합니다.

```c
size_t text_length(const char *text) {
    size_t length = 0;

    if (text == NULL) {
        return 0;
    }
    while (text[length] != '\0') {
        length++;
    }
    return length;
}
```

문자열 길이에는 마지막 `\0`이 포함되지 않기때문에 문자열을 복사할 공간을 계산할 때는 `length + 1`이 필요합니다.

## 문자열 리터럴과 수정 가능 여부

C에서 문자열 리터럴은 큰따옴표로 작성한 문자열입니다.

```c
const char *message = "hello";
```

여기서 `message`는 `"hello"`라는 문자열 리터럴의 첫 번째 문자를 가리킵니다.

문자열 리터럴의 내용은 수정해서는 안 됩니다.

```c
message[0] = 'H'; /* 정의되지 않은 동작 */
```

중요한 점은 `const` 때문에 수정이 금지되는 것이 아니라는 것입니다. 문자열 리터럴 자체가 수정 대상이 아니기 때문입니다.

다음처럼 `const`를 생략해도 수정할 수 있는 문자열이 되는 것은 아닙니다.

```c
char *message = "hello";

message[0] = 'H'; /* 여전히 정의되지 않은 동작 */
```

컴파일러가 이 코드를 허용할 수는 있지만, 실행 결과는 보장되지 않습니다. 실제 환경에서는 읽기 전용 메모리에 문자열 리터럴이 저장되어 프로그램이 비정상 종료될 수도 있습니다.

문자열의 내용을 수정해야 한다면 문자열 리터럴로 배열을 초기화합니다.

```c
char message[] = "hello";

message[0] = 'H';
```

이 경우에는 `"hello"`의 문자들이 `message` 배열에 복사되어 다음과 같은 배열이 만들어집니다.

```text
'h' 'e' 'l' 'l' 'o' '\0'
```

`message`는 수정 가능한 `char` 배열이므로 `message[0] = 'H';`는 정상적인 연산입니다.

즉, 두 선언은 겉보기에는 비슷하지만 의미가 다릅니다.

```c
const char *message = "hello"; /* 문자열 리터럴을 가리킴 */
char message[] = "hello";      /* 수정 가능한 배열을 생성함 */
```

문자열을 읽기만 한다면 첫 번째 형태를 사용하고, 문자열 내용을 직접 수정해야 한다면 두 번째 형태를 사용합니다.


## `ctype` 함수와 음수 `char`

`isspace`, `isdigit`, `isalpha` 같은 `<ctype.h>` 함수에 전달할 수 있는 값은 제한되어 있습니다.

인자는 다음 둘 중 하나여야 합니다.

- `EOF`
- `unsigned char`로 표현 가능한 값

따라서 일반적인 `char` 값을 검사할 때는 다음처럼 `unsigned char`로 변환하는 것이 안전합니다.

```c
if (isspace((unsigned char)*cursor)) {
    /* 공백 문자입니다. */
}
```

이 변환이 필요한 이유는 `char`의 부호 여부가 구현에 따라 달라질 수 있기 때문입니다.

어떤 환경에서는 `char`가 `signed char`처럼 동작합니다. 이 경우 상위 비트가 설정된 바이트, 예를 들어 `0xFF` 같은 값은 음수로 해석될 수 있습니다.

```c
char c = (char)0xFF;

isspace(c); /* 정의되지 않은 동작이 될 수 있습니다. */
```

`isspace` 같은 함수는 임의의 음수 값을 받을 수 없습니다. 음수 중 허용되는 값은 `EOF`뿐입니다.

따라서 바이트 값을 검사할 때는 먼저 `unsigned char` 범위로 변환합니다.

```c
isspace((unsigned char)c);
```

다만 `EOF`를 직접 검사해야 하는 경우에는 주의해야 합니다. 예를 들어 `fgetc`의 반환값은 `char`가 아니라 `int`로 받아야 합니다.

```c
int c = fgetc(file);

if (c != EOF && isspace((unsigned char)c)) {
    /* 공백 문자입니다. */
}
```

`EOF`를 `char`에 저장하면 실제 문자 값과 구분할 수 없게 될 수 있으므로, 입력 함수의 반환값을 먼저 `int`로 유지하는 것이 중요합니다.

## `struct`로 함께 변하는 값 묶기

서로 관련된 여러 값을 하나의 구조체로 묶으면, 해당 값들이 하나의 상태를 표현하도록 만들 수 있습니다.

예를 들어 동적 버퍼를 다음과 같이 표현할 수 있습니다.

```c
struct buffer {
    char *data;
    size_t length;
    size_t capacity;
};
```

각 필드는 보통 다음 의미를 가집니다.

- `data`: 실제 데이터를 저장하는 메모리의 시작 주소
- `length`: 현재 사용 중인 데이터의 길이
- `capacity`: 현재 할당된 공간의 크기

이 세 값은 서로 독립적이지 않습니다. 하나가 바뀌면 다른 값도 함께 맞춰져야 합니다.

예를 들어 다음과 같은 조건을 항상 유지하도록 설계할 수 있습니다.

```c
length <= capacity
// capacity == 0이면 
data == NULL
// 문자열을 저장한다면
data[length] == '\0'
```

이처럼 객체가 정상 상태일 때 항상 만족해야 하는 조건을 보통 **불변 조건(invariant)**이라고 합니다.

첫 번째 조건은 현재 저장된 데이터가 할당된 공간을 넘지 않는다는 뜻입니다.

```c
length <= capacity
```

예를 들어 다음 상태는 잘못되었습니다.

```text
length = 10
capacity = 8
```

실제 공간보다 더 많은 데이터가 저장되어 있다고 표현하기 때문입니다.

두 번째 조건은 빈 버퍼를 하나의 일관된 형태로 표현하기 위한 규칙입니다.

```c
// capacity == 0
data == NULL
```

예를 들어 초기 상태를 다음처럼 정할 수 있습니다.

```c
struct buffer buffer = {
    .data = NULL,
    .length = 0,
    .capacity = 0,
};
```

하지만 이 조건은 언어가 요구하는 규칙은 아닙니다. 구현이 선택한 설계 규칙입니다. `capacity == 0`이면서 `data`가 `NULL`이 아닌 구현도 이론적으로 가능합니다.

문자열을 저장하는 버퍼라면 다음 조건도 필요할 수 있습니다.

```c
data[length] == '\0'
```

여기서 중요한 점은 `length`가 보통 널 종료 문자를 제외한 문자열 길이라는 것입니다.

예를 들어 `"hello"`를 저장한다면 상태는 다음과 같을 수 있습니다.

```text
data:
'h' 'e' 'l' 'l' 'o' '\0'

length = 5
capacity >= 6
```

따라서 문자열용 버퍼에서는 실제로 최소 `length + 1`바이트의 공간이 필요합니다.

이 경우 더 정확한 조건은 다음과 같습니다.

```c
length + 1 <= capacity
data[length] == '\0'
```

단, `capacity`를 "널 종료 문자까지 포함한 전체 할당 크기"로 정의했을 때 그렇습니다.

구조체를 선언했다고 이러한 조건이 자동으로 유지되는 것은 아닙니다.

예를 들어 다음 코드는 구조체 형식상으로는 유효하지만 상태는 잘못되었습니다.

```c
struct buffer buffer = {
    .data = NULL,
    .length = 100,
    .capacity = 0,
};
```

따라서 구조체를 다루는 모든 함수가 같은 규칙을 지켜야 합니다.

```c
buffer_init(&buffer);
buffer_append(&buffer, data, length);
buffer_reserve(&buffer, capacity);
buffer_clear(&buffer);
buffer_destroy(&buffer);
```

예를 들어 `buffer_append`가 데이터를 추가했다면 다음을 모두 함께 갱신해야 할 수 있습니다.

- 필요한 경우 `data`를 더 큰 메모리로 재할당
- `capacity` 갱신
- 새 데이터 복사
- `length` 갱신
- 문자열 버퍼라면 `data[length] = '\0'` 설정

핵심은 `struct`가 단순히 관련된 값을 묶어 줄 뿐이라는 점입니다. 그 값들 사이의 관계를 실제로 유지하는 것은 해당 구조체를 초기화하고 수정하고 정리하는 코드입니다.

## 함수 이름은 실제 동작을 드러내기

함수 이름은 함수가 **무엇을 입력받아 어떤 작업을 수행하는지** 가능한 한 구체적으로 드러내는 편이 좋습니다.

예를 들어 다음과 같은 작업이 있다고 가정합니다.

```text
문자열을 정수로 변환합니다.
통계 값에 새 정수를 추가합니다.
결과를 stdout에 출력합니다.
동적 버퍼의 크기를 늘립니다.
파일에서 남은 바이트를 읽습니다.
```

이를 함수 이름으로 표현하면 다음처럼 만들 수 있습니다.

```c
parse_integer(...)
stats_add_value(...)
print_result(...)
buffer_grow(...)
read_remaining_bytes(...)
```

이런 이름은 호출부만 보더라도 대략적인 동작을 추론할 수 있습니다.

```c
value = parse_integer(text);
stats_add_value(&stats, value);
print_result(&stats);
```

반대로 다음과 같은 이름은 동작을 거의 설명하지 못합니다.

```c
process(...)
handle(...)
manage(...)
```

문제는 이 단어들이 항상 나쁘다는 것이 아니라, **대상과 결과를 함께 나타내지 않으면 의미가 지나치게 넓어진다는 점**입니다.

예를 들어 다음 이름은 비교적 구체적입니다.

```c
handle_client_connection(...)
process_input_line(...)
manage_worker_threads(...)
```

여기서는 무엇을 처리하는지가 드러납니다.

따라서 단순히 `process`, `handle`, `manage`라는 단어를 금지하기보다는 다음을 확인하는 편이 정확합니다.

- 무엇을 대상으로 하는 함수인지 알 수 있는가?
- 함수가 읽는지, 변환하는지, 추가하는지, 출력하는지 구분되는가?
- 호출부만 보고도 반환값이나 주요 부수 효과를 어느 정도 예상할 수 있는가?

예를 들어 다음 두 함수는 의미 차이가 명확합니다.

```c
buffer_reserve(...)
buffer_append(...)
```

`buffer_reserve`는 저장 공간을 확보하는 함수이고, `buffer_append`는 실제 데이터를 추가하는 함수라는 점을 이름만으로 구분할 수 있습니다.

함수 이름은 구현 세부사항을 모두 설명할 필요는 없습니다. 대신 호출자가 알아야 하는 **주요 동작**이 드러나야 합니다.

## 반환값과 출력 매개변수

C 함수는 `return` 문으로 하나의 값만 직접 반환할 수 있습니다. 그래서 함수가 **성공했는지 여부**와 **실제로 계산하거나 조회한 결과**를 모두 전달해야 할 때는, 반환값과 출력 매개변수(output parameter)를 나누어 사용하는 방식이 흔합니다.

```c
int vector_get(
    const struct vector *vector,
    size_t index,
    int *out_value
) {
    if (vector == NULL || out_value == NULL || index >= vector->size) {
        return -1;
    }

    *out_value = vector->data[index];
    return 0;
}
```

이 함수는 호출자에게 두 종류의 정보를 전달합니다.

- 반환값: 함수 실행의 성공 여부
- `*out_value`: 성공했을 때 조회한 실제 값

`out_value`에는 결과 자체가 아니라 **결과를 저장할 변수의 주소**가 전달됩니다. 함수는 이 주소를 역참조하여 호출자의 변수에 값을 기록합니다.

사용 예시는 다음과 같습니다.

```c
int value;

if (vector_get(vector, 3, &value) == 0) {
    printf("%d\n", value);
}
```

여기서 `&value`는 `value`의 주소입니다. `vector_get()`이 성공하면 내부의

```c
*out_value = vector->data[index];
```

가 실행되고, 결국 호출자의 `value`가 변경됩니다.

즉 개념적으로는 다음과 같습니다.

```text
반환값
    → 함수 자체의 실행 결과

출력 매개변수
    → 함수가 만들어 낸 실제 데이터
```

### 실패했을 때 출력 매개변수를 어떻게 처리할 것인가

이런 인터페이스에서 중요한 설계 선택 중 하나는 **함수가 실패했을 때 출력 매개변수의 값을 어떻게 할 것인지**입니다.

직접 설계하는 API라면 다음 규칙이 특히 다루기 쉽습니다.

```text
성공:
    반환값 == 0
    *out_value에 유효한 결과 저장

실패:
    반환값 != 0
    *out_value를 변경하지 않음
```

예를 들어 다음 호출을 생각해 보겠습니다.

```c
int value = 42;

if (vector_get(vector, 100, &value) != 0) {
// vector.size의 값이 100보다 작다면 value는 여전히 42입니다.
}
```

`100`이 유효하지 않은 인덱스라면 함수는 `-1`을 반환합니다. 이때 함수가 `out_value`에 아무것도 쓰지 않았으므로 `value`는 기존 값인 `42`를 유지합니다.

이를 보장하려면 **출력 매개변수에 값을 쓰기 전에 실패할 수 있는 조건을 먼저 검사해야 합니다.**

```c
if (vector == NULL || out_value == NULL || index >= vector->size) {
    return -1;
}

*out_value = vector->data[index];
return 0;
```

이 순서에는 중요한 의미가 있습니다. 위 함수에서는 `*out_value`에 쓰는 순간 이후에는 실패하지 않습니다. 따라서 호출자는

```text
성공했다 → out_value에 새 값이 있다.
실패했다 → out_value는 건드리지 않았다.
```

라고 단순하게 판단할 수 있습니다.

반대로 다음 구현은 문제가 있습니다.

```c
*out_value = 0;

if (index >= vector->size) {
    return -1;
}
```

함수는 실패했지만 호출자가 전달한 변수는 이미 `0`으로 변경되었습니다.

```c
int value = 42;

if (vector_get(vector, 100, &value) != 0) {
    /* 함수는 실패했지만 value는 0으로 바뀌었을 수 있습니다. */
}
```

이렇게 되면 호출자는 단순히 반환값만 확인해서는 출력 변수의 상태를 알 수 없습니다.

### 검증 순서에서 주의할 점

`out_value == NULL` 검사도 중요한 이유가 있습니다.

```c
*out_value = vector->data[index];
```

를 실행하려면 `out_value`가 실제로 쓰기 가능한 `int` 객체를 가리켜야 합니다. `NULL`을 역참조하면 정의되지 않은 동작(undefined behavior)이 발생합니다.

마찬가지로 `vector->size`를 읽기 전에 `vector == NULL`인지 먼저 확인되어야 합니다.

```c
if (vector == NULL || out_value == NULL || index >= vector->size)
```

이 표현이 안전한 이유는 C의 `||` 연산자가 **왼쪽에서 오른쪽으로 평가되고, 결과가 이미 참이면 나머지를 평가하지 않는 단락 평가(short-circuit evaluation)**를 하기 때문입니다.

따라서 `vector == NULL`이면

```c
index >= vector->size
```

는 아예 평가되지 않습니다. `NULL`인 `vector`를 역참조하지 않는 것입니다.

### 출력값을 사용해도 되는 시점

출력 매개변수의 값은 **함수가 성공했다는 사실을 확인한 뒤에만 사용해야 합니다.**

```c
int value;

if (vector_get(vector, 3, &value) == 0) {
    printf("%d\n", value);
}
```

반대로 다음 코드는 잘못될 수 있습니다.

```c
int value;

vector_get(vector, 3, &value);
printf("%d\n", value);
```

함수가 실패했다면 `value`에는 유효한 결과가 저장되지 않았습니다. 특히 `value`를 초기화하지 않았다면 그 값을 읽는 것 자체가 올바르지 않습니다.

따라서 출력 매개변수를 사용하는 API에서는 보통 다음 관계를 기억하면 됩니다.

```text
반환값을 먼저 확인한다.
        ↓
성공한 경우에만
        ↓
출력 매개변수에 저장된 결과를 사용한다.
```

### 테스트

실패 시 출력 매개변수를 변경하지 않는다는 규칙은 테스트로 명확하게 검증할 수 있습니다.

```c
int value = 123;

assert(vector_get(vector, invalid_index, &value) == -1);
assert(value == 123);
```

첫 번째 `assert`는 함수가 실패했음을 확인하고, 두 번째 `assert`는 실패 과정에서 출력 변수가 변경되지 않았음을 확인합니다.

다만 **실패 시 출력 매개변수를 변경하지 않는 것은 C 언어 자체의 규칙이 아닙니다.** 함수 인터페이스를 설계한 사람이 정하는 규칙입니다.

어떤 API는 실패했을 때 출력 매개변수의 값을 보장하지 않을 수도 있고, 일부 API는 실패 원인이나 부분 결과를 출력 매개변수에 기록하기도 합니다. 따라서 외부 API를 사용할 때는 문서에서 해당 동작을 확인해야 합니다.

직접 함수를 설계한다면, 특별한 이유가 없는 한 다음처럼 정의하는 편이 호출자 입장에서 단순합니다.

```text
성공하면 출력 매개변수에 유효한 값을 기록한다.
실패하면 출력 매개변수를 변경하지 않는다.
```

핵심은 **반환값과 출력 매개변수가 각각 무엇을 의미하는지, 그리고 실패했을 때 출력 매개변수의 상태가 어떻게 되는지를 명확하게 정의하는 것**입니다.

## 완료 기준

1. 함수 선언만 보고 입력과 결과를 설명합니다.
2. 배열을 함수에 전달할 때 배열의 길이도 함께 전달합니다.
3. 문자열 저장 공간을 계산할 때 마지막 `NUL` 바이트를 포함합니다.
4. 문자열 literal을 수정하지 않습니다.
5. `ctype` 함수에 일반 `char`값을 전달할 때 `unsigned char`로 변환합니다.
6. 함께 변하는 상태를 구조체로 묶고, 유효한 상태가 만족해야 하는 조건을 명시합니다.
