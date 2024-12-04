# IMPORT AGAIN

## 总结

- 推荐使用`module`(带`__init__.py`的目录), 同时支持相对导入和绝对导入
- 相对导入：`python main.py`的`main.py`不能出现在(或者说要高于)相对导入涵盖的范围且不含相对导入
- 绝对导入：统一导入的立足点（一般为根目录），执行时强调导入立足点`export PYTHONPATH=rootdir/ && python main.py`
- 推荐相对导入和绝对导入的混合使用,不是说在一个文件内混合使用，而是项目内两者都有，灵活使用

## 分析

- 示例项目结构

    ```zsh
    pojo
    └── importer
        ├── README.md
        ├── importer
        │   ├── __init__.py
        │   ├── func1.py
        │   ├── func2.py
        │   └── tool
        │       ├── __init__.py
        │       └── tool.py
        ├── importer.ipynb
        ├── main.py
        └── tests
            └── test_tool.py
    ```

- 使用相对导入法

    形如 `from ..x import foo | from .y import bar`

    须保证执行函数(`python main.py`的`main.py`)的位置要高于所有相对导入涵盖的范围, 并且`main.py`内不能含任何相对导入,因为`main.py`是程序入口，程序入口就是Python发现机制的最高级的目录

    假设现在要在 `test_tool.py` 中测试`tool.py::a_tool`函数，使用相对导入法，即：

    ```python
    from ..importer.tool.tool import a_tool
    def test_tool():
        a_tool()

    ```

    我们现在位于目录`pojo/importer`(下面用根目录代替),开始尝试执行 `test_tool.py`

    第一想法就是

    ```zsh
    python tests/test_tool.py

    # 或者

    cd tests && python test_tool.py
    ```

    但是不幸的告诉你，因为你想`from ..importer.tool.tool import a_tool`, 这个`..`实际上是`/pojo/importer`

    而你的入口不是`pojo/importer`，而是 `pojo/importer/tests/test_tool.py`

    这个和Python搜索的机制有关，因为它不是基于你在哪个目录(`pojo/importer`)执行就在哪里开始寻找的

    而是基于你当前执行的文件(`pojo/importer/tests/test_tool.py`)

    入口`test_tool.py`的第一行是`from ..importer.tool.tool import a_tool`, 执行会直接报错`attempted relative import with no known parent package`

    因为入口就不能带有任何的相对引用，任何`from ..`或者`from .`都不对

    好了，那怎么办，我就是要执行`test_tool.py` 来测试`tool.py`

    那就去掉`from ..` 就好了，所以需要提升`test_tool.py`到根目录，即`pojo/importer/test_tool.py`
    且导入改为 `from importer.tool.tool import a_tool`
    然后在根目录下执行`python test_tool.py`

    但是你发现没，结构会变得很丑

    ```zsh
    pojo
    └── importer
        ├── README.md
        ├── importer
        │   ├── __init__.py
        │   ├── func1.py
        │   ├── func2.py
        │   └── tool
        │       ├── __init__.py
        │       └── tool.py
        ├── importer.ipynb
        ├── main.py
        ├── test_tool.py
        └── tests
    ```

    因为本来就是想把`test_*.py`都放在`tests/`下的，现在好了，只能离开`tests/`目录，也就是说未来`test_*.py`越多，根目录下就全是`test_*.py`,这也太丑了吧

    那怎么办，那只能别用相对引用了，下面介绍一个基于某个立足点的绝对调用，是更为常见的调用方式

