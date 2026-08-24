# 빌드, 링크, 테스트

빌드 도구를 사용하는 목적은 Makefile 문법을 외우는 것이 아닙니다. 핵심은 **어떤 입력으로 어떤 산출물을 만들고, 입력이 바뀌었을 때 무엇을 다시 만들어야 하는지 정확하게 표현하는 것**입니다.

Makefile은 결국 다음 관계를 적는 파일입니다.

```text
입력 파일
→ 변환 명령
→ 출력 파일
```

그리고 입력이 출력보다 새로우면 필요한 명령만 다시 실행합니다.

## 직접 명령부터 확인하기

Makefile을 작성하기 전에 실제 빌드 명령을 직접 실행해 보는 것이 좋습니다.

```sh
cc -Iinclude -std=c11 -Wall -Wextra -Wpedantic \
    -c src/text.c -o build/text.o

ar rcs build/libtext.a build/text.o

cc -Iinclude app/main.c build/libtext.a -o build/text-report

cc -Iinclude tests/test_text.c build/libtext.a -o build/test-text

./build/test-text
```

각 명령의 역할은 다음과 같습니다.

```text
src/text.c
→ compile
→ build/text.o

build/text.o
→ archive
→ build/libtext.a

app/main.c + build/libtext.a
→ compile + link
→ build/text-report

tests/test_text.c + build/libtext.a
→ compile + link
→ build/test-text

build/test-text
→ 실행
→ 테스트 결과
```

Makefile은 이 명령들을 대신 만들어 주는 것이 아니라, **이 파일들 사이의 의존 관계를 기록하고 필요한 명령만 선택적으로 실행하도록 만드는 도구**입니다.

## 빌드는 입력과 결과의 관계입니다

예를 들어 다음 구조를 생각해 봅니다.

```text
src/text.c + include/text.h
    → build/obj/text.o

build/obj/text.o
    → build/libtext.a

app/main.c + include/text.h + build/libtext.a
    → build/text-report
```

`include/text.h`가 바뀌면 이 헤더를 포함하는 번역 단위를 다시 컴파일해야 합니다.

예를 들어:

```text
text.c → text.o
main.c → main.o
```

두 파일 모두 `text.h`를 포함한다면 `text.h` 변경 후 둘 다 다시 컴파일되어야 합니다.

그 결과 `text.o`가 바뀌면:

```text
text.o
→ libtext.a 재생성
→ text-report 재링크
```

가 이어질 수 있습니다.

즉 Makefile의 핵심은 명령 나열이 아니라 **변경 전파 관계를 정확하게 표현하는 것**입니다.

## target, prerequisite, recipe

Makefile의 기본 규칙은 다음 형태입니다.

```make
target: prerequisite
	recipe
```

예:

```make
build/obj/text.o: src/text.c include/text.h
	@mkdir -p build/obj
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@
```

각 부분은 다음 의미입니다.

```text
target
    build/obj/text.o

prerequisite
    src/text.c
    include/text.h

recipe
    text.o를 만드는 명령
```

`make`는 prerequisite 중 하나라도 target보다 새로우면 recipe를 다시 실행합니다.

즉:

```text
text.c 수정
→ text.o보다 새로움
→ text.o 다시 컴파일
```

이 됩니다.

## 자동 변수

Makefile에서는 현재 규칙의 파일 이름을 자동 변수로 참조할 수 있습니다.

```make
$@
```

현재 target입니다.

```make
$<
```

첫 번째 prerequisite입니다.

```make
$^
```

중복을 제거한 모든 일반 prerequisite입니다.

예:

```make
build/obj/text.o: src/text.c include/text.h
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@
```

여기서:

```text
$@ → build/obj/text.o
$< → src/text.c
$^ → src/text.c include/text.h
```

입니다.

레시피 줄은 일반적으로 앞에 **탭(tab)** 을 사용해야 합니다.

## 컴파일 옵션과 링크 옵션

옵션은 어느 단계에 영향을 주는지 구분하는 것이 좋습니다.

```make
CC ?= cc

CPPFLAGS ?= -Iinclude

CFLAGS ?= -std=c11 -Wall -Wextra -Wpedantic -Werror -O2

LDFLAGS ?=

LDLIBS ?=
```

### `CPPFLAGS`

전처리 단계에 필요한 옵션을 둡니다.

예:

```make
CPPFLAGS += -Iinclude
CPPFLAGS += -DDEBUG
```

