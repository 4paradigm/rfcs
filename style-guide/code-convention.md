- Feature Name: `code-convention`
- Start Date: 2021-3-29
- RFC PR: #1 #2

# Coding Convention

## Summary

[summary]: summary

4Paradigm's coding convention apply to C/CPP, Java, Scala, Python, Yaml, Json, Shell Script.

In summary, imperative languages (C/CPP, Java, Scala, Python), is based on Google's Style Guide if have, with a bit adjustment:

- indent: 4 space
- max line length: 120 character
- name convention: we follow Google's naming convention which is 

Concrete style guide can be found:

- [Google CPP Style Guide](https://google.github.io/styleguide/cppguide.html)
- [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html)
- [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html)

Data-Serialization language like yaml/json, should follow:

- indent: 2 space
- max line length: 120 character
- prefer double quote (json excluded)

## Code Style linter and formater

### tools use

| language  | linter                                               | linter config                                                                      | formater                                                           | format config                                                                  |
| --------- | ---------------------------------------------------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------------------ |
| cpp       | [cpplint](https://github.com/cpplint/cpplint)        | default config                                                                     | [clang-format](https://clang.llvm.org/docs/ClangFormat.html)       | [.clang-format](https://github.com/4paradigm/HybridSE/blob/main/.clang-format) |
| java      | [checkstyle](https://checkstyle.sourceforge.io/)     | [style.xml](https://github.com/4paradigm/HybridSE/blob/main/java/style_checks.xml) | [eclipse.jdt](https://github.com/eclipse/eclipse.jdt.core) | [eclipse-formater.xml](https://github.com/4paradigm/HybridSE/blob/main/java/eclipse-formatter.xml)                                                        |
| python    | [pylint](https://www.pylint.org/)                    | [pylintrc](https://github.com/4paradigm/HybridSE/blob/main/pylintrc)               | [yapf](https://github.com/google/yapf)                             | default with google style                                                      |
| yaml/json | -                                                    | -                                                                                  | [prettier](https://prettier.io/)                                   | [prettierrc](https://github.com/4paradigm/HybridSE/blob/main/.prettierrc.yml)  |
| shell     | [shellcheck](https://github.com/koalaman/shellcheck) | default                                                                            | [shfmt](https://github.com/mvdan/sh)                               | default with indednt = 4                                                       |

By default, linters should setup correctly in CICD in order to enforce convention in place like Pull Request. The formaters just give suggestions, there is no restriction which specific formater to use.
formater do not always solve lint error, dig hard to the lint rule.

### Cpp

[style-cpp]: style-cpp

[.clang-format](https://github.com/4paradigm/HybridSE/blob/main/.clang-format) is based on Google style config, with indent and max line length modification.

### java

- checkstyle as style linter
  - use template [google_checks.xml](https://github.com/checkstyle/checkstyle/blob/master/src/main/resources/google_checks.xml), modify indent and max line length
  - use checkstyle maven plugin: <https://maven.apache.org/plugins/maven-checkstyle-plugin/>
- google-java-format: use `--aosp` style, the 4 space indent version of google java style

## Scala

### Python

- pylint: based on config provided by google: [pylintrc](https://google.github.io/styleguide/pylintrc), modify indent and max line length.
- yapf:

### Yaml/Json/Xml

- config in [prettierrc](https://github.com/4paradigm/HybridSE/blob/feat/style-and-doc/.prettierrc.yml)
  - tabWidth: 2
  - singleQuote: false
  - printWidth: 120

### Shell Script

- follow shellcheck rules: <https://github.com/koalaman/shellcheck/wiki/Checks>
- follow shfmt format rules: <https://github.com/mvdan/sh/blob/master/cmd/shfmt/shfmt.1.scd>

### doxygen

- doxygen comment rules: <https://www.doxygen.nl/manual/docblocks.html>
- doxygen markdown support: <https://www.doxygen.nl/manual/markdown.html>
- c++ comment style: <https://github.com/4paradigm/HybridSE/discussions/27#discussioncomment-546319>
- java/scala comment style: @tobegit3hub
- python comment style: @tobegit3hub