- 使用绝对导入法（一般基于项目名）

    我们先恢复到最初的目录:

    ```zsh
    pojo
    └── importer
        ├── README.md
        ├── importer
        │   ├── __init__.py
        │   ├── func1.py
        │   ├── func2.py
        │   └── tool
        │       ├── __init__.py
        │       └── tool.py
        ├── importer.ipynb
        ├── main.py
        └── tests
            └── test_tool.py
    ```

    在 `test_tool.py` 中，站在根目录(`pojo/importer`)下使用绝对导入法，即：

    ```python
    from importer.tool.tool import a_tool

    def test_tool():
        a_tool()
    ```

    再次执行

    ```zsh
    python tests/test_tool.py

    # 或者

    cd tests && python test_tool.py
    ```

    而显然，你当前执行入口(`pojo/importer/tests/test_tool.py`) 也不认识文件第一行的`from importer.tool.tool import a_tool` 所以失败了，报错`No Module named 'importer'`

    所以你得强制在命令行中指明你当前执行的路径(`export PYTHONPATH=pojo/importer`)告诉Python我就是立足于这个目录来写导入的，给我看好了

    因此完整命令是`export PYTHONPATH=pojo/importer && python tests/test_tool.py`

    成功了！好不容易，因为你告诉了Python，从`pojo/importer`开始搜索,基于这个立足点，也是我们代码上的立足点，所以没有问题

    这个方法在什么文件下都适用，是万金油，然后推荐整个项目的引用方式都基于同一个立足点（一般就是根目录），更加井然有序

- 插个题外话，如果你需要使用`pytest`，那么你得在和`tests`同级目录下添加`pytest.ini`，并指定`testpaths`

    ```ini
    [pytest]
    testpaths = tests
    ```

    然后也是在根目录下执行`pytest`, `pytest`会将执行路径当成入口，和我们预料的一样。
    而如果你在目录(`pojo/importer/tests`)下执行`pytest`, 同样会报错，因为`test_tool.py`的第一行是`from importer.tool.tool import a_tool`，而`pojo/importer/tests`并不认识`importer`，所以报错。

- 那 main 函数一般写哪呢？

    一般来说会写在 `pojo/importer/main.py`, 为什么呢？ 因为刚才的学习中我们知道，函数入口是很重要的，那么就需要尽可能给高的位置，这样在调用的时候，就不容易出问题,而且站在根目录下进行导入，和其他文件的导入想法是一样的，整体结构会更漂亮，所以一般会写在 `pojo/importer/main.py`, 但是你想写在 `pojo/importer/importer/main.py`也没有问题，看你想在哪个目录下执行导入，是`pojo/importer` 还是 `pojo/importer/importer`, 根据上面的相对导入可以知道，函数入口不要出现在导包范围内，所以如果`pojo/importer/importer/`内使用了相对导入的话，`main.py`执行肯定是报错的，所以这个办法有风险，因此不推荐。

- 一些相对导入的QA

  - 还是这个项目结构:D

    ```zsh
    pojo
    └── importer
        ├── README.md
        ├── importer
        │   ├── __init__.py
        │   ├── func1.py
        │   ├── func2.py
        │   └── tool
        │       ├── __init__.py
        │       └── tool.py
        ├── importer.ipynb
        ├── main.py
        └── tests
            └── test_tool.py
    ```

  - 在`func1.py`或者`func2.py`内可以使用相对导入吗，如

    ```python
    from .tool.tool import a_tool
    from .func1 import foo
    from .func2 import bar
    ```

    : 可以的，因为只要函数不是程序的执行入口，相对导入不会影响正确性

  - 那在`tool.py`可以使用相对导入吗，如

    ```python
    from ..func1 import foo
    ```

    : 可以的，理由同上

  - 那更深的结构还能使用相对导入吗，如新增与`pojo/importer/importer/tool`同级的`api/`及`api/v1.py`？ 截取部分项目结构如下

    ```zsh
    ├── importer
    │   ├── __init__.py
    │   ├── api
    │   │   ├── __init__.py
    │   │   └── v1.py
    │   ├── func1.py
    │   ├── func2.py
    │   └── tool
    │       ├── __init__.py
    │       └── tool.py
    ```

    : 可以的，在`v1.py`直接使用

    ```python
     from ..tool.tool import a_tool
    ```

    即可, 其他的结构亦是如此，相对导入都是可以的，不可以的是程序的执行入口出现在了相对导入的范围内，而执行入口是写执行函数的人决定的，非程序执行入口函数不用担心任何关于相对导入的问题，只要不循环引用就好了。
