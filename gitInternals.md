# learn

```bash
find .git/objects
```

输出结果：

```text
.git/objects
.git/objects/pack
.git/objects/info
```

```bash
find .git/objects -type f
```

输出结果为空

Git 对 objects 目录进行了初始化，并创建了 pack 和 info 子目录，但均为空。
