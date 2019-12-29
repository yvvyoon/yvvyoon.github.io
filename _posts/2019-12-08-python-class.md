---
title: 'Python 기본 - Class'
date: 2019-12-08 18:48:00
categories: Python
---

파이썬 웹 프레임워크인 Django, Flask를 공부하면서 정작 기반이 되는 파이썬 언어 자체의 기본에 대해 취약해서 혹시나 면접같은 장소에서 깊은 질문을 받으면 어버버거릴 가능성이 너무 높다는 걸 느꼈다. Django 프로젝트의 CBV와 테스트 코드 작성을 진행하면서 자주 등장했던 `@classmethod` 데코레이터가 무슨 역할을 하는지도 궁금해졌다. 이 데코레이터를 알기 위해선 파이썬 클래스의 개념부터 잘 알아야 한다.

시작해보자.



## Class

파이썬에서 클래스 생성은 여타 언어와 다르게 `new` 키워드를 사용하지 않는다.

기본적으로 `SnakeCase`를 사용하는 변수와 함수와 달리 클래스의 네이밍컨벤션은 `CamelCase`를 사용한다.  

```python
# airtravel.py

class Flight:
    def __init__(self):
        print('init')
        super().__init__()

    def __new__(cls):
        print('new')

        return super().__new__(cls)

    def number(self):
        return 'SN060'
```

 위 코드가 기본적인 클래스와 인스턴스를 생성하는 코드인데 `__init__` 메소드는 `self`, `__new__` 메소드는 `cls`를 argument로 사용하고 있다. 이 부분은 유닛 테스트를 진행하면서 굉장히 많은 의문이 들었다.

결론적으로, `cls`를 argument로 받는 메소드는 클래스 메소드이고, `self`를 argument로 받는 메소드는 인스턴스 메소드이다. 클래스 메소드는 `@classmethod`라는 데코레이터를 사용하여 선언한다.

클래스 메소드는 인스턴스를 따로 선언할 필요 없이 호출할 수 있다.

```python
class ClassMethod:
  @classmethod
  def print_name(cls):
    print('My name is %s' % (cls.__class__.__name__))
```

```
>>> ClassMethod.print_name()
My name is type
```



```
>>> from airtravel import Flight
>>> f = Flight()
new
init
```



적지 않은 사람들이 `__init__`이 생성자라고 말하는데 생성자가 아니다. `__new__` 메소드가 생성자이다. 위 코드처럼 `f = Flight()`생성자로 객체를 생성하라고 호출받으면 `__new__` 메소드를 호출하여 객체를 생성 및 할당하고, `__new__`메소드가   `__init__` 메소드를 호출하여 객체에서 사용할 값들을 초기화한다.

파이썬에서 일반적으로 클래스를 생성할 때 `__init__` 메소드만 오버라이딩하여 생성이 아닌 객체 초기화에만 이용한다.



### 클래스 변수와 인스턴스 변수

`self` 객체가 인스턴스 자체를 가리키기 때문에  `__init__` 메소드에 전달하는 `self.AAA` 파라미터는 인스턴스 변수이다. 반면에 클래스 내에서 self없이 선언하는 변수를 `클래스 변수`라고 한다. 클래스 내에서 공유되기 때문에 이 클래스로 생성한 모든 인스턴스에서 사용할 수 있다.



```python
# self없이 클래스 내에서 바로 선언한다.

class Sample:
  AAA = aaa
  
  def method():
    pass
  
  ...
```



```python
class Flight:
    class_attr = []

    def add_class_attr(self, number):
        Flight.class_attr.append(number)


first = Flight()
second = Flight()

first.add_class_attr(11)
print(Flight.class_attr)	# 11
print(first.class_attr)		# 11
```



위 코드를 보면 분명히 `g` 인스턴스를 통해 `class_attr`을 할당해주지 않았는데도 동일한 값을 출력하고 있다. 여러 인스턴스가 클래스 변수을 공유하고 있음을 알 수 있다.



클래스 변수과 인스턴스 변수를 같은 이름으로 선언해보자.

```python
class Flight:
    class_attr = []

    def __init__(self):
        self.class_attr = []

    def add_class_attr(self, number):
        Flight.class_attr.append(number)

    def add_instance_attr(self, number):
        self.class_attr.append(number)


first = Flight()
second = Flight()

first.add_class_attr(5)
print(first.class_attr)		# [5]
print(Flight.class_attr)	# []
print(second.class_attr)	# []
```



클래스 변수과 인스턴스 변수를 동일하게 선언 후 호출해보면 클래스 변수보다 인스턴스 변수를 먼저 접근하는 것을 확인할 수 있다.



### 상속





### 정적 메소드

정적 메소드는 클래스에서 직접 접근 및 참조가 가능한 메소드이다. 파이썬에서 정의한 정적 메소드에는 staticmethod와 classmethod가 있는데 staticmethod는 추가되는 argument가 없는 반면, classmethod는 첫 번째 argument로 클래스를 입력한다.

