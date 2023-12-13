## Dart 使用



#### hello, dart!

保存如下代码至 `hello.dart` 文件

```dart
void main() {
  print('hello, dart!');
}
```

终端执行 `dart hello.dart`  或者 `dart run hello.dart`

#### import 

```dart
import 'dart:math';

void main() {
  print(sin(45 * pi / 180);
}
```

#### data types in dart

```dart
int
double 
num 
dynamic
String  
```

#### Creating Constant Variables

```dart
const int myInteger = 10;

final double myDouble = 3.14;
```

#### 类型判断

```dart
num myNumber = 3.14 
print(myNumber is double); // true
print(myNumber is int); // false
```

#### 类型转换

```dart
var integer = 100;
var decimal = 12.5;
integer = decimal.toInt();
```

#### 默认参数值

```dart
bool withinTolerance(int value, [int min = 0, int max = 10]) {
  return min <= value && value <= max
}

// 调用
withinTolerance(5)
withinTolerance(9, 7, 11)  
```

#### 命名参数(Naming Parameters) 

```dart
bool withinTolerance(int value, {int min = 0, int max = 10}) {
  return min <= value && value <= max
}

// 调用
withinTolerance(9, min:7, max:11)  
```

必选命名参数

```dart
bool withinTolerance({required int value, int min = 0, int max = 10}) {
  return min <= value && value <= max
}

// 调用
withinTolerance(9, min:7, max:11)  
```

####  .. 级联 （Cascade Notation）

```dart
final user = User();
user.name = 'name';
user.id = 42;

可以写成
final user = User() 
  ..name = 'name'
  ..id = 42;
```

#### Short‐Form Constructor

```dart
User(int id, String name) {
  this.id = id;
  this.name = name;
}

// equals 
User(this.id, this.name);
```

#### Named Constructors

```dart
ClassName.identifierName()

// unnamed constructor
ClassName()
  
// named constructor
ClassName.identifierName()  
```

#### Forwarding Constructors

```dart
User.anonymous() : this(0, 'anonymous');
```

#### Optional and Named Parameters

#### Singleton Pattern

```dart
class MySingleton { 
  MySingleton._();
	static final MySingleton instance = MySingleton._();
}

// 或者
class MySingleton { 
  MySingleton._();
	static final MySingleton _instance = MySingleton._();
  
  factory MySingleton() => _instance;
}  
```

####  Initializer List

```dart
class User {
  User(String name) 
    : _name = name;
  String _name;
}
```

