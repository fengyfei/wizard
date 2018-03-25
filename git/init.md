# Init

## git init

```git init``` 用于创建一个新库。由于大多数 git 命令，需要操作 .git 目录下内容，因此，我们需要了解 git 下内容如何存储。

根据 git 命令执行，我们知道，需要找到 cmd_init_db。

```C
{ "init", cmd_init_db },
```

通过简单的 grep 操作，可定位 cmd_init_db 位置：

```sh
$ grep -r -n "cmd_init_db" .
./builtin.h:175:extern int cmd_init_db(int argc, const char **argv, const char *prefix);
./builtin/init-db.c:468:int cmd_init_db(int argc, const char **argv, const char *prefix)
./git.c:415:	{ "init", cmd_init_db },
./git.c:416:	{ "init-db", cmd_init_db },
```

cmd_init_db 基本流程如下图：

![Init DB Flow Chart](./images/cmd_init_db_flow.png)

## reposity

先看关系图:

![Reposity Data Structure](./images/reposity.png)

## References

- [Git Config](https://git-scm.com/docs/git-config)
