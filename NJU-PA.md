# NJU-PA
## 如何测试你的代码



你将来是要使用你自己实现的表达式求值功能来帮助你来进行后续的调试的, 这意味着程序设计课上那种"代码随便测试一下就交上去然后就可以撒手不管"的日子已经一去不复返了. 测试需要测试用例, 通过越多测试, 你就会对代码越有信心. 但如果让你来设计测试用例, 设计十几个你就会觉得没意思了, 有没有一种方法来自动产生测试用例呢?
一种常用的方法是随机测试. 首先我们需要来思考如何随机生成一个合法的表达式. 事实上, 表达式生成比表达式求值要容易得多. 同样是上面的BNF, 我们可以很容易写出生成表达式的框架:
```c
void gen_rand_expr() {
  switch (choose(3)) {
    case 0: gen_num(); break;
    case 1: gen('('); gen_rand_expr(); gen(')'); break;
    default: gen_rand_expr(); gen_rand_op(); gen_rand_expr(); break;
  }
}
```
如果可以在生成这些表达式的同时, 也能生成它们的结果
但我们在NEMU中实现的表达式求值是经过了一些简化的, 所以我们需要一种满足以下条件的"计算器":
- 进行的都是无符号运算
- 数据宽度都是32bit
- 溢出后不处理

  
不过实现的时候, 你很快就会发现需要面对一些细节的问题:
- 如何保证表达式进行无符号运算?
- 如何随机插入空格?
- 如何生成长表达式, 同时不会使buf溢出?
- 如何过滤求值过程中有除0行为的表达式?



保证表达式无符号运算符:
sprintf("%uu")

如何随机插入空格?
空格列表 “” “ ” “  ”

如何生成长表达式, 同时不会使buf溢出？
长度超过一半就只生成数字

过滤求值过程中的除0：
gcc -Werror 有除0 gcc 会warning 直接过滤 偷鸡..........



```C
#include <assert.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

// this should be enough
static char buf[65536] = {}; // save expression
// a little larger than buf
static char code_buf[65536 + 128] = {}; // save the whole code
static char *code_format = "#include <stdio.h>\n"
                           "int main() { "
                           "  unsigned result = %s; "
                           "  printf(\"%%u\", result); "
                           "  return 0; "
                           "}";

static char *pbuf; // pointer to not used buf
#define format_buf(fmt, ...) pbuf += sprintf(pbuf, fmt, ##__VA_ARGS__)

static inline uint32_t choose(uint32_t max) { return rand() % max; }

static inline void gen_rand_op() {
  char op_list[] = {'+', '-', '*', '/', '+', '-', '*'};
  format_buf("%c", op_list[choose(7)]);
}

static inline void gen_num() { format_buf("%uu", rand()); }

static inline void gen_space() {
  char *space_list[3] = {
      "",
      " ",
      "  ",
  };
  format_buf("%s", space_list[choose(3)]);
}

static int nr_op = 0;

static void gen_rand_expr() {
  gen_space();
  switch (choose(3)) {
  default:
    if (nr_op == 0)
      gen_rand_expr();
    else
      gen_num();
    break;
  case 1:
    format_buf("(");
    gen_rand_expr();
    format_buf(")");
    break;
  case 0:
    nr_op++;
    if (pbuf - buf >= sizeof(buf) / 2) {
      // length > half
      gen_num();
      break;
    }
    gen_rand_expr();
    gen_space();
    gen_rand_op();
    gen_space();
    gen_rand_expr();
    break;
  }
  gen_space();
}

void remove_u(char *p) {
  char *q = p;
  while ((q = strchr(q, 'u')) != NULL) {
    strcpy(code_buf, q + 1);
    strcpy(q, code_buf);
  }
}

int main(int argc, char *argv[]) {
  int seed = time(0);
  srand(seed);
  int loop = 1;
  if (argc > 1) {
    sscanf(argv[1], "%d", &loop);
  }
  int i;
  for (i = 0; i < loop; i++) {
    nr_op = 0;
    pbuf = buf;
    gen_rand_expr();

    sprintf(code_buf, code_format, buf);

    FILE *fp = fopen("/tmp/.code.c", "w");
    assert(fp != NULL);
    fputs(code_buf, fp);
    fclose(fp);

    int ret = system("gcc -Werror /tmp/.code.c -o /tmp/.expr");
    if (ret != 0)
      continue;

    fp = popen("/tmp/.expr", "r");
    assert(fp != NULL);

    int result;
    fscanf(fp, "%d", &result);
    ret = pclose(fp);
    if (ret != 0)
      continue;
    remove_u(buf);
    printf("%u %s\n", result, buf);
  }
  return 0;
}

```


