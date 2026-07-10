# 单元测试指南

## Python (pytest)

```python
import pytest

def test_add():
    assert 1 + 2 == 3

def test_divide_by_zero():
    with pytest.raises(ZeroDivisionError):
        1 / 0
```

## JavaScript (Jest)

```javascript
describe('calculate', () => {
  test('adds 1 + 2 to equal 3', () => {
    expect(1 + 2).toBe(3)
  })
})
```