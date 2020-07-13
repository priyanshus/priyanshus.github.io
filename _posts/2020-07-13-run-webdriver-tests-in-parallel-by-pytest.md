---
layout: post
title: '[WebDriver-1] Run Webdriver Tests in Parallel'
date: '2020-07-13'
published: true
tags: [python,automation,pytest,webdriver,selenium,qa]
---
Very often we find that sequential execution of UI tests becomes expensive to use as it directly impacts the delivery time. Nowadays, parallelization has become one of the essential features to have in a test framework. I have recently started using Python for WebDriver project and after searching through different ways of achieving parallelization, I found an out of the box solution for that using [pytest-xdist](https://pypi.org/project/pytest-xdist/).

So writing something simple like this `pytest -n N` runs the N tests in parallel. A simple use case would be writing selenium tests as below and run them in parallel using `pytest`.

```python
def test_login(self):
  create_driver(browser_to_run)
  login()
  assert success_login()
  quit_driver()
```


Pretty simple!!

But there are a few limitations with the above test.

1. If assert fails, above will not quit the driver and it will end up having several browsers open in your system.
2. The above test can't be run on different browsers at the same time in parallel.

To fix the first issue, we essentially need to move the `create_driver()` and `quit_driver()` out of the test code. That can be achieved by writing a pytest fixture to work like a beforeTest and afterTest block.


```python
@pytest.fixture(autouse=True)
def run_around_test():
    create_driver(browser_to_run)
    yield
    quit_driver()
```

* To access the driver instance in the test code, I used a test profile. It can be a separate blog post.*

The other issue of running the tests in parallel on different browsers can be fixed by generating the parametrized tests at run time. For that, I leveraged `pytest_generate_tests` as below:


```python
# content of conftest.py

def pytest_addoption(parser):
    """CLI args which can be used to run the tests with specified values."""
    parser.addoption('--browser', action="append", default=[], choices=['chrome', 'firefox', 'edge'],
                     help='Your choice of browser '
                          'to run tests.')


def pytest_generate_tests(metafunc):
    """To generate the parametrized tests"""
    browsers = metafunc.config.getoption("browser")
    if "browser_to_run" in metafunc.fixturenames:
        metafunc.parametrize("browser_to_run", browsers)


@pytest.fixture(autouse=True)
def run_around_test(browser_to_run):
    create_driver(browser_to_run)
    yield
    quit_driver()
```


The `pytest_addoption` fetches the browser args passed from command line as `pytest --browser=chrome --browser=firefox --browser=edge` and `pytest_generate_tests` parametrize the fixture named with `browser_to_run`. It also generates three tests at run time to run with parametrized fixture.

The `run_around_test()` is a consumer of fixture `browser_to_run` and `browser_to_run` is parametrized by `pytest_generate_tests()`. Note the `@pytest.fixture(autouse=True)` decorator which specifies that all the tests are consumer of this fixture. Which means tests do not need to specify anything about its fixture. A sample test:


```python
def test_login(self):
    login()
    assert success_login()
```


To run the tests on different browsers in parallel can be simply achieved by `pytest -browser=chrome --browser=firefox --browser=edge -n 3`.
