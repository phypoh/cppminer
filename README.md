# cppminer
cppminer produces a [code2seq](https://github.com/tech-srl/code2seq) compatible datasets from C++ code bases.

This tool consists from three scripts which should be consistently.
 
# 1. Miner 
The `miner.py` is the main utility which traverse c++ sources, parse them and produce raw dataset files.

It has following command line interface:
~~~
usage: miner.py [-h] [-c contexts-number] [-l path-length] [-d ast-depth] [-p processes-number] [-e libclang-path] path out

positional arguments:
  path                  the path sources directory
  out                   the output path

optional arguments:
  -h, --help            show this help message and exit
  -c contexts-number, --max_contexts_num contexts-number
                        maximum number of contexts per sample
  -l path-length, --max_path_len path-length
                        maximum path length (0 - no limit)
  -d ast-depth, --max_ast_depth ast-depth
                        maximum depth of AST (0 - no limit)
  -p processes-number, --processes_num processes-number
                        number of parallel processes
  -e libclang-path, --libclang libclang-path
                        path to libclang.so file
~~~

The input path is traversed recursively and all files with following extensions `c, cc, cpp` are parsed. 
It is recommended to use the [c++ compilation database](https://clang.llvm.org/docs/JSONCompilationDatabase.html) which provides all required compilation flags for project files.

These files have following format:

* Each row is an example.
* Each example is a space-delimited list of fields, where:

    1. The first field is the target label, internally delimited by the "|" character (for example: compare|ignore|case)
    2. Each of the following field are contexts, where each context has three components separated by commas (","). None of these components can include spaces nor commas.

Context's components are a token, a path, and another token.

Each `token` component is a token in the code, split to subtokens using the "|" character.

Each `path` is a path between two tokens, split to path nodes using the "|" character. Example for a context:
```
my|key,StringExression|MethodCall|Name,get|value
```
Here `my|key` and `get|value` are tokens, and `StringExression|MethodCall|Name` is the syntactic path that connects them.

# 2. Merge
The `merge.py` is the utility which concatenates all raw file, shuffles them and produce three files `dataset.train.c2s`, `dataset.test.c2s` and `dataset.val.c2s` into the given directory.
Also it can clean source files after merging. The important settings is the `map_file_size` which defines the size of the database file used for merging, 
you should increase default value of 6Gb for large datasets. 

It has following command line interface:

~~~
usage: merge.py [-h] [-c clear_resources_flag] [-m map_file_size] path

merge resources generated by cppminer to a code2seq dataset

positional arguments:
  path                  the dataset sources path

optional arguments:
  -h, --help            show this help message and exit
  -c clear_resources_flag, --clear_resources clear_resources_flag
                        if True clear resource files
  -m map_file_size, --map_size map_file_size
                        size of the DB file, default(6442450944 bytes)
~~~

# 3. Code2vec preprocess

The third utility is the `preprocess.sh` from the `code2seq` folder, this is modified script from the original project which generates dataset in format suitable for the `code2seq` model.
in general it creates new files with truncated and padded number of paths for each example.