# cppminer

**This is a modified fork of [Kolkir's cppminer](https://github.com/Kolkir/cppminer) for easier out-of-the-box running.**



cppminer produces a [code2seq](https://github.com/tech-srl/code2seq) compatible datasets from C++ code bases.

Experimental [C++](https://drive.google.com/file/d/15BDd6zHFkVJXl95FG4JnnSse48k1UR3E/view?usp=sharing) dataset mined from the Chromium project sources.

This tool consists from three scripts which should be run consistently.



# 0. Installation and Dependencies 

### Python Dependencies

After creating your new environment, install all python dependencies with:

```bash
cd src
pip3 install -r requirements.txt
```



### Libclang Installation

The `miner.py` script runs on libclang-16-dev package by default. 

```bash
# For Ubuntu installation
wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key | sudo apt-key add -
sudo add-apt-repository "deb http://apt.llvm.org/focal/ llvm-toolchain-focal-16 main"
sudo apt-get update
sudo apt-get install libclang-16-dev
```



The libclang.so file is found in `/usr/lib/x86_64-linux-gnu/libclang-16.so.1` by default. It is recommended to create a symbolic link for easier reference later on: 

```bash
sudo ln -s /usr/lib/x86_64-linux-gnu/libclang-16.so.1 /usr/lib/x86_64-linux-gnu/libclang.so
```



### Generating `compile_commands.json`

If you have a make file:

```bash
sudo apt-get install bear
bear make -j$(nproc)
```



If you are using cmake, add this following to your `CmakeLists.txt` file:

```cmake
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
```





### Integrating into code2seq

When cloning Kolkir's Tensorflow 2.1 version of [code2seq](https://github.com/Kolkir/cppminer), run the scripts on Python 3.7, with  `requirements_code2seq.txt` pip installed. 



Note that code2seq is somewhat version sensitive, and may differ depending on your GPU and CUDA version. For more information in dependencies, please see the following articles:

- https://github.com/tech-srl/code2seq/issues/117
- https://github.com/tech-srl/code2seq/issues/90



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
The `merge.py` is the utility which concatenates all raw file, shuffles them and produce three files `dataset.train.c2s`, `dataset.test.c2s` and `dataset.val.c2s` into the given directory. Also it can clean source files after merging. The important settings is the `map_file_size` which defines the size of the database file used for merging, you should increase default value of 6Gb for large datasets. 

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
                        size of the DB file, default(100000000000 bytes)
~~~



# 3. Code2vec preprocess

The third utility is the `preprocess.sh` from the `code2seq` folder, this is modified script from the original project which generates dataset in format suitable for the `code2seq` model. In general it creates new files with truncated and padded number of paths for each example.

```bash
chmod +x preprocess.sh
./preprocess.sh <input_folder> <output_folder>
```