주로 다음이 들어갑니다.

```text
-I...
-D...
-U...
```

### `CFLAGS`

C 소스를 컴파일할 때 사용하는 옵션입니다.

```make
CFLAGS += -std=c11
CFLAGS += -Wall -Wextra -Wpedantic -Werror
CFLAGS += -O2
```

예를 들어:

* 언어 표준
* 경고 옵션
* 최적화
* 디버그 정보
* sanitizer instrumentation

등이 들어갈 수 있습니다.

### `LDFLAGS`

링커 동작에 영향을 주는 옵션입니다.

예:

```make
LDFLAGS += -L/path/to/lib
```

또는 sanitizer처럼 링크 단계에서도 필요한 옵션을 둘 수 있습니다.

### `LDLIBS`

링크할 라이브러리를 적습니다.

예:

```make
LDLIBS += -lm
LDLIBS += -lpthread
```

보통 실제 링크 명령은 다음처럼 구성합니다.

```make
$(CC) $(LDFLAGS) objects... $(LDLIBS) -o target
```

`-pthread`처럼 컴파일 단계와 링크 단계 모두에 영향을 줄 수 있는 옵션도 있습니다. 이런 옵션은 사용하는 컴파일러의 문서를 확인하고 필요한 단계에 모두 전달해야 합니다.

## 패턴 규칙

소스 파일이 여러 개라면 개별 규칙을 반복해서 적는 대신 패턴 규칙을 사용할 수 있습니다.

```make
OBJ_DIR := build/obj

SOURCES := \
	src/owned_string.c \
	src/buffer.c

OBJECTS := $(SOURCES:src/%.c=$(OBJ_DIR)/%.o)
```

그리고:

```make
$(OBJ_DIR)/%.o: src/%.c
	@mkdir -p $(dir $@)
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@
```

와 같이 작성할 수 있습니다.

예를 들어:

```text
src/owned_string.c
```

는:

```text
build/obj/owned_string.o
```

에 대응합니다.

패턴 규칙의 `%`는 파일 이름의 공통 부분을 나타냅니다.

## 헤더 의존성

다음 규칙만 있다고 가정합니다.

```make
$(OBJ_DIR)/%.o: src/%.c
```

이 경우 `make`는 `.c` 파일이 바뀌었는지는 알 수 있지만, 그 `.c` 파일이 포함하는 헤더가 바뀌었는지는 자동으로 알 수 없습니다.

작은 프로젝트에서는 직접 적을 수 있습니다.

```make
build/obj/text.o: src/text.c include/text.h
```

하지만 파일 수가 늘어나면 모든 `#include` 관계를 사람이 유지하기 어렵습니다.

그래서 컴파일러가 의존성 파일을 생성하게 할 수 있습니다.

```make
DEPFLAGS := -MMD -MP

$(OBJ_DIR)/%.o: src/%.c
	@mkdir -p $(dir $@)
	$(CC) $(CPPFLAGS) $(CFLAGS) $(DEPFLAGS) -c $< -o $@
```

이렇게 하면 일반적으로 `.o` 옆에 `.d` 파일이 만들어집니다.

예:

```text
build/obj/text.o
build/obj/text.d
```

`text.d`에는 개념적으로 다음과 같은 정보가 들어갑니다.

```make
build/obj/text.o: src/text.c include/text.h
```

이를 다시 Makefile에서 읽습니다.

```make
-include $(OBJECTS:.o=.d)
```

`-include`의 앞 `-`는 파일이 아직 존재하지 않아도 오류로 중단하지 않게 합니다.

첫 빌드에서는 `.d` 파일이 아직 없기 때문입니다.

## `-MMD`와 `-MP`

`-MMD`는 사용자 헤더를 중심으로 의존성 정보를 생성합니다.

`-MP`는 헤더가 삭제되었을 때 오래된 `.d` 파일 때문에 `make`가 즉시 실패하는 문제를 줄이기 위해 빈 규칙을 함께 생성합니다.

따라서 다음 조합이 자주 사용됩니다.

```make
DEPFLAGS := -MMD -MP
```

## 정적 라이브러리

여러 오브젝트 파일을 하나의 정적 라이브러리로 묶을 수 있습니다.

```make
AR ?= ar
ARFLAGS ?= rcs

LIB := build/libowned_string.a

$(LIB): $(OBJECTS)
	@mkdir -p $(dir $@)
	rm -f $@
	$(AR) $(ARFLAGS) $@ $^
```

