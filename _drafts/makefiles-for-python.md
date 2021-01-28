---
layout: post
title: Makefiles for Python
---

Lately, I have been getting back into using Make.  Among other things, 
I find it's a good way to create some runnable documentation for a repository.
It is also a handy place to keep report-generating scripts that can be just
as easily run locally as they are on the CI server.


```Make
PHONY: static test cover dependency_check questionable_tests

APP_DIR=$(CURDIR)/python
PYTHON_SOURCES=$(shell find python -type f -name *py)
TEST_SOURCES=$(shell find tests/unit -type f -name *py)

ifdef USE_CURRENT_PYTHON
    PYTHON_DIR=
else
    PYTHON_DIR=venv/bin/
endif

static: mypy.out pylint.out pur.out

test: pytest_unit.xml

cover: coverage.xml

dependency_check: safety.out

questionable_tests: venv $(TEST_SOURCES)
	PYTHONPATH=$(APP_DIR) $(PYTHON_DIR)pytest --junitxml=pytest_unit.xml -vm questionable tests/unit

venv: scripts/requirements.txt
ifndef USE_CURRENT_PYTHON
	virtualenv -ppython3.8 venv
endif
	$(PYTHON_DIR)pip install -U pip
	$(PYTHON_DIR)pip install -Ur scripts/requirements.txt

vulture.out: venv $(PYTHON_SOURCES)
	$(PYTHON_DIR)vulture --min-confidence 80 python | tee vulture.out

mypy.out: venv $(PYTHON_SOURCES)
	$(PYTHON_DIR)mypy python --show-error-context | tee mypy.out

pylint.out: venv pylintrc $(PYTHON_SOURCES)
	$(PYTHON_DIR)pylint python | tee pylint.out

pytest_unit.xml: venv $(PYTHON_SOURCES) $(TEST_SOURCES)
	PYTHONPATH=$(APP_DIR) $(PYTHON_DIR)pytest --junitxml=pytest_unit.xml tests/unit

coverage.xml: venv .coveragerc $(PYTHON_SOURCES) $(TEST_SOURCES)
	PYTHONPATH=$(APP_DIR) $(PYTHON_DIR)pytest --junitxml=pytest_unit.xml --cov-report=xml:coverage.xml --cov-report=html:htmlcov --cov=python tests/unit

safety.out: venv
	$(PYTHON_DIR)safety check --full-report | tee safety.out

pur.out: venv
	$(PYTHON_DIR)pur -r scripts/requirements.txt -o latest_requirements.txt > pur.out

```