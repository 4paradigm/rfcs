- Start Date: 2021-05-13
- Target Major Version: 1.0
- Reference Issues: https://github.com/4paradigm/SparkFE/issues/69
- Implementation PR: 

# Summary

We should provide some tools with easy-to-use API like Python for users to meet the featue extraction requirements like creating FEDB DDL.


# Basic example

Users can use fetool command to create FEDB DDL of creating tables.

```
fetool gen_ddl sql.yaml
```

Users can use fetool command to finish the following tasks without developing.

```
fetool csv_to_parquet /csv_files /parquet_files
fetool sample_parquet /parquet_files
fetool inspect_parquet /parquet_files
fetool check_skew /parquet_files
fetool benchmark 'spark-submit --master local /pyspark_app.py'
```

# Motivation

Now users can use SparkFE for feature extraction with SQL API. Development is required since it's the library of distributed computing and do not solve problems without specific SQLs. 

However, there are some common tools which may be used for general feature extraction scenarios. For examples, we may sample the input dataset and check if its distribution is balanced. These tools are general and useful for feature extraction which may reduce the cost of development if we want to use SparkFE for AI. If we inspect the distribution of dataset in advance, the window skew optimization may use the distribution to achieve better performance.

Therefore, providing the common tools for feature extraction is useful for developers. The easiest way to use is commad-line and we should provide Python and Java API for different developers.

# Detailed design

We want to use Python to wrap the feature extraction tools since Python is easy to use and has integrated with other programming languages.

Users can install this tool with `pip` and all the functions can be called with command-line tool and Python functions. The fetool should be the standard Python package and command-line tool. There are some kinds of secnarios which fetool should support.

* For data processing jobs like converting dataset and sampling dataset, we can use PySpark API which can submit the jobs in Python programming language.
* For benchmark tool which requires running multiple jobs with different Spark distributions, we may use `subprocess` api to invoke the shell commands.
* For some utilities written in Java/Scala like generating FEDB DDL, we may use `py4j` to invoke Java functions for Python API and command-line.

Python package is able to meet the above requirements and easy to maintain. The codebase would be like these.

```
- python
  setup.py
  requirements.txt
  - fetool
    __init__.py
    fetool.py
    csv_to_parquet.py
    sample_parquet.py
    inspect_parquet.py
    check_skew.py
    gen_fdb_ddl.py
    ......
```

# Drawbacks

Since it is the extension of SparkFE, there is no drawback for the existing core project.

Implementation and maintenance cost is small if the they are used by most developers because they don't need to maintain by themselves.

# Alternatives

What other designs have been considered? What is the impact of not doing this?

The command-line tool could be implemented in Java, C++ or other programming languages. But they may require to compile before using which is not easy as Python.

# Adoption strategy

Users may use the source Python scripts or install the tool with `pip install fetool`.

# Unresolved questions

None.
