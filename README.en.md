# IMPORTER

English | [中文](./README.md)  

## Summary
- It is recommended to use `module` (directories with `__init__.py`), which supports both relative imports and absolute imports.
- Relative imports: The `main.py` in `python main.py` should not appear within (or rather, should be above) the scope covered by relative imports and should not contain relative imports itself.
- Absolute imports: Unify the starting point for imports (usually the root directory). When executing, emphasize the import starting point by using the command like `export PYTHONPATH=rootdir/ && python main.py`.
- It is recommended to use a combination of relative imports and absolute imports. This doesn't mean mixing them within one file, but having both in a project and using them flexibly.

## Analysis

- Example project structure

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

- Using relative imports

    It's in the form of `from..x import foo | from.y import bar`.

    It must be ensured that the position of the executing function (the `main.py` in `python main.py`) is higher than all the scopes covered by relative imports, and there should be no relative imports in `main.py` because `main.py` is the entry point of the program, and the program entry point is the highest-level directory in Python's discovery mechanism.

    Suppose we want to test the function `a_tool` in `tool.py` in `test_tool.py` now. Using the relative import method, it would be like this:

    ```python
    from..importer.tool.tool import a_tool
    def test_tool():
        a_tool()

    ```

    We are currently in the directory `pojo/importer` (hereinafter referred to as the root directory) and start to try to execute `test_tool.py`.

    The first thought would be

    ```zsh
    python tests/test_tool.py

    # Or

    cd tests && python test_tool.py
    ```

    But unfortunately, because you want to use `from..importer.tool.tool import a_tool`, this `..` actually refers to `/pojo/importer`.

    And your entry point is not `pojo/importer`, but `pojo/importer/tests/test_tool.py`.

    This is related to Python's search mechanism, because it doesn't start searching from the directory where you are executing (`pojo/importer`).

    Instead, it's based on the currently executing file (`pojo/importer/tests/test_tool.py`).

    The first line of the entry `test_tool.py` is `from..importer.tool.tool import a_tool`, and when executed, it will directly report an error `attempted relative import with no known parent package`.

    Because the entry point cannot have any relative references, any `from..` or `from.` is incorrect.

    Well, what should we do? I just want to execute `test_tool.py` to test `tool.py`.

    Just remove `from..`, so we need to move `test_tool.py` up to the root directory, that is, `pojo/importer/test_tool.py`, and change the import to `from importer.tool.tool import a_tool`, and then execute `python test_tool.py` in the root directory.

    But have you noticed that the structure will become very ugly?

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

    Because originally we wanted to put all `test_*.py` under `tests/`, but now, we have to leave the `tests/` directory. That is to say, the more `test_*.py` there are in the future, the root directory will be full of `test_*.py`. How ugly is that!

    So what should we do? Then we can't use relative imports. The following introduces an absolute call based on a certain starting point, which is a more common calling method.

- Using absolute imports (usually based on the project name)

    Let's first restore to the original directory:

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

    In `test_tool.py`, using the absolute import method standing at the root directory (`pojo/importer`), it would be like this:

    ```python
    from importer.tool.tool import a_tool

    def test_tool():
        a_tool()
    ```

    Execute it again:

    ```zsh
    python tests/test_tool.py

    # Or

    cd tests && python test_tool.py
    ```

    Obviously, the current execution entry point (`pojo/importer/tests/test_tool.py`) doesn't recognize `from importer.tool.tool import a_tool` in the first line of the file, so it fails and reports an error `No Module named 'importer'`.

    So you have to forcefully specify the current execution path in the command line (`export PYTHONPATH=pojo/importer`) to tell Python that I'm basing the imports on this directory. Pay attention.

    So the complete command is `export PYTHONPATH=pojo/importer && python tests/test_tool.py`.

    Success! Finally, because you told Python to start searching from `pojo/importer`, based on this starting point, which is also the starting point in our code, there's no problem.

    This method is applicable to any file and is a universal solution. Then it is recommended that the reference methods in the whole project are all based on the same starting point (usually the root directory), which makes it more organized.

- By the way, if you need to use `pytest`, you have to add `pytest.ini` at the same level as `tests` and specify `testpaths` and `pythonpath`. As for why some repositories can run accurately without specifying `pythonpath`, I don't know.

    ```ini
    [pytest]
    testpaths = tests
    pythonpath =. # for import xx, removing this will make it unable to find packages.

    ```

    Then also execute `pytest` in the root directory. `pytest` will take the execution path as the entry point, just as we expected.

    And if you execute `pytest` in the directory (`pojo/importer/tests`), it will also report an error because the first line of `test_tool.py` is `from importer.tool.tool import a_tool`, and `pojo/importer/tests` doesn't recognize `importer`, so it reports an error.

- Where is the main function usually written?

    Generally, it will be written in `pojo/importer/main.py`. Why? Because from what we've learned before, the function entry point is very important, so it needs to be given as high a position as possible. In this way, when calling, it's less likely to have problems. Moreover, importing under the root directory has the same idea as importing in other files, and the overall structure will be more beautiful. So generally, it will be written in `pojo/importer/main.py`. But it's also okay if you want to write it in `pojo/importer/importer/main.py`. It depends on which directory you want to execute imports under, whether it's `pojo/importer` or `pojo/importer/importer`. According to the relative imports mentioned above, the function entry point should not appear within the scope of package imports. So if relative imports are used in `pojo/importer/importer/`, executing `main.py` will definitely report an error. So this approach has risks and is not recommended.

- Some Q&A about relative imports

  - Still with this project structure:

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

  - Can relative imports be used in `func1.py` or `func2.py`, like

    ```python
    from.tool.tool import a_tool
    from.func1 import foo
    from.func2 import bar
    ```

    Answer: Yes, because as long as the function is not the execution entry point of the program, relative imports will not affect correctness.

  - Can relative imports be used in `tool.py`, like

    ```python
    from..func1 import foo
    ```

    Answer: Yes, for the same reason as above.

  - Can relative imports still be used in a deeper structure? For example, adding `api/` at the same level as `pojo/importer/importer/tool` and `api/v1.py`? Part of the project structure is shown as follows:

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

    Answer: Yes. In `v1.py`, just use

    ```python
     from..tool.tool import a_tool
    ```

    That's it. It's the same for other structures. Relative imports are okay. What's not okay is that the execution entry point of the program appears within the scope of relative imports. And the execution entry point is determined by the person who writes the executing function. Functions that are not the program execution entry point don't need to worry about any issues related to relative imports, as long as there's no circular reference. 