`ar`는 여러 오브젝트 파일을 하나의 archive에 저장합니다.

예:

```text
owned_string.o
buffer.o
allocator.o
    ↓
libowned_string.a
```

정적 라이브러리는 실행 파일이 아닙니다. 링크할 때 필요한 오브젝트 코드를 제공하는 archive입니다.

## 왜 기존 archive를 지우는가

다음 소스 목록으로 라이브러리를 만들었다고 가정합니다.

```text
a.o
b.o
c.o
```

이후 `c.o`를 프로젝트에서 제거했습니다.

새로:

```sh
ar rcs libexample.a a.o b.o
```

만 실행해도 기존 archive 안에 `c.o`가 남을 수 있습니다.

즉 현재 소스 목록과 archive 내부 멤버가 일치하지 않을 수 있습니다.

그래서 다음처럼 기존 파일을 먼저 지우는 방법이 단순합니다.

```make
rm -f $@
$(AR) $(ARFLAGS) $@ $^
```

그러면 매번 현재 `$(OBJECTS)`만 들어 있는 archive를 만들 수 있습니다.

실제 멤버를 확인하려면:

```sh
ar t build/libowned_string.a
```

를 사용합니다.

심볼은:

```sh
nm build/libowned_string.a
```

로 확인할 수 있습니다.

## 정적 라이브러리와 링크

예를 들어 `main.o`가 `text_length`를 사용하고 `libtext.a`가 이를 정의한다고 가정합니다.

```text
main.o
    undefined: text_length

libtext.a
    defined: text_length
```

다음처럼 링크할 수 있습니다.

```sh
cc build/main.o build/libtext.a -o build/text-report
```

전통적인 Unix 정적 링크에서는 링커가 명령행을 왼쪽에서 오른쪽으로 처리하면서 **현재까지 해결되지 않은 심볼을 기준으로 archive에서 필요한 멤버를 선택하는 경우가 많습니다.**

그래서 일반적으로:

```text
심볼을 사용하는 object
→ 그 심볼을 제공하는 library
```

순서로 둡니다.

```sh
cc main.o libtext.a -o program
```

반대로:

```sh
cc libtext.a main.o -o program
```

는 환경과 링커 동작에 따라 필요한 archive 멤버가 선택되지 않아 링크가 실패할 수 있습니다.

이 규칙은 특히 정적 라이브러리를 사용할 때 중요합니다.

## 실행 파일 규칙

예를 들어:

```make
APP := build/text-report

APP_OBJECTS := build/obj/main.o

$(APP): $(APP_OBJECTS) $(LIB)
	$(CC) $(LDFLAGS) $(APP_OBJECTS) $(LIB) $(LDLIBS) -o $@
```

처럼 작성할 수 있습니다.

여기서 `$(LIB)`가 prerequisite이므로 라이브러리가 새로 만들어지면 실행 파일도 다시 링크됩니다.

## 재링크와 재빌드 구분

소스 하나가 바뀌었다고 해서 모든 파일을 다시 컴파일할 필요는 없습니다.

예를 들어:

```text
src/text.c 수정
```

이라면 이상적인 동작은:

```text
text.o 재컴파일
→ libtext.a 재생성
→ 관련 실행 파일 재링크
```

입니다.

하지만 `main.c`가 바뀌지 않았다면 `main.o`는 다시 컴파일할 필요가 없습니다.

즉 다음을 구분해야 합니다.

```text
재컴파일
→ .c → .o

재링크
→ .o / .a → executable
```

## 불필요한 재빌드 확인

빌드가 끝난 직후 다음을 실행합니다.

```sh
make
make
```

두 번째 `make`에서 아무것도 다시 만들지 않는 것이 정상적인 목표입니다.

보통:

```text
make: Nothing to be done for 'all'.
```

같은 결과가 나옵니다.

두 번째 실행에서도 빌드가 반복되면 다음을 확인합니다.

* target 파일이 실제로 생성되는가
* target 이름이 실제 출력 파일 이름과 같은가
* `.PHONY`를 파일 target에 잘못 사용하지 않았는가
* prerequisite 중 항상 수정 시간이 바뀌는 파일이 있는가
* recipe가 target보다 최신인 파일을 매번 생성하는가
* timestamp를 강제로 갱신하는 명령이 있는가

## `.PHONY`

