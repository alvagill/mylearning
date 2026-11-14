# 《Learn C The Hard Way》知识点总结（复习版）

> 覆盖范围：练习 0~22、24~25、27~33（练习 23 Duff's Device、26 logfind 不在要求内）
>
> 环境：Ubuntu 22.04 + gcc + make + gdb + valgrind
>
> 复习原则：编译加 `-Wall -g`；内存问题先上 valgrind；逻辑问题用 debug 输出或 gdb；看不懂就 RTFSC。

---

## 目录

1. [总览：练习与知识模块对应](#一总览练习与知识模块对应)
2. [编译、Make 与 Valgrind（0、1、2、4、28、29）](#二编译make-与-valgrind)
3. [类型、变量与格式化输出（3、6、7、21）](#三类型变量与格式化输出)
4. [控制流（8、9、10、13）](#四控制流)
5. [数组、字符串、函数与作用域（11、12、13）](#五数组字符串函数与作用域)
6. [文件 I/O（14、24）](#六文件-io)
7. [指针、结构体、堆与栈（15、16、17）](#七指针结构体堆与栈)
8. [函数指针与对象系统（18、19）](#八函数指针与对象系统)
9. [调试：dbg.h、valgrind、gdb（20、22、31）](#九调试dbghvalgrindgdb)
10. [变参函数（25）](#十变参函数)
11. [自动化测试（30）](#十一自动化测试)
12. [链表与排序（32、33）](#十二链表与排序)
13. [防御性编程（27）](#十三防御性编程)
14. [高频易错点清单](#十四高频易错点清单)
15. [常用命令速查](#十五常用命令速查)
16. [复习自测题](#十六复习自测题)

---

## 一、总览：练习与知识模块对应

| 模块 | 练习 | 关键词 |
|---|---|---|
| 环境与工具链 | 0、1、2、4 | gcc、make、`-Wall -g`、valgrind |
| 基本语法 | 3、5、6、7 | printf、main、类型、sizeof |
| 控制流 | 8、9、10、13 | if/else、while、switch、for |
| 数组与字符串 | 11、13 | 下标、`'\0'`、argv |
| 函数与作用域 | 12 | 参数、返回值、局部/全局 |
| 文件 I/O | 14、24 | fopen/fgets/fscanf/fprintf |
| 指针与内存 | 15、16、17 | `&`、`*`、struct、malloc/free |
| 函数指针 | 18 | 回调、qsort |
| 对象系统 | 19 | 结构体+函数指针、`void *self` |
| 调试 | 20、22、31 | dbg.h、valgrind、gdb |
| 变参 | 25 | stdarg.h、va_list |
| 构建与共享库 | 28、29 | Makefile、`.a`、`.so`、dlopen |
| 自动化测试 | 30 | minunit、runtests.sh |
| 数据结构与算法 | 32、33 | 双向链表、冒泡/归并排序 |
| 理论章节 | 21、27 | 数据类型/运算符、防御性编程 |

---

## 二、编译、Make 与 Valgrind

### 练习 0：环境准备

- 需要：`gcc`、`make`、`gdb`、`valgrind`、编辑器。
- 验证：`gcc --version`、`make --version`、`valgrind --version`。
- 本机已具备：gcc、make、gdb 12.1、valgrind。

### 练习 1：Dust Off That Compiler

- C 程序从 `main` 开始，`return 0` 表示成功。
- `#include <stdio.h>` 提供 `printf`、`puts` 等声明。
- 编译命令：

```bash
gcc -Wall -g ex1.c -o ex1
./ex1
```

- `-Wall`：打开常用警告；`-g`：加入调试信息（gdb 需要）。
- 如果没包含头文件，会出现 `implicit declaration of function 'puts'` 警告。这实际上是错误，不要忽略警告。

### 练习 2：用 Make 代替 Python

最基础的 Makefile：

```make
CFLAGS=-Wall -g

all: ex1

clean:
	rm -f ex1
```

要点：

- 命令行前必须是 **TAB**，不能是空格。
- `CFLAGS` 会传给 `cc`。
- make 有内置规则：知道如何从 `ex1.c` 生成 `ex1`。
- `clean` 是伪目标，用于删除产物。
- 附加题：加 `all: ex1`，这样直接 `make` 就能构建。
- 常见错误：`Makefile:4: *** missing separator. Stop.` 通常是缩进用了空格。

### 练习 4：Valgrind

```bash
valgrind ./ex4
```

常见报告：

- `Invalid write of size N`：越界写。
- `Invalid read of size N`：越界读/读已释放内存。
- `Use of uninitialised value`：使用了未初始化变量。
- `definitely lost`：确定内存泄漏。
- `indirectly lost`：因其他泄漏而不可达。
- `still reachable`：程序结束时仍可达，通常是全局/静态数据。

要点：

- valgrind 是动态分析工具，不替代编译警告。
- 越界写入常常不会立刻崩溃，但会破坏内存，必须重视。
- 编译时带 `-g`，valgrind 才能给出准确行号。

### 练习 28：Intermediate Makefiles（c-skeleton）

目录结构：

```text
c-skeleton/
├── LICENSE
├── README.md
├── Makefile
├── bin/          # 可执行文件
├── build/        # 库/目标文件
├── src/          # 源码与头文件
└── tests/        # 单元测试
```

核心 Makefile 变量：

```make
CFLAGS=-g -O2 -Wall -Wextra -Isrc -rdynamic -DNDEBUG $(OPTFLAGS)
SOURCES=$(wildcard src/**/*.c src/*.c)
OBJECTS=$(patsubst %.c,%.o,$(SOURCES))
TEST_SRC=$(wildcard tests/*_tests.c)
TESTS=$(patsubst %.c,%,$(TEST_SRC))
TARGET=build/libYOUR_LIBRARY.a
```

要点：

- `wildcard`：展开通配符，找源文件。
- `patsubst`：模式替换，`.c` → `.o`。
- `$@`：目标名；`$<`：第一个依赖；`$^`：所有依赖。
- `ar rcs $@ $(OBJECTS)`：创建静态库；`ranlib`：更新索引。
- `-fPIC`：生成位置无关代码，供共享库使用。
- `-shared`：生成 `.so`。
- `tests: CFLAGS += $(TARGET)`：让测试链接库（本项目测试用 dlopen，实际不一定需要）。
- `.PHONY: tests`：声明伪目标，避免和同名文件冲突。
- `clean` 里用 `find` 清理 `.gc*`、`.dSYM` 等临时文件。
- `PREFIX?=/usr/local`：`?=` 表示未定义时才赋值。
- `install` 目标把库安装到 `$(DESTDIR)/$(PREFIX)/lib/`。
- `check` 目标用 `egrep` 找危险函数（`strcpy`、`gets` 等）。

### 练习 29：Protecting from C's Brokenness

- 构建静态库和共享库：

```bash
make
# build/libex29.a
# build/libex29.so
```

- 共享库需要在 `$(TARGET)` 规则上加 `-fPIC`，再：

```make
$(SO_TARGET): $(TARGET) $(OBJECTS)
	$(CC) -shared -o $@ $(OBJECTS)
```

- 动态加载：

```c
#include <dlfcn.h>

void *lib = dlopen("build/libex29.so", RTLD_NOW);
if(!lib) {
    fprintf(stderr, "dlopen: %s\n", dlerror());
}

typedef int (*lib_function)(const char *msg);
lib_function func = dlsym(lib, "uppercase");
int rc = func("Hello");

dlclose(lib);
```

要点：

- `dlopen` 返回 `void *` 句柄；`dlsym` 用函数名取地址。
- 函数指针类型必须和库里的函数签名完全一致，否则是未定义行为。
- 每次 `dlsym`/`dlopen` 失败都要看 `dlerror()`。
- 编译主程序时可加 `-rdynamic`/`-ldl`（glibc 2.34 后 dl 已并入 libc，但写 `-ldl` 兼容性更好）。
- “C 的脆弱性”核心：越界、悬空指针、未定义行为、不安全的字符串函数。

---

## 三、类型、变量与格式化输出

### 练习 3：Formatted Printing

常用格式符：

| 格式 | 含义 |
|---|---|
| `%d` | `int` |
| `%u` | `unsigned int` |
| `%ld` / `%lu` | `long` / `unsigned long` |
| `%lld` | `long long` |
| `%zu` | `size_t` |
| `%f` | `double`（`float` 会自动提升为 `double`） |
| `%Lf` | `long double` |
| `%c` | `char` |
| `%s` | 字符串 |
| `%p` | 指针 |
| `%x` / `%X` | 十六进制 |
| `%o` | 八进制 |
| `%%` | 百分号 |

宽度与精度：

```c
printf("%10d\n", 42);      // 右对齐，占 10 格
printf("%-10s|\n", "abc"); // 左对齐
printf("%.2f\n", 3.14159); // 两位小数
printf("%08.2f\n", 3.5);   // 补零
```

要点：

- 格式符必须和参数类型匹配，否则是未定义行为。
- `printf` 不会检查参数个数/类型，编译器警告（`-Wformat`）要重视。

### 练习 6：Types

- 基本类型：`int`、`float`、`double`、`char`。
- 字符串是 `char *` 或 `char[]`，以 `'\0'` 结尾。
- `sizeof` 返回字节数，类型是 `size_t`，用 `%zu` 打印。
- 整数除法会截断：`1 / 3 == 0`。
- 算术运算会发生类型提升（`int` → `float` → `double`）。

### 练习 7：More Variables and Types

- `stdbool.h`：`bool`、`true`、`false`。
- `unsigned`：无符号，只有非负数；`%u` 打印。
- `char` 可能是 `signed` 或 `unsigned`，取决于实现。
- `sizeof(char) == 1`，但其他类型大小依平台而定。
- 未初始化变量内容是垃圾值，必须初始化后再使用。

### 练习 21：Advanced Data Types and Flow Control（理论，无代码）

- 类型：`int`、`double`、`float`、`char`、`void`、`enum`。
- 修饰符：`unsigned`、`signed`、`long`、`short`。
- 限定符：`const`、`volatile`、`register`。
- 类型转换阶梯：`long double` > `double` > `float` > `long long` > `long` > `int`（`short`、`char` 先转 `int`）。
- `stdint.h`：`int8_t`/`uint8_t`、`int16_t`、`int32_t`、`int64_t` 等定长类型。
- 极值宏：`INT_MAX`、`INT_MIN`、`UINT_MAX`、`INT32_MAX`、`INT64_MAX` 等。
- 指针相关：`intptr_t`、`uintptr_t`；`size_t`、`ptrdiff_t`。
- 运算符分类：算术、数据访问（`.`、`->`、`[]`、`*`、`&`、`sizeof`）、逻辑、位运算、布尔、赋值。
- 控制结构：`do-while`、`break`、`continue`、`goto`。

---

## 四、控制流

### 练习 8：If, Else-If, Else

- C 中 `0` 为假，非 `0` 为真。
- `if / else if / else` 只会执行一个分支。
- 比较运算符：`==`、`!=`、`<`、`<=`、`>`、`>=`。
- 布尔运算符：`&&`、`||`、`!`。
- 经典错误：把 `==` 写成 `=`。

### 练习 9：While-Loop and Boolean Expressions

```c
while(condition) {
    // 循环体
}
```

- `break`：立即跳出循环。
- `continue`：跳过本次循环剩余部分，进入下一次判断。
- 布尔表达式可以用 `&&`、`||`、`!` 组合。
- 循环变量必须在循环体内/条件中改变，否则可能死循环。
- 三元运算符：`cond ? a : b`。

### 练习 10：Switch Statements

```c
switch(ch) {
    case 'a':
        printf("a\n");
        break;
    default:
        printf("other\n");
}
```

- `switch` 只能用于整数/字符/枚举。
- 忘记 `break` 会继续执行下一个 `case`（fallthrough），有时是故意的，但多数是 bug。
- `default` 处理其他情况。

### 练习 13：For-Loops and Arrays of Strings

```c
for(i = 0; i < argc; i++) {
    printf("arg %d: %s\n", i, argv[i]);
}
```

- `for(初始化; 条件; 递增)`。
- `argv` 是 `char *argv[]`，即字符串数组。
- `argc` 是参数个数，`argv[0]` 是程序名。
- 数组下标从 0 开始，越界不会自动报错。

---

## 五、数组、字符串、函数与作用域

### 练习 11：Arrays and Strings

```c
char name[20] = "Zed";
printf("%s\n", name);
```

- 数组大小固定，声明后不能改变。
- 字符串末尾必须有 `'\0'`，`strlen` 不计它，`sizeof` 计它。
- `sizeof(array)` 是数组总字节数；作为参数传给函数后会退化为指针。
- C 不检查数组越界。
- `gets`、`strcpy` 等函数不安全，优先用 `fgets`、`strncpy` 或自己检查长度。

### 练习 12：Functions, Scope, and Globals

```c
int add(int a, int b);   // 声明

int add(int a, int b) {  // 定义
    return a + b;
}
```

- 函数声明（原型）告诉编译器参数和返回值类型。
- 局部变量只在函数/代码块内可见。
- 全局变量在整个文件（以及 `extern` 声明处）可见。
- 全局变量会被默认初始化为 0；局部变量不会。
- 函数参数按值传递；想修改调用者的变量要传指针。

### 综合：作用域与生命周期

- 栈上的局部变量在函数返回后失效，不能返回它的地址。
- 全局/静态变量生命周期贯穿整个程序。
- `static` 修饰局部变量时，只初始化一次，函数返回后仍保留值（本课程中了解即可）。

---

## 六、文件 I/O

### 练习 14：Writing and Reading Files

```c
FILE *f = fopen("ex14.txt", "w");
if(!f) {
    perror("fopen");
    return 1;
}
fprintf(f, "hello\n");
fclose(f);
```

模式：

- `"r"`：只读，文件必须存在。
- `"w"`：只写，截断/创建。
- `"a"`：追加。
- `"r+"`、`"w+"`、`"a+"`：读写模式。

读取：

```c
char line[256];
while(fgets(line, sizeof(line), f)) {
    printf("%s", line);
}
```

- `fgets` 会保留换行符，读到 EOF 返回 `NULL`。
- `fscanf` 返回值是成功匹配的项数，必须检查。
- `feof` 只有在尝试读越过文件末尾后才为真；不要用它来提前判断。
- 用完必须 `fclose`。

### 练习 24：Input, Output, Files

核心：安全地读入用户输入和文件内容。

- `fgets(buf, size, stdin)` 比 `scanf("%s")` 安全，能限制长度。
- `scanf` 会留下换行符，后续 `fgets` 可能读到空行；需要清掉。
- `fscanf(stdin, "%d", &age)` 返回 1 才表示读到了整数。
- 数字转换可用 `strtol`/`strtod`，并检查 `errno` 和结束指针。
- 读文件时要检查 `fopen` 是否为 `NULL`，读写后检查 `ferror`。
- 输入长度、类型、范围都要校验。

---

## 七、指针、结构体、堆与栈

### 练习 15：Pointers, Dreaded Pointers

```c
int x = 10;
int *p = &x;   // p 保存 x 的地址
printf("%d\n", *p);  // 解引用
*p = 20;       // 通过指针修改 x
```

要点：

- `&` 取地址，`*` 解引用。
- 指针变量本身也有大小：`sizeof(int *)` 通常是 8 字节（64 位）。
- 数组名在表达式中常退化为指向首元素的指针。
- 指针算术：`p + 1` 前进一个元素大小，不是 1 字节。
- `NULL` 表示不指向任何对象；解引用 `NULL` 会崩溃。
- 未初始化指针是野指针，不能解引用。

### 练习 16：Structs and Pointers to Them

```c
struct Person {
    char *name;
    int age;
};

struct Person *p = malloc(sizeof(struct Person));
p->name = strdup("Zed");
p->age = 37;
```

- `struct` 把多个字段组合成一个类型；`typedef` 可以简化名字。
- 值访问用 `.`，指针访问用 `->`。
- 结构体可以整体赋值（浅拷贝），指针字段只复制地址。
- 动态分配的结构体要 `free`，字段里的 `malloc`/`strdup` 也要单独 `free`。
- 用 `sizeof(*p)` 比 `sizeof(struct Person)` 更不容易写错。

### 练习 17：Heap and Stack Memory Allocation

```c
int *p = malloc(sizeof(int) * 10);
if(!p) { /* 处理内存不足 */ }
free(p);
p = NULL;
```

- 栈：函数局部变量，自动分配/释放，容量有限。
- 堆：`malloc`/`calloc`/`realloc` 手动申请，`free` 手动释放。
- `calloc(n, size)` 会清零；`malloc` 不清零。
- `realloc` 可能返回新地址，必须用临时指针接收。
- 常见错误：越界、重复 `free`、释放后继续使用、忘 `free`、释放栈地址。
- `free(NULL)` 是安全的；`free` 后建议把指针置 `NULL`。

---

## 八、函数指针与对象系统

### 练习 18：Pointers to Functions

```c
int (*func)(int);              // 函数指针
typedef int (*compare_cb)(int, int);

void sort(int *a, int n, compare_cb cmp) {
    // 调用 cmp(a, b)
}
```

- 函数名本身可以取地址；`func = add;` 和 `func = &add;` 等价。
- 函数指针用于回调、策略模式、排序比较函数。
- 数组元素也可以是函数指针。
- `qsort` 的比较函数就是函数指针。
- 声明时注意括号：`int *f(int)` 是返回指针的函数；`int (*f)(int)` 才是函数指针。

### 练习 19：A Simple Object System

核心思想：用结构体 + 函数指针模拟面向对象。

```c
typedef struct {
    char *description;
    int (*init)(void *self);
    void (*describe)(void *self);
    void (*destroy)(void *self);
    void *(*move)(void *self, Direction direction);
    int (*attack)(void *self, int damage);
} Object;

#define NEW(T, N) Object_new(sizeof(T), T##Proto, N)
#define _(N) proto.N
```

要点：

- `Object` 是“基类”，其他结构体把 `Object proto` 放在第一个成员。
- 因为第一个成员地址和结构体地址相同，可以把子类指针强转为 `Object *`/`void *`。
- 方法统一接收 `void *self`，在函数内部再转回具体类型。
- `T##Proto` 是预处理器的 token paste（`Monster` + `Proto` → `MonsterProto`）。
- `NEW` 分配 `sizeof(T)` 字节，把原型复制进去，再调用 `init`。
- 默认方法由 `Object_new` 填充，子类只覆盖自己关心的。
- `Object_destroy` 负责释放 `description` 和对象本身。
- `strdup` 会为字符串分配内存，记得释放。

---

## 九、调试：dbg.h、valgrind、gdb

### 练习 20 / 22：dbg.h 调试宏

`dbg.h` 常用宏：

```c
debug("value=%d", x);           // 仅调试版输出
log_err("failed: %s", msg);     // 错误日志
log_warn("warning");            // 警告
log_info("info");               // 信息

check(x != NULL, "x is NULL");  // 条件失败则跳到 error
check_mem(p);                   // 检查 malloc 结果
sentinel("invalid format");     // 无条件报错并跳转
```

宏背后的关键点：

```c
#define debug(M, ...) fprintf(stderr, "DEBUG %s:%d: " M "\n", \
                              __FILE__, __LINE__, ##__VA_ARGS__)
#define check(A, M, ...) if(!(A)) { \
    log_err(M, ##__VA_ARGS__); errno=0; goto error; }
```

- `__FILE__`、`__LINE__`、`__func__` 是编译器提供的调试信息。
- `##__VA_ARGS__` 是 GCC 扩展，处理可变参数为空的情况。
- `goto error` 是 C 中常见的集中错误处理方式。
- 定义 `NDEBUG` 后，`debug` 等调试日志会被关闭；`check` 通常仍保留。
- 使用宏时要小心副作用，例如 `check(i++ > 0, ...)` 可能让 `i` 自增多次。
- `errno` 在错误处理前应清零或保存，避免被日志函数覆盖。

### 练习 31：Debugging Code

调试手段的选择：

- **debug 输出**：能看到变量随时间的完整轨迹，适合逻辑错误。
- **valgrind**：内存错误、泄漏、未初始化值。
- **gdb**：断点、单步、查看/修改变量、调用栈，适合“程序卡住/崩溃”。

gdb 常用命令：

| 命令 | 作用 |
|---|---|
| `gdb -q ./prog` | 启动调试 |
| `run` | 运行 |
| `break file:line` / `break func` | 下断点 |
| `next` | 下一行（不进入函数） |
| `step` | 进入函数 |
| `continue` | 继续运行 |
| `print var` | 查看变量 |
| `set var i = 200` | 修改变量 |
| `bt` | 查看调用栈 |
| `frame N` | 切换栈帧 |
| `info locals` | 查看局部变量 |
| `watch var` | 监视变量变化 |
| `list` | 查看源码 |
| `attach PID` / `detach` | 附加/脱离运行中的进程 |
| `quit` | 退出 |

练习 31 的示例程序：

```c
int i = 0;
while(i < 100) {
    usleep(3000);
}
```

- 问题：循环体没有 `i++`，`i` 永远为 0，程序不会退出。
- 验证：`break` 到循环行，`print i` 得到 0。
- 修复路径：`set var i = 200` 后 `continue`，程序正常退出。
- 真实修复当然是在源码里补 `i++`。
- 附加题：让程序崩溃并生成 core dump，再用 `gdb ./prog core` 分析。

---

## 十、变参函数

### 练习 25：Variable Argument Functions

```c
#include <stdarg.h>

int sum(int count, ...) {
    va_list argp;
    va_start(argp, count);

    int total = 0;
    for(int i = 0; i < count; i++) {
        total += va_arg(argp, int);
    }

    va_end(argp);
    return total;
}
```

要点：

- `...` 必须是参数列表最后一个。
- `va_start` 的第二个参数是最后一个固定参数。
- `va_arg(argp, TYPE)` 的 `TYPE` 必须和实际传入类型一致（默认提升后）。
- `char`/`short` 会提升为 `int`，`float` 会提升为 `double`。
- 类型不匹配是未定义行为，编译器不会检查。
- 必须成对调用 `va_start` / `va_end`。
- 变参函数无法自动知道参数个数，需要格式串或数量参数。

练习中的 `read_scan`：

- 解析格式串 `%d`、`%c`、`%s`。
- `%s` 需要两个参数：最大长度和 `char **` 输出指针。
- `read_string` 用 `calloc` 分配缓冲区，用 `fgets` 读取。
- 出错时统一 `goto error`，释放已分配内存。
- 附加题 `my_printf`：用 `va_list` 自己实现 `%d/%c/%s/%%`。

---

## 十一、自动化测试

### 练习 30：Automated Testing（minunit）

`minunit.h` 四个核心宏：

```c
#define mu_suite_start() char *message = NULL
#define mu_assert(test, message) if (!(test)) { log_err(message); return message; }
#define mu_run_test(test) debug("\n-----%s", " " #test); \
    message = test(); tests_run++; if (message) return message;
#define RUN_TESTS(name) int main(int argc, char *argv[]) { ... }
```

测试文件结构：

```c
#include "minunit.h"

char *test_dlopen() { return NULL; }   // 成功返回 NULL

char *all_tests() {
    mu_suite_start();
    mu_run_test(test_dlopen);
    return NULL;
}

RUN_TESTS(all_tests);
```

要点：

- 测试函数无参数，返回 `char *`；成功返回 `NULL`，失败返回错误消息。
- `RUN_TESTS` 生成 `main`，统一处理退出码和测试计数。
- 测试文件名约定 `tests/*_tests.c`，Makefile 用通配符收集。
- `runtests.sh` 逐个运行测试二进制，失败时打印日志并 `exit 1`。
- 可用 `VALGRIND="valgrind ..." make valgrind` 让测试在 valgrind 下跑。
- 测试要覆盖：正常情况、边界（空链表/已排序）、失败路径。
- `minunit.h` 开头 `#undef NDEBUG`，保证测试里 assert 不被关闭。

---

## 十二、链表与排序

### 练习 32：Double Linked Lists

节点与链表：

```c
typedef struct ListNode {
    struct ListNode *next;
    struct ListNode *prev;
    void *value;
} ListNode;

typedef struct List {
    int count;
    ListNode *first;
    ListNode *last;
} List;
```

常用 API：

| 函数/宏 | 作用 | 复杂度 |
|---|---|---|
| `List_create()` | 创建空链表 | O(1) |
| `List_destroy(list)` | 释放链表和节点 | O(n) |
| `List_clear(list)` | 释放所有 value | O(n) |
| `List_clear_destroy(list)` | 两者都做 | O(n) |
| `List_push(list, v)` | 尾部插入 | O(1) |
| `List_pop(list)` | 尾部删除 | O(1) |
| `List_unshift(list, v)` | 头部插入 | O(1) |
| `List_shift(list)` | 头部删除 | O(1) |
| `List_remove(list, node)` | 删除指定节点 | O(1) |
| `List_count(list)` | 节点数量 | O(1) |
| `List_first(list)` / `List_last(list)` | 首/尾值 | O(1) |
| `LIST_FOREACH(list, first, next, cur)` | 遍历 | O(n) |

要点：

- 双向链表每个节点有 `prev` 和 `next`。
- 插入/删除时要维护 `first`、`last`、`count` 和相邻节点的指针。
- 谁分配谁释放：链表只负责节点，不自动释放 `value`（除非 `List_clear`）。
- 遍历时删除节点要用安全遍历方式，避免访问已释放节点。
- 单元测试要覆盖空链表、单节点、多节点、头尾操作。

### 练习 33：Linked List Algorithms

比较函数类型：

```c
typedef int (*List_compare)(const void *a, const void *b);
```

#### 冒泡排序（链表版）

思路：

1. 重复遍历链表。
2. 比较相邻两个节点的 value。
3. 如果前一个大于后一个，交换 value。
4. 如果一整轮没有交换，说明已有序，结束。

复杂度：时间 O(n²)，空间 O(1)。

要点：

- 交换的是 `value`，不是节点，避免重连指针。
- 空链表或单节点直接返回。
- 已排序链表只需一轮即可结束。

#### 归并排序（链表版）

思路：

1. 找中点，把链表分成左右两半。
2. 递归排序左右两半。
3. 合并两个有序链表：每次取较小的头节点。

复杂度：时间 O(n log n)，额外空间 O(n)（本实现会创建新链表）。

要点：

- 递归终止条件：节点数 ≤ 1。
- 合并时用 `List_shift` 取出头部，`List_push` 放入结果。
- 需要正确处理某一半为空的情况。
- 附加题 `List_insert_sorted`：遍历找到第一个比新值大的节点，插入到它前面。

---

## 十三、防御性编程

### 练习 27：Creative and Defensive Programming（理论，无代码要求）

核心观念：

- 防御性编程假设“自己一定会犯错”，在每一步尽量预防。
- 创造性阶段可以大胆写原型；防御性阶段要严谨、可测试、可维护。
- 不要相信输入：长度、类型、范围、NULL 都要检查。
- 使用 `assert` 表达“这里必须成立”的前置/后置条件。
- 用长度参数控制复制，不依赖字符串的 `'\0'`。
- 避免 `strcpy`、`gets` 等不检查边界的函数。
- 每个返回值都要检查（`malloc`、`fopen`、`read`、`fscanf`）。
- 边界测试：空、0、负数、最大值、超长输入。
- `assert` 在定义 `NDEBUG` 后会被移除，不能把必须执行的逻辑放进 `assert`。

练习中的对比：

```c
// 不安全：依赖 from 以 '\0' 结尾
void copy(char to[], char from[]) {
    int i = 0;
    while((to[i] = from[i]) != '\0') {
        ++i;
    }
}

// 更安全：显式传入长度，并保证目标字符串终止
int safercopy(int from_len, char *from, int to_len, char *to) {
    assert(from != NULL && to != NULL && "from and to can't be NULL");
    if(from_len < 0 || to_len <= 0) return -1;

    int max = from_len > to_len - 1 ? to_len - 1 : from_len;
    for(int i = 0; i < max; i++) to[i] = from[i];

    to[to_len - 1] = '\0';
    return max;
}
```

---

## 十四、高频易错点清单

1. **`=` 与 `==` 混淆**：`if(x = 1)` 永远为真。
2. **`switch` 忘记 `break`**：发生 fallthrough。
3. **字符串没有 `'\0'`**：`printf("%s")`、`strlen` 会越界。
4. **`sizeof(pointer)` 不是数组长度**：数组传参后退化为指针。
5. **数组/字符串越界**：C 不检查，可能不崩溃但会破坏内存。
6. **整数除法截断**：`1 / 2 == 0`，要得到小数需 `1.0 / 2`。
7. **`printf` 格式符与参数不匹配**：未定义行为。
8. **`malloc` 不检查返回值**：内存不足时得到 `NULL`。
9. **`realloc` 直接覆盖原指针**：失败时原内存泄漏。
10. **重复 `free` / 释放后使用 / 释放栈地址**：崩溃或未定义行为。
11. **返回局部变量地址**：函数返回后栈内存失效。
12. **`fgets` 保留换行符**：比较字符串时容易多一个 `\n`。
13. **`scanf` 留下换行符**：后续 `fgets`/`getchar` 读到空行。
14. **`feof` 用法错误**：应在读失败后判断，不要提前判断。
15. **`va_arg` 类型不匹配**：变参函数不会做类型检查。
16. **`NDEBUG` 关闭 `assert`**：发布版中断言消失，不能依赖它做业务逻辑。
17. **有符号与无符号混用**：负数转 `unsigned` 会变成很大的数。
18. **宏参数副作用**：`check(i++ > 0, ...)` 可能多次自增。
19. **指针未初始化**：野指针解引用。
20. **忘记维护链表 `count`/`first`/`last`**：遍历或计数出错。

---

## 十五、常用命令速查

### 编译

```bash
gcc -Wall -Wextra -g -Isrc prog.c -o prog
gcc -Wall -g -c file.c -o file.o
gcc -shared -fPIC -o libfoo.so foo.o
ar rcs libfoo.a foo.o
```

### Make

```bash
make                 # 默认第一个目标
make all
make clean
make tests
make valgrind
make install PREFIX=/usr/local
```

### Valgrind

```bash
valgrind ./prog
valgrind --leak-check=full --show-leak-kinds=all ./prog
valgrind --track-origins=yes ./prog
```

### GDB

```bash
gdb -q ./prog
gdb -q --batch -ex 'break main' -ex run -ex bt ./prog
gdb -q ./prog core
```

### 动态库排查

```bash
ldd ./prog
nm -D build/libfoo.so | grep func
```

---

## 十六、复习自测题

### 基础

- [ ] `-Wall` 和 `-g` 分别做什么？
- [ ] Makefile 命令行为什么必须用 TAB？
- [ ] `%d`、`%u`、`%zu`、`%f`、`%s` 分别对应什么类型？
- [ ] `sizeof(array)` 和 `sizeof(pointer)` 有什么区别？
- [ ] 为什么字符串必须以 `'\0'` 结尾？
- [ ] `while` 循环中 `break` 和 `continue` 的区别？

### 函数与内存

- [ ] 函数参数按值传递，如何修改调用者的变量？
- [ ] `malloc`、`calloc`、`realloc` 的区别？
- [ ] 什么是悬空指针、野指针、内存泄漏？
- [ ] 为什么不能返回局部数组的地址？
- [ ] `struct` 的 `.` 和 `->` 分别在什么时候用？

### 指针与对象

- [ ] 函数指针的声明语法是什么？
- [ ] `qsort` 的比较函数为什么需要函数指针？
- [ ] 对象系统中为什么可以把子类指针转成 `Object *`？
- [ ] `#define _(N) proto.N` 的作用是什么？
- [ ] `NEW(T, N)` 里的 `T##Proto` 如何展开？

### 调试与测试

- [ ] `check`、`check_mem`、`sentinel` 分别做什么？
- [ ] `goto error` 的错误处理模式有什么优点？
- [ ] `NDEBUG` 对 `debug` 和 `assert` 分别有什么影响？
- [ ] gdb 的 `next` 和 `step` 有什么区别？
- [ ] minunit 的 `mu_assert` 失败时如何传递错误信息？
- [ ] `runtests.sh` 为什么要检查退出码？

### 变参、构建与数据结构

- [ ] `va_start`、`va_arg`、`va_end` 的使用顺序？
- [ ] 为什么 `va_arg` 的类型必须准确？
- [ ] 静态库和共享库的构建命令分别是什么？
- [ ] `dlopen`/`dlsym`/`dlclose` 的作用？
- [ ] 双向链表 push/pop/shift/unshift 的时间复杂度？
- [ ] 链表冒泡排序和归并排序的时间复杂度？
- [ ] 归并排序合并两个有序链表的步骤？
- [ ] 防御性编程的核心原则是什么？

---

## 附：对应练习文件索引

```text
ex1/ ~ ex20/    单文件练习（每个目录含 .c 和同名可执行文件）
ex22/           调试宏
ex24/           输入输出与文件
ex25/           变参函数
ex31/           调试练习
ex32/           双向链表
ex33/           链表排序算法
c-skeleton/     ex28/ex29/ex30（Makefile 工程、共享库、自动化测试）
```

> 练习 21（高级数据类型）和 27（防御性编程）是理论章节，复习时重点看概念和易错点即可。
