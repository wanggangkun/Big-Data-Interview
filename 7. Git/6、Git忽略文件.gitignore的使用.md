##gitignore

.gitignore 一般会放到项目的根目录，他的作用是什么呢？ git提交时要忽略的文件 比如：node_modules


规则很简单，不做过多解释，但是有时候在项目开发过程中，突然心血来潮想把某些目录或文件加入忽略规则，按照上述方法定义后发现并未生效，原因是.gitignore只能忽略那些原来没有被track的文件，
如果某些文件已经被纳入了版本管理中，则修改.gitignore是无效的。那么解决方法就是先把本地缓存删除（改变成未track状态），然后再提交：

>git rm -r --cached .

>git add .

>git commit -m 'update .gitignore'

注意：

不要误解了 .gitignore 文件的用途，该文件只能作用于 Untracked Files，也就是那些从来没有被 Git 记录过的文件（自添加以后，从未 add 及 commit 过的文件）。

如果文件曾经被 Git 记录过，那么.gitignore 就对它们完全无效。

###规则
下面就是对这些规则的一些简单说明

```java

prj				忽略所有名字是prj的文件和文件夹，不管其目录的相对位置在哪。
/prj			忽略根目录下的名字是prj的文件和文件夹
prj/			后面添加/，是指文件夹，不包括文件。位置没有写，忽略所有位置的名为prj的文件夹。
#               表示此为注释,将被Git忽略
*.a             表示忽略所有 .a 结尾的文件
!lib.a          表示但lib.a除外
/TODO           表示仅仅忽略项目根目录下的 TODO 文件，不包括 subdir/TODO
build/          表示忽略 build/目录下的所有文件，过滤整个build文件夹；
doc/*.txt       表示会忽略doc/notes.txt但不包括 doc/server/arch.txt

bin/:           表示忽略当前路径下的bin文件夹，该文件夹下的所有内容都会被忽略，不忽略 bin 文件
/bin:           表示忽略根目录下的bin文件
/*.c:           表示忽略cat.c，不忽略 build/cat.c
debug/*.obj:    表示忽略debug/io.obj，不忽略 debug/common/io.obj和tools/debug/io.obj
**/foo:         表示忽略/foo,a/foo,a/b/foo等
a/**/b:         表示忽略a/b, a/x/b,a/x/y/b等
!/bin/run.sh    表示不忽略bin目录下的run.sh文件
*.log:          表示忽略所有 .log 文件
config.php:     表示忽略当前路径的 config.php 文件

/mtk/           表示过滤整个文件夹
*.zip           表示过滤所有.zip文件
/mtk/do.c       表示过滤某个具体文件

被过滤掉的文件就不会出现在git仓库中（gitlab或github）了，当然本地库中还有，只是push的时候不会上传。

需要注意的是，gitignore还可以指定要将哪些文件添加到版本管理中，如下：
!*.zip
!/mtk/one.txt

唯一的区别就是规则开头多了一个感叹号，Git会将满足这类规则的文件添加到版本管理中。为什么要有两种规则呢？
想象一个场景：假如我们只需要管理/mtk/目录中的one.txt文件，这个目录中的其他文件都不需要管理，那么.gitignore规则应写为：：
/mtk/*
!/mtk/one.txt

假设我们只有过滤规则，而没有添加规则，那么我们就需要把/mtk/目录下除了one.txt以外的所有文件都写出来！
注意上面的/mtk/*不能写为/mtk/，否则父目录被前面的规则排除掉了，one.txt文件虽然加了!过滤规则，也不会生效！

```


本来是想把相对路径符合xx/prj/的文件夹都忽略的，但没有用。

说明只要你在规则里写了路径，就是绝对路径，没有相对路径的

xx/prj/和prj/*是一样的道理，都表明了路径，开头加不加/，都表示“绝对路径”。

结论：

1.    开头的/并不是标识文件夹的，要表明仅忽略文件夹需要在名称后面添加/，而不是前面。

2.    要想忽略某文件夹，但其下部分文件不能忽略。则需要添加通配符*，然后在后面添加！开头的规则，来指出不忽略的文件或文件夹。

3.    只要写了路径，即/左右两边都有字符，那么就是指的“绝对路径”(相对仓库的，仓库.git文件夹所在目录为根目录)，但可以用*来指定层级，指定第几层子目录下的某个文件夹。

4.    不忽略的规则只要写文件名或文件夹名(名称中可以加通配符)前面加！就可以了。会在上面有通配符忽略的列表里找到匹配项来不忽略(反忽略)它。
5.    
6.     补充：刚才试验，发现：.gitignore不仅可以写在.git所有文件夹，还可以写在子目录里。ignore文件里写的路径全是相对该ignore文件所在位置。这样一个复杂工程，就可以分模块写忽略文件了。这个挺好的。

补充2: 刚才在网上发现有大神提出 ** 表示任意层级目录，我就试了一下


待验证：[](https://blog.csdn.net/o07sai/article/details/81043474?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-4.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-4.nonecase)