다음 target들은 실제 파일을 만들기 위한 이름이 아니라 명령 그룹을 나타냅니다.

```make
.PHONY: all test sanitize clean fclean re
```

예:

```make
test: $(TEST_BIN)
	$(TEST_BIN)
```

`test`라는 파일을 만드는 것이 목적이 아니라 테스트를 실행하는 것이 목적입니다.

그래서 `.PHONY`로 선언합니다.

반대로:

```make
.PHONY: build/libtext.a
```

처럼 실제 산출물 파일을 phony로 만들면 `make`는 해당 target이 항상 오래되었다고 판단하여 매번 다시 만듭니다.

## 테스트 target

테스트 실행 파일도 일반 프로그램처럼 빌드 산출물입니다.

```make
TEST_BIN := build/test-text
TEST_OBJECTS := build/obj/test_text.o

$(TEST_BIN): $(TEST_OBJECTS) $(LIB)
	$(CC) $(LDFLAGS) $(TEST_OBJECTS) $(LIB) $(LDLIBS) -o $@

test: $(TEST_BIN)
	$(TEST_BIN)
```

테스트가 제품 코드를 직접 복제해서 사용해서는 안 됩니다.

가능하면 테스트 실행 파일은 실제 제품 라이브러리를 링크합니다.

```text
제품:
    libtext.a

테스트:
    test_text.o + libtext.a
```

그래야 테스트가 실제 배포 대상과 같은 구현을 검사합니다.

## 테스트의 종료 상태

`make test`는 테스트 실행 파일을 실행하는 것뿐 아니라 **실패 시 Makefile도 실패해야 합니다.**

예:

```c
int main(void)
{
    if (test_something() != 0) {
        return 1;
    }

    return 0;
}
```

테스트 프로그램이 non-zero로 종료하면:

```make
test:
	$(TEST_BIN)
```

의 recipe도 실패하고 `make test` 역시 실패합니다.

따라서 테스트가 실패했는데도 항상 `0`으로 종료하게 만들면 자동화에서 실패를 감지할 수 없습니다.

## Sanitizer

AddressSanitizer와 UndefinedBehaviorSanitizer를 사용할 수 있습니다.

```make
SANITIZE_FLAGS := \
	-fsanitize=address,undefined \
	-fno-omit-frame-pointer \
	-g
```

컴파일할 때 instrumentation을 넣어야 하고, 링크할 때 sanitizer runtime도 연결해야 합니다.

따라서 보통 양쪽에 적용합니다.

```make
sanitize: CFLAGS += $(SANITIZE_FLAGS)
sanitize: LDFLAGS += $(SANITIZE_FLAGS)
```

예:

```make
sanitize: CFLAGS += $(SANITIZE_FLAGS)
sanitize: LDFLAGS += $(SANITIZE_FLAGS)

sanitize: fclean $(TEST_BIN)
	$(TEST_BIN)
```

## 왜 sanitizer 빌드와 일반 빌드를 섞으면 안 되는가

다음 순서를 생각해 봅니다.

```sh
make
make sanitize
```

이미 일반 `CFLAGS`로 만들어진 `.o` 파일이 존재하면 `make`는 소스가 바뀌지 않았다고 판단하여 이를 재사용할 수 있습니다.

하지만 sanitizer 옵션은 **컴파일 명령이 달라진 것**이지 파일 timestamp가 달라진 것이 아닙니다.

기본 Make는 CFLAGS 변경을 자동 의존성으로 추적하지 않습니다.

따라서 일반 오브젝트를 그대로 링크하면 일부 코드에 sanitizer instrumentation이 들어가지 않을 수 있습니다.

그래서 다음 중 하나를 사용합니다.

### 방법 1: 먼저 정리

```make
sanitize: fclean $(TEST_BIN)
```

단순하지만 전체를 다시 빌드합니다.

### 방법 2: 별도 빌드 디렉터리

```text
build/obj/
build/sanitize/obj/
```

처럼 분리합니다.

규모가 커질수록 두 번째 방식이 더 안정적입니다.

## ThreadSanitizer

ThreadSanitizer는 데이터 경쟁을 찾는 데 사용할 수 있습니다.

예:

```sh
-fsanitize=thread
```

하지만 다음에 따라 지원 여부나 동작이 달라질 수 있습니다.

* 컴파일러
* CPU 아키텍처
* 운영체제
* runtime
* 다른 sanitizer와의 조합

지원하지 않는 환경에서 실행이 실패했을 때:

