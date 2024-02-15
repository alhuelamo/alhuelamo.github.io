---
title: "The 'indirect' argument in pytest fixture parametrization"
date: 2024-02-15T16:30:00+01:00
publishDate: 2024-02-15T16:30:00+01:00
draft: false
toc: false
images:
categories:
  - code
tags:
  - python
  - pytest
  - testing
---

{{<figure src="/img/julia-koblitz-RlOAwXt2fEA-unsplash.jpg" caption="Photo by [Julia Koblitz](https://unsplash.com/@jkoblitz) in [Unsplash](https://unsplash.com/s/photos/pieces)">}}

The purpose of `indirect=True` in the `pytest.mark.parametrize` decorator was not very clear to me just by reading [pytest documentation](https://docs.pytest.org/en/7.1.x/example/parametrize.html#indirect-parametrization):

> Using the `indirect=True` parameter when parametrizing a test allows to parametrize a test with a fixture receiving the values before passing them to a test

```python
import pytest

@pytest.fixture
def fixt(request):
    return request.param * 3

@pytest.mark.parametrize("fixt", ["a", "b"], indirect=True)
def test_indirect(fixt):
    assert len(fixt) == 3
```

I found this definition rather unclear, and a bit of a mouthful. So I just created a toy example to test it's purpose... (Could this be *meta-testing*? 🤔)

# Hands-on!

After writing some toy examples, the purpose of `indirect` becomes apparent when we remove it:

```python
@pytest.mark.parametrize("fixt", ["a", "b"])
def test_indirect(fixt):
    assert len(fixt) == 3
```

This is what we get after running pytest:

```
collected 2 items

tests/test_indirect.py::test_indirect[a] FAILED            [ 50%]
tests/test_indirect.py::test_indirect[b] FAILED            [100%]

============================ FAILURES ============================
________________________ test_indirect[a] ________________________

my_fixture = 'a'

    @pytest.mark.parametrize("my_fixture", ["a", "b"])
    def test_indirect(my_fixture):
>       assert len(my_fixture) == 3
E       AssertionError: assert 1 == 3
E        +  where 1 = len('a')

tests/test_indirect.py:12: AssertionError
________________________ test_indirect[b] ________________________

my_fixture = 'b'

    @pytest.mark.parametrize("my_fixture", ["a", "b"])
    def test_indirect(my_fixture):
>       assert len(my_fixture) == 3
E       AssertionError: assert 1 == 3
E        +  where 1 = len('b')

tests/test_indirect.py:12: AssertionError
==================== short test summary info =====================
FAILED tests/test_indirect.py::test_indirect[a] - AssertionError: assert 1 == 3
FAILED tests/test_indirect.py::test_indirect[b] - AssertionError: assert 1 == 3
======================= 2 failed in 0.01s ========================
```

What we can see is that `parametrize` is *directly* passing each parameter value to the `fixt` test argument. Therefore, it ignores the fixture and parametrizes the test.

Should we want to parametrize the fixture back, we would set `indirect=True`, and so the tests pass again:

```
collected 2 items

tests/test_indirect.py::test_indirect[a] PASSED            [ 50%]
tests/test_indirect.py::test_indirect[b] PASSED            [100%]

======================= 2 passed in 0.00s ========================
```

pytest would still spawn 2 tests cases—one for each parameter—but this time, the test would receive the result of the fixture, and `parametrize` would send the argument to the fixture instead of the test.

Hence we can conclude that `indirect` controls whether `parametrize` *sends* the arguments to either the test, or a fixture with the same argument's name. If we tried to use `indirect=True`, and there was no fixture `fixt`, we would get an error:

```
_______________ ERROR at setup of test_indirect[a] _______________
file /Users/alberto/Code/pytest-indirect-parametrization/tests/test_indirect.py, line 10
  @pytest.mark.parametrize("fixt", ["a", "b"], indirect=True)
  def test_indirect(fixt):
E       fixture 'fixt' not found
```

# pytest is great!

pytest's fixture system is awesome. I can declare test resources in a very modular, flexible fashion. Then, I can use those resources in my tests, and pytest takes care of running those fixtures for me, and cleaning-up those test resources when each test case is finished. With `parametrize`, I can ensure to cover more cases without having to write more tests. And, if I have fixtures that should be parametrized, I can tweak those parameters from each particular test.

I wrote this piece back in the day as an exercise for me to understand a bit better what I could achieve with pytest. I hope it can be useful to somebody else!

Happy testing! 🧪
