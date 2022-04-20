---
layout: post
title: "pre-commit basic usage"
description: "描述"
categories: [shell]
tags: [bash, ]
---


* Kramdown table of contents
{:toc .toc}

install

```
pip install pre-commit

```

init
```
git clone https://xxx/xxx.git
cd xxx
pre-commit install

pre-commit sample-config >.pre-commit-config.yaml
```

test
```
pre-commit run --all-files
pre-commit run --files xxx.py

```

sample conf

```

# See https://pre-commit.com for more information
# See https://pre-commit.com/hooks.html for more hooks
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.1.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files

  - repo: https://github.com/talos-systems/conform
    rev: v0.1.0-alpha.25
    hooks:
      - id: conform
        stages:
          - commit-msg

#  - repo: https://github.com/koalaman/shellcheck-precommit
#    rev: v0.7.2
#    hooks:
#      - id: shellcheck
#        args:
#          - --exclude=SC2009,SC2086

  - repo: https://github.com/5xops/mirrors-shellcheck
    rev: v1.0
    hooks:
      - id: shellcheck


#  - repo: https://github.com/pre-commit/mirrors-mypy
#    rev: v0.770
#    hooks:
#      - id: mypy
#        language: python_venv
#        exclude: ^(docs/|example-plugin/|tests/fixtures)

  - repo: https://gitlab.com/pycqa/flake8.git
    rev: 3.9.2
    hooks:
      - id: flake8
        exclude: $(.tox/|.git/|__pycache__/|build/|dist/|.cache|.eggs/)
        args:
          - --ignore=E501,W503,E722,W605

  - repo: https://github.com/PyCQA/pylint
    rev: v2.12.2
    hooks:
      - id: pylint
        language: python_venv
        args:
          - --disable=C0114,C0115,C0116,C0415,E0401,W1401,R0912,R0914,W0212
          - --max-line-length=120
```
