---

### 一、 文件与输入输出（I/O）
*   **`fopen` / `fclose`**
    *   **功能**：打开/关闭文件。
    *   **格式**：`FILE *fp = fopen("test.txt", "r"); fclose(fp);`
*   **`fgetc` / `fputc`**
    *   **功能**：从文件流中读/写 **1 个字符**。
    *   **格式**：`int ch = fgetc(fp); fputc('A', fp);`
*   **`fgets` / `fputs`**
    *   **功能**：从文件流中读/写 **1 行字符串**（`fgets` 保留 `\n`，适合文本处理）。
    *   **格式**：`fgets(buf, sizeof(buf), fp); fputs(buf, fp);`
*   **`fread` / `fwrite`**
    *   **功能**：按二进制块读/写 **任意数据**（适合复制图片、音频等，不关心 `\0` 和 `\n`）。
    *   **格式**：`fread(buffer, 1, 4096, fp); fwrite(buffer, 1, bytes_read, fp);`
*   **`fseek` / `ftell` / `rewind`**
    *   **功能**：移动文件光标、获取当前位置、将光标重置到文件开头。
    *   **格式**：`fseek(fp, 0, SEEK_END); long size = ftell(fp); rewind(fp);`
*   **`fflush`**
    *   **功能**：**强制刷新输出缓冲区**（将内存缓冲数据立刻写入磁盘或屏幕，防断电丢数据）。
    *   **格式**：`fflush(fp);`

### 二、 字符串与内存处理
*   **`strncmp`**
    *   **功能**：比较两个字符串的**前 n 个字符**，常用于判断字符串是否以特定前缀开头。
    *   **格式**：`if (strncmp(line, "#include", 8) == 0)`
*   **`sscanf`**
    *   **功能**：从**内存中的字符串**格式化提取数据。
    *   **格式**：`sscanf(line, "%d", &num);`
*   **`strtok_r`**
    *   **功能**：**线程安全**的字符串分割工具，依赖外部传入的 `saveptr` 记录断点。
    *   **格式**：`strtok_r(str, delim, &saveptr);`
*   **`strtol` / `strtod`**
    *   **功能**：将字符串转为长整型/双精度浮点数，**能精准判断转换失败、溢出，以及数字末尾剩余字符**。
    *   **格式**：`strtol(str, &endptr, 10);`
*   **`memset`**
    *   **功能**：**按字节**快速填充一块内存（最常用于**初始化为 0**）。
    *   **格式**：`memset(ptr, 0, size);` *(警告：绝不能给 int 数组赋初值 1)*
*   **`memcpy` / `memmove`**
    *   **功能**：内存块拷贝。**必须用 `memmove` 处理源和目标内存重叠的情况**。
    *   **格式**：`memcpy(dst, src, n); memmove(dst, src, n);`

### 三、 动态内存管理
*   **`malloc` / `free`**
    *   **功能**：在堆区动态分配内存 / 手动释放。
    *   **格式**：`int *p = malloc(10 * sizeof(int)); free(p);`
*   **`calloc`**
    *   **功能**：分配内存并**立即将内存全部清零**。
    *   **格式**：`int *p = calloc(10, sizeof(int));`
*   **`realloc`**
    *   **功能**：调整已有内存块的大小（扩容/缩容），会自动搬运数据。**必须用中间指针接收防止内存泄露**。
    *   **格式**：`void *new_p = realloc(p, new_size); if (new_p) p = new_p;`
*   **`alloca`**
    *   **功能**：在**函数栈**上快速分配内存（随函数结束自动释放，**无需 `free`，但容易爆栈**）。
    *   **格式**：`void *p = alloca(size);`

### 四、 预处理指令
*   **`#include`**：包含头文件（`<>`系统目录，`""`当前目录）。
*   **`#define` / `#` / `##`**：宏定义/字符串化/拼接。
*   **`#ifdef` / `#ifndef` / `#endif`**：条件编译，防止头文件重复包含。格式：`#ifndef HEADER_H ... #endif`
*   **可变参数宏**：`__VA_ARGS__`。格式：`#define DEBUG(...) printf(__VA_ARGS__)`

### 五、 错误处理与系统接口
*   **`errno`**：记录最后一次系统调用的错误码。
*   **`perror`**：将当前 `errno` 翻译成英文错误信息并输出到 `stderr`。
*   **`ferror`**：检查文件流是否发生错误。
*   **时间处理**：`time_t t = time(NULL); struct tm *tm = localtime(&t); strftime(buf, size, "%Y-%m-%d", tm);`
*   **`sleep`**：挂起程序指定秒数。格式：`sleep(1);` (Windows 为 `Sleep(1000);`)

### 六、 可变参数与入口
*   **可变参数函数**：`va_list`、`va_start`、`va_arg`、`va_end`。用于接收不定长参数（如 `printf`）。
    *   **格式**：`va_start(ap, last_arg); int val = va_arg(ap, int); va_end(ap);`
*   **`main` 函数参数**：`int main(int argc, char *argv[])`
    *   **功能**：从命令行获取参数。`argc` 是参数个数，`argv[]` 是参数字符串数组。
*   **`size_t` 类型**：专门用于**表示内存大小和数组长度**的无符号自适应类型（32位为4字节，64位为8字节）。
*   **`typedef`**：为复杂数据类型起别名。格式：`typedef unsigned long size_t;`