---
layout: post
title: django model 동적으로 생성하기
author: lazyer
tags: django
---





Django에서 Model을 생성할 때, 동일한 스키마를 가진 테이블을 2개 이상 생성하는 것을 시도해보게 되었다.

은행의 거래 내역을 저장하는 테이블을 설계하는데, 한명이 하루에 5개의 거래내역을 생성한다고 가정하였을 때 1년에 1825개 (5*365), 10년에 18250개의 레코드가 생성된다. 사용자가 총 1,000,000명이라고 가정한다면 레코드가 총 18,250,000,000개 (182억 5천만개)가 된다.

실제로 백만명이 이렇게 쓰지는 않겠지만.. 하나의 테이블에 모두 저장하는 것 보다 여러 테이블에 나누어 저장하는 것이 좋을 것 같았다. (계좌별로 hash값을 생성하여, hash값에 매핑시켜놓은 거래내역 테이블에 사용자의 거래 내역을 저장하려고 한다.)

여러 테이블에 나누어 거래내역을 저장하는 것이 옳은 것인지는 아직 모르겠지만, 우선 Django Model 을 생성해보자.

<br>

**Transaction(거래내역) Table**

```python
class AccountTransaction(models.Model):
    class Meta:
        abstract = True

    account_num = models.ForeignKey('Account', on_delete=models.CASCADE)
    tran_amt = models.PositiveBigIntegerField()
    tran_type = models.CharField(max_length=10, null=False, blank=False)
    tran_detail = models.CharField(max_length=100, null=True, default="")
    tran_time = models.DateTimeField(auto_now_add=True)
```

account_num 은 Account Table을 참조하는 왜래키이고, 기타 필드는 위와 같다.

<br>

동일한 구조를 가진 테이블을 생성해보려고 하니

```python
class AccountTransaction1(models.Model):
    class Meta:
        db_table = "account_transaction_1"

    account_num = models.ForeignKey('Account', on_delete=models.CASCADE)
    tran_amt = models.PositiveBigIntegerField()
    tran_type = models.CharField(max_length=10, null=False, blank=False)
    tran_detail = models.CharField(max_length=100, null=True, default="")
    tran_time = models.DateTimeField(auto_now_add=True)
    

class AccountTransaction2(models.Model):
    class Meta:
        db_table = "account_transaction_2"

    account_num = models.ForeignKey('Account', on_delete=models.CASCADE)
    tran_amt = models.PositiveBigIntegerField()
    tran_type = models.CharField(max_length=10, null=False, blank=False)
    tran_detail = models.CharField(max_length=100, null=True, default="")
    tran_time = models.DateTimeField(auto_now_add=True)
```

이렇게 생성해서는 영 마음에 들지 않는다. 테이블을 10개를 생성한다면 10개의 클래스를 각각 작성해야한다.

<br>

상속을 이용하여 생성을 위 코드를 조금 단순화하면 아래와 같다.

```python
class AccountTransaction(models.Model):
    class Meta:
        abstract = True

    account_num = models.ForeignKey('Account', on_delete=models.CASCADE)
    tran_amt = models.PositiveBigIntegerField()
    tran_type = models.CharField(max_length=10, null=False, blank=False)
    tran_detail = models.CharField(max_length=100, null=True, default="")
    tran_time = models.DateTimeField(auto_now_add=True)
    

class AccountTransaction1(AccountTransaction):
    class Meta:
        db_table = "account_transaction_1"


class AccountTransaction2(AccountTransaction):
    class Meta:
        db_table = "account_transaction_2"
```

<br>

For 루프를 이용하여 지정된 숫자만큼 테이블을 자동으로 생성을 하기위해서 코드를 수정해 보았다.

```python
class AccountTransaction(models.Model):
    class Meta:
        abstract = True

    account_num = models.ForeignKey('Account', on_delete=models.CASCADE)
    tran_amt = models.PositiveBigIntegerField()
    tran_type = models.CharField(max_length=10, null=False, blank=False)
    tran_detail = models.CharField(max_length=100, null=True, default="")
    tran_time = models.DateTimeField(auto_now_add=True)


def create_account_transaction(val):
    class Model(AccountTransaction):
        class Meta:
            db_table = "account_transaction_{}".format(val)
    return Model


tables = list()
for i in range(10):
    tables.append(create_account_transaction(i))
```

<br>

이 상태에서 makemigrations을 실행해보면..

```python
  bank\migrations\0001_initial.py
    - Create model Model
```

`AccountTransaction`를 상속받아 생성되는 클래스는 Model 이라는 model 1개 뿐이다.. Django에서 개별 클래스 모델을 생성하고 Meta class 데이터를 다르게 지정하더라도, `클래스명이 같으면 같은 model로 인식을 한다`.

<br>

서로 다른 클래스명을 가지는 클래스를 생성하여야하는데, `globals`를 이용하여 해결하였다.

```python
for i in range(10):
    model_name = f'account_transaction_{i:02}'
    globals()[model_name] = type(model_name, (AccountTransaction, ), {'__module__': AccountTransaction.__module__})
```

makemigrations을 실행해보면 10개 테이블이 생성된다!

```
bank\migrations\0001_initial.py
    - Create model account_transaction_09
    - Create model account_transaction_08
    - Create model account_transaction_07
    - Create model account_transaction_06
    - Create model account_transaction_05
    - Create model account_transaction_04
    - Create model account_transaction_03
    - Create model account_transaction_02
    - Create model account_transaction_01
    - Create model account_transaction_00
```

---

<br>

**globals**

`globals`는 전역변수를, `locals`는 지역변수의 값들을 dictionary 형태로 return 하는데, dictionary와 같은 방식으로 값을 생성 할 수 있다. 선언된 변수, 정의된 클래스 모두 `globals`에서 확인 가능하다.

<br>

```sh
>>> globals()
{'__name__': '__main__', '__doc__': None, '__package__': None, '__loader__': <class '_frozen_importlib.BuiltinImporter'>, '__spec__': None, '__annotations__': {}, '__builtins__': <module 'builtins' (built-in)>}

>>> a = 1  

>>> globals()
{'__name__': '__main__', '__doc__': None, '__package__': None, '__loader__': <class '_frozen_importlib.BuiltinImporter'>, '__spec__': None, '__annotations__': {}, '__builtins__': <module 'builtins' (built-in)>, 'a': 1}

>>> type(globals()['a']) 
<class 'int'>

>>> class Test:
...     pass
... 
>>> globals()['Test']
<class '__main__.Test'>
```

<br>

```python
# a.py
globals()["value"] = 1
```

```python
# b.py
print(value)
```

너무 당연한 것일 수 있지만. 동일 파이썬 패키지내에 a.py, b.py가 위와 같을 때 b.py를 실행하면 value라는 변수가 없다고 에러가 발생한다. b.py에서 value를 사용하려면 `from a import value`를 추가 해주어야 한다. 당연하지만 `globals`는 동일 파일 내에 전역변수를 생성한다.