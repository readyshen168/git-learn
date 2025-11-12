# learn

```shell
# 查看objects目录
find .git/objects
```

输出结果：

```text
.git/objects
.git/objects/pack
.git/objects/info
```

```shell
find .git/objects -type f
```

输出结果为空

Git 对 objects 目录进行了初始化，并创建了 pack 和 info 子目录，但均为空。

## git hash-object & git update-index

### git add 时，其内部调用了以上两个程序

1. git hash-object 在.git/objects 中写入对应文件的 SHA-1 文件

```shell
echo 'test content2' > test2.txt
git add test.txt
find .git/objects -type f
# 出现test.txt对应的SHA-1文件，现删除之
rm -r .git/objects/4c
git commit -m 'test2.txt'
# 此时出现错误，表示找不到对应的4c...SHA-1对象
# 可以手动调用git hash-object来补上这个对象
cat test2.txt | git hash-object -w --stdin
find .git/objects -type f
# 此时出现了test.txt对应的SHA-1文件，再次commit便可成功
```

'test2.txt' commit 863525 所做的事情:

```shell
echo 'test content' | git hash-object -w --stdin
# SHA-1: d670

echo 'version 1' > test.txt
git hash-object -w test.txt
# SHA-1: 83baa

echo 'test content2' > test2.txt
git add test2.txt
# SHA-1:4c4c
rm -r .git/objects/4c
cat test2.txt | git hash-object -w --stdin
# SHA-1:4c4c

git commit -m 'test2.txt'
# SHA-1: a985
# SHA-1: 8635

git cat-file -t 8635 # commit
git cat-file -p 8635 # tree a985

git cat-file -t a985 # tree
git cat-file -p a985 # 100644 blob 4c4c

```

2. git update-index 使用第 1 步中产生的 blob 数据对象来创建树对象

```shell
git update-index --add --cacheinfo 100644 83baae61804e65cc73a7201a7252750c76066a30 test.txt
git status
# 此时绿色的内容是 new file: test.txt
# 红色的内容是 modified: test.txt
# 原因是在此之前执行了： git cat-file -p 1f7a > test.txt
# 所以工作区的text.txt内容是'version 2'
# 而暂存区拉取的是83ba，内容是'version 1'

git write-tree # SHA-1: e82e6
git status # 和上次git status一样

git cat-file -p e82e6
# 100644 blob 83baae61804e65cc73a7201a7252750c76066a30    test.txt
# 100644 blob 4c4c5cdd3f7e76632060ade0602df7ed76deaea7    test2.txt
# ???为什么test2.txt也被写入了树e82e6，需要复盘来再看看
```

## git write-tree & git update-index 多个文件组成目录，生成树对象

```shell
echo 'new file' > new.txt
cat new.txt | git hash-object --stdin # 验证一下：SHA-1 fa49
git update-index --add new.txt
git status
# 绿色的有 new file: new.txt     new file: text.txt
# 红色的有 modified: test.txt

git update-index --cacheinfo 100644 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a test.txt
# 此时git status中的红色消失，因为暂存区拉取的blob对象test.txt内容和工作区的一致，都是'version 2'

git write-tree # 5e0b

git cat-file -p 5e0b
# 100644 blob fa49b077972391ad58037050f2a75f74e3671e92    new.txt
# 100644 blob 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a    test.txt
# 100644 blob 4c4c5cdd3f7e76632060ade0602df7ed76deaea7    test2.txt
# ??? test2.txt依然存在，应该如何去除呢？

```

'new.txt&test.txt' commit 5f1e 所做的事情：

把 new.txt('new file') 和 test.txt('version 2')写入暂存区，生成树对象，进行了提交

## git commit 调用了 git write-tree，把暂存区的 blob 对象写成树对象，然后生成 commit 对象