```sh
./test || true
```

처럼 실패를 무시하면 안 됩니다.

그러면 실제 테스트 실패도 성공처럼 보일 수 있습니다.

대신:

```text
지원되지 않음
```

과:

```text
테스트가 실패함
```

을 구분해서 기록하는 것이 좋습니다.

## 정리 target

빌드 과정에서 생성한 파일만 삭제합니다.

```make
clean:
	rm -rf build/obj

fclean: clean
	rm -rf build

re: fclean all
```

다음 차이를 정할 수 있습니다.

```text
clean
→ 중간 산출물 제거

fclean
→ 모든 빌드 산출물 제거

re
→ 완전히 정리 후 다시 빌드
```

프로젝트마다 정의는 약간 다를 수 있지만 일관되게 사용해야 합니다.

중요한 것은 다음을 지우지 않는 것입니다.

* 소스 코드
* 헤더
* 테스트 소스
* 테스트 fixture
* 직접 작성한 설정 파일

즉 Makefile이 만든 산출물만 제거해야 합니다.

## 테스트는 정상 입력만 확인하지 않습니다

다음 테스트만 있다면:

```text
입력: "hello"
출력: 정상
```

정상 경로만 확인한 것입니다.

실제 구현 오류는 실패 경로나 극단 조건에서 많이 드러납니다.

예를 들어 다음을 확인합니다.

* `NULL` 인자
* 잘못된 옵션
* 빈 입력
* 첫 원소
* 마지막 원소
* 범위 밖 index
* 최솟값과 최댓값
* overflow 직전 값
* overflow를 일으키는 값
* 첫 할당 실패
* 이후 재할당 실패
* 실패 후 기존 상태 보존
* 파일 열기 실패
* 부분 읽기와 부분 쓰기
* 파일 디스크립터 누수
* 자식 프로세스 비정상 종료
* 예상과 다른 종료 코드
* timeout
* 교착 상태
* 동시 접근에서 유지해야 하는 값

좋은 테스트는 단순히 "통과했다"가 아니라 **어떤 잘못된 구현을 잡을 수 있는지 설명할 수 있어야 합니다.**

## 테스트와 구현 세부사항

테스트는 가능하면 공개 동작을 검사하는 것이 좋습니다.

예를 들어 동적 배열의 내부 구현이:

```text
capacity 2배 증가
```

인지:

```text
capacity 1.5배 증가
```

인지는 API가 이를 공개하지 않는다면 테스트에서 고정할 필요가 없습니다.

대신 다음을 확인합니다.

```text
삽입 성공
기존 값 유지
새 값 추가
실패 후 상태 보존
```

즉 구현 세부사항이 아니라 **외부에서 관찰 가능한 동작**을 중심으로 테스트합니다.

다만 자료구조 자체를 학습하기 위한 프로젝트라면 내부 불변식 검증을 별도 테스트로 둘 수도 있습니다.

## 파일 디스크립터 누수

파일이나 pipe를 사용하는 프로그램이라면 메모리만 확인해서는 부족합니다.

예:

```text
open
pipe
dup
socket
```

등으로 얻은 descriptor도 제한된 자원입니다.

반복 실행 후 descriptor가 계속 늘어난다면 누수가 있을 수 있습니다.

테스트에서는 정상 경로뿐 아니라 실패 경로에서도:

```text
open 성공
→ 이후 처리 실패
→ fd close 되었는가
```

를 확인해야 합니다.

## 자식 프로세스 테스트

`fork`, `exec`, pipe 등을 사용하는 프로그램에서는 출력만 맞는지 확인하는 것으로 부족합니다.

다음도 확인해야 합니다.

* `wait` 또는 `waitpid` 수행 여부
* 자식의 정상 종료 여부
* 종료 코드
* signal 종료 여부
* zombie가 남지 않는지
* pipe 끝을 올바르게 닫았는지

예를 들어:

```text
출력은 맞음
하지만 child exit status는 실패
```

라면 정상 동작으로 볼 수 없습니다.

## timeout과 교착 상태

동시성 또는 IPC 테스트는 잘못된 구현이 영원히 끝나지 않을 수 있습니다.

예:

```text
deadlock
무한 대기
pipe EOF가 오지 않음
condition variable을 깨우지 못함
```

이런 테스트는 timeout이 필요할 수 있습니다.

하지만 timeout은 단순히 테스트 속도를 제한하는 것이 아니라:

> 프로그램이 정해진 시간 안에 진행하지 못했다는 실패 조건

으로 사용해야 합니다.

## 독립 실행 확인

exercise나 작은 프로젝트는 부모 저장소 없이도 동작하는지 확인하는 것이 좋습니다.

```sh
cp -R exercises/owned-string /tmp/owned-string

cd /tmp/owned-string

make
make test
```

여기서 빌드나 테스트가 실패한다면 다음과 같은 숨은 의존성이 있을 수 있습니다.

* 부모 디렉터리의 헤더
* 부모 Makefile의 변수
* 공용 shell script
* 다른 exercise의 코드
* repository root 기준 상대 경로
* Git에만 존재하는 생성 파일
* 로컬 환경에 우연히 설치된 파일

독립 프로젝트라면 해당 디렉터리 자체에 필요한 파일이 모두 존재해야 합니다.

## 깨끗한 환경에서 확인하기

현재 작업 디렉터리에서는 오래된 산출물이 문제를 숨길 수 있습니다.

따라서 다음 순서를 확인하는 것이 좋습니다.

```sh
make fclean
make
make test
make
```

의미는 다음과 같습니다.

```text
make fclean
→ 이전 산출물 없이 시작

make
→ 처음부터 정상 빌드

make test
→ 테스트 빌드 및 실행

make
→ 테스트 후에도 불필요한 rebuild가 없는지 확인
```

가능하다면 프로젝트 디렉터리를 다른 위치에 복사해서도 확인합니다.

## Makefile을 볼 때 확인할 질문

```text
입력:
    어떤 .c와 .h가 필요한가?

중간 산출물:
    어떤 .o가 생성되는가?

최종 산출물:
    실행 파일인가?
    정적 라이브러리인가?
    둘 다인가?

변경 전파:
    헤더가 바뀌면 어떤 object가 다시 만들어지는가?
    object가 바뀌면 어떤 library가 다시 만들어지는가?
    library가 바뀌면 어떤 executable이 다시 링크되는가?

옵션:
    전처리 옵션인가?
    컴파일 옵션인가?
    링크 옵션인가?
    링크할 library인가?

테스트:
    어떤 binary를 실행하는가?
    실패 시 non-zero로 종료하는가?

정리:
    Makefile이 생성한 파일만 삭제하는가?

재빌드:
    아무 입력도 바뀌지 않았을 때 두 번째 make가 아무 작업도 하지 않는가?
```

## 완료 기준

1. `source → object → library/executable` 관계를 Makefile에 표현합니다.
2. target, prerequisite, recipe의 역할을 설명합니다.
3. 헤더가 변경되면 이를 포함한 번역 단위가 다시 컴파일되게 합니다.
4. 직접 헤더 의존성을 적는 방식과 `.d` 파일을 생성하는 방식의 차이를 설명합니다.
5. `CPPFLAGS`, `CFLAGS`, `LDFLAGS`, `LDLIBS`를 구분합니다.
6. 정적 라이브러리가 여러 오브젝트 파일을 묶은 archive임을 설명합니다.
7. 소스 목록에서 제거된 오래된 archive 멤버가 남지 않도록 합니다.
8. 정적 라이브러리의 링크 순서가 중요한 이유를 설명합니다.
9. 재컴파일과 재링크의 차이를 설명합니다.
10. 실제 파일 target에 `.PHONY`를 사용하지 않습니다.
11. `make test`가 테스트 실패 시 non-zero로 실패하도록 합니다.
12. 테스트가 실제 제품 라이브러리를 사용해 링크되게 합니다.
13. sanitizer 옵션이 컴파일과 링크 양쪽에 필요할 수 있음을 설명합니다.
14. 일반 빌드 오브젝트와 sanitizer 오브젝트를 섞지 않습니다.
15. `make clean`, `make fclean`, `make re`가 무엇을 지우는지 명확히 정합니다.
16. 정상 입력뿐 아니라 실패 경로와 경계값을 테스트합니다.
17. 테스트가 어떤 잘못된 구현을 잡기 위한 것인지 설명할 수 있습니다.
18. 두 번째 `make`에서 불필요한 재빌드가 발생하지 않는지 확인합니다.
19. 깨끗한 상태에서도 `make`와 `make test`가 성공하는지 확인합니다.
20. 프로젝트 디렉터리만 복사해도 부모 저장소 없이 빌드하고 테스트할 수 있게 합니다.
