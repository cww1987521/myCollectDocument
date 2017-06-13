# iOS 接口定义规范

[TOC]

---

## 初始化方法
### NS_DESIGNATED_INITIALIZER
对于多个 `init` 方法，苹果给出了一个调用顺序，而我们也应该遵守这种调用顺序，以确保无论外部调用者从哪个入口进入，都能正确的初始化：
![](media/14926777792826/14945768578403.jpg)
上面这张方法调用顺序，很清晰的描述了正确的初始化逻辑。

可以看到真正在进行初始化参数的，是 `initWithTitle:date:`，如果调用者通过 `init` 或者 `initWithTitle:` 进入，都应该确保变量 `title` 和 `date` 能正确赋值，所以 `init` 与 `initWithTitle:` 都通过调用 `initWithTitle:date:` 来初始化。

最后 `initWithTitle:date:` 在通过父类的 init 初始化，并初始化两个变量。

对于这种能初始化全部必需变量的方法，一般可作为 `designed initializer`。所以，可以明确的告诉外部调用者，无论调用哪种初始化方法，最终，都会调用 `designed initializer`：

```objc
- (instancetype)initWithTitle:(NSString *)title date:(NSDate *)date NS_DESIGNATED_INITIALIZER; 
```

一个子类如果有自己的 `designed initializer`，则必须要实现父类的 `designed initializer`。比如一个继承自 `NSObject` 的 `Person` 类，就必须要重写 `init` 方法，并在 `init` 方法中，调用自己的 `designed initializer`，而不是调用 `super` 的初始化方法。如果未实现，可以看到编译警告：

> Method override for the designed initializer of the superclass ‘- init’ not found.

除此之外，子类的 `designed initializer` 方法，在调用 `super` 时，也应该调用 `super` 的 `designed initializer`。

### NS_UNAVAILABLE
相比于 `NS_DESIGNATED_INITIALIZER`，`NS_UNAVAILABLE` 直接禁用其他初始化方法。

```objc
+ (instancetype)new NS_UNAVAILABLE;
- (instancetype)init NS_UNAVAILABLE; ///< 直接标记 init 方法不可用
- (instancetype)initWithID:(NSNumber *)userID;
```

方法一旦标记 `NS_UNAVAILABLE`， 那么在IDE自动补全时，就不会索引到该方法，如果强制调用该方法，编译器会报错（但使用runtime可以动态调用方法）。

除了使用 `NS_UNAVAILABLE` 的其他方式：

```objc
// 作用与 NS_UNAVAILABLE 类似
- (id) init __unavailable;
- (id) init __attribute__((unavailable));
- (id) init UNAVAILABLE_ATTRIBUTE;

// 在调用时给出提示，当然系统提供了宏：OBJC_UNAVAILABLE(<#_msg#>)
- (id) init __attribute__((unavailable("Must use initWithFoo: instead.")));
```

### 总结
* 如果是可以给出默认值的初始化方法，使用 `NS_DESIGNATED_INITIALIZER`。
* 如果是必须要用某参数来初始化的，使用 `NS_UNAVAILABLE`。

---

## 泛型
### __covariant(协变) 与 __contravariant(逆变)
限制集合类型子元素类型，使用 `__convariant` 可以将子类型转换到父类型（[里式转换](https://en.wikipedia.org/wiki/Liskov_substitution_principle)）。如果想支持父类型转换到子类型（**尽量避免这种转换**）可以将 `__covariant` 改为 `__contravariant`。 

```objc
// NSMutableString : NSMutableString
NSDictionary<NSMutableString *, NSMutableString *> *dic1 = @{@"dic1" : @1};
// NSMutableString : NSMutableString
NSDictionary<NSMutableString *, NSMutableString *> *dic2 = nil;
// NSString : NSString
NSDictionary<NSString *, NSString *> *dic3 = nil;

// 将 NSMutableString 赋值给 NSString(子类 -> 父类)
// 编译通过，无警告
dic3 = dic1;

// 将 NSString 赋值给 NSMutableString(父类 -> 子类)
// warning: Incampatible pointer types assgining to ...
// 警告：无法将类型为 <NSString :  NSString> 的字典赋值给 <NSMutableString : NSMutableString>
dic2 = dic3;
```

---

## __kindof
限制集合类型内子元素类型是某个固定类型或其子类，在获取集合内类型时可直接指定接受者类型是其子类即可，不需强制转换。
类似于 `isKingOf:`

```objc
@property (nonatomic,readonly,copy) NSArray<__kindof UIView *> *subviews;
- (nullable __kindof UIView *)viewWithTag:(NSInteger)tag;
```

---

## const
在方法实现时，不会写改参数对象的用 `const` 修饰。

```objc
+ (instancetype)dictionaryWithObjects:(const ObjectType [])objects forKeys:(const KeyType [])keys count:(NSUInteger)cnt;
```

---

## NS_NOESCAPE
与 `swift` 中的 `@noescape` 对应，表示所修饰的 `block` 在方法内能执行完毕。

```objc
- (void)enumerateKeysAndObjectsUsingBlock:(void (NS_NOESCAPE ^)(KeyType key, ObjectType obj, BOOL *stop))block;
```

扩展链接：[swift-envolution](https://github.com/apple/swift-evolution/blob/master/proposals/0012-add-noescape-to-public-library-api.md)

> A closure argument is guaranteed to be executed (if executed at all) before the function returns. This enables the compiler to perform various optimizations, such as omitting unnecessary capturing/retaining/releasing of self.
> 如果闭包能在方法返回之前调用，那么该修饰符可以让编译器做很多优化，比如剔除对 self 的捕获、持有、释放等。

---

## NS_REQUIRES_NIL_TERMINATION
要求最后一个值为 `nil`。

```objc
- (instancetype)initWithObjectsAndKeys:(id)firstObject, ... NS_REQUIRES_NIL_TERMINATION;

// 调用
NSDictionary *dic = [NSDictionary dictionaryWithObjectsAndKeys:@"1", @"1", nil];
```

---

## NS_REQUIRES_SUPER
当前方法必须调用 `super`。

```objc
// UIView
- (void)updateConstraints NS_REQUIRES_SUPER;
```

---

## Nullability
对应 `swift` 的 `optional`。
即使不与 `swift` 对接也推荐使用，增加程序严谨性。
### 关键字
* `null_unspecified`: 对应 `swift` 的隐式解包可选（缺省值）
* `nullable`: 可空，对应 `swift` 的 `?`
* `nonnull`: 不可空，对应 `swift` 的 `!`
* `null_resettable`: `set` 可空，`get` 不可空。仅用于 `property`。

其他关键字：
* `_Nonnull`、`_Nullable`、`_Null_unspecified`
* `__nonnull`、`__nullable`、`__null_unspecified`

解释如下：

> The double-underscored nullability qualifiers (nullable, nonnull, and _null_unspecified) have been renamed to use a single underscore with a capital letter:Nullable, _Nonnull, and _Null_unspecified, respectively). The compiler predefines macros mapping from the old double-unspecified names to the new names for source compatibility.
两个下划线的关键字（`__nullable`，`__nonnull`，`__null_unspecified`）被重命名为了一个下划线的（`_Nullable`，`_Nonnull`，`_Null_unspecified`）。编译器使用宏定义来将两个下划线的关键字映射到一个下划线的，以实现兼容。

即有下划线的关键字要写在定义类型后面，没有下划线的关键字写在类型前面。

### 属性

```objc
@property (nullable, nonatomic, strong) NSNumber *status;
@property NSNumber * __nullable status;
@property NSNumber * _Nullable status;
```

### 函数参数

```objc
- (void)doSomethingWithString:(nullable NSString *)str;
- (void)doSomethingWithString:(NSString * _Nullable)str;
- (void)doSomethingWithString:(NSString * __nullable)str;
```

### block

```objc
- (void)convertObject:(nullable id __nonnull (^)(nullable id obj))handler;
- (void)convertObject:(id __nonnull (^ _Nullable)())handler;
- (void)convertObject:(id _Nonnull (^ __nullable)())handler;
```

### 通用宏
写在 `NS_ASSUME_NONNULL_BEGIN` 与 `NS_ASSUME_NONNULL_END` 之间的属性、方法均会使用 `nonnull` 修饰，我们只需要修饰可选值就可以。

```objc
NS_ASSUME_NONNULL_BEGIN

@interface NSDate : NSObject <NSCopying, NSSecureCoding>

@property (readonly) NSTimeInterval timeIntervalSinceReferenceDate;

- (instancetype)init NS_DESIGNATED_INITIALIZER;
// 如果在其中有变量需要用 nullable 修饰，直接加就好
- (nullable instancetype)initWithCoder:(NSCoder *)aDecoder NS_DESIGNATED_INITIALIZER;

@end

NS_ASSUME_NONNULL_END
```

---

## NSNotificationName 与 FOUNDATION_EXPORT
通知名称的规范定义。
### FOUNDATION_EXPORT
系统 `NSDate` 定义：

```objc
FOUNDATION_EXPORT NSNotificationName const NSSystemClockDidChangeNotification NS_AVAILABLE(10_6, 4_0);
```

`FOUNDATION_EXPORT` 即 `FOUNDATION_EXTERN` 即 `extern`，用于告知编译器该变量或函数可在本地模块或其他模块中使用。在 Cocoa 中， `FOUNDATION_EXTERN` 的定义为：

```objc
// 如果是 C++，如下代码采用 C 编译器
#if defined(__cplusplus)
#define FOUNDATION_EXTERN extern "C"
#else
#define FOUNDATION_EXTERN extern
#endif
```

### NSNotificationName

```objc
typedef NSString *NSNotificationName NS_EXTENSIBLE_STRING_ENUM;
```

`NSNotificationName` 即 `NSString *`，不过这样看着更为直观，所以在 `NSNotification` 中，所有有关通知名称的都用的 `NSNotificationName` 作为参数类型。

`NS_EXTENSIBLE_STRING_ENUM` 是为了与 `swift` 适配。

#### 定义

```objc
// .h
FOUNDATION_EXPORT NSNotificationName const SomeNotification;

// .m
NSNotificationName const SomeNotification = @"SomeNotification";
```

不使用 `#define` 来定义通知是因为作为宏定义，仅作为替换，所以使用的时候，依然是字符串，对比需要使用 `isEqualToString`。而使用 `FOUNDATION_EXPORT NSNotificationName const` 方式定义，则是直接比较地址，效率比 `#define` 要高出许多。

---

## `__attribute__`
### constructor 与 destructor
使用 `constructor` 属性修饰的函数能在 `main()` 函数之前执行，使用 `destructor` 属性修饰的函数在 `main()` 函数结束或 `exit()` 函数调用后执行。
该属性不能修饰 **Objective-C** 的方法。

```objc
__attribute__((constructor))
void runBeforeMain() {
    NSLog(@"run before main");
}
```

⚠️：使用 `constructor` 修饰的函数虽然在 `main()` 函数之前调用，但是依然后于 `+ load()` 方法，所以需要注意各个方法的调用时机。

### deprecated
标明方法、变量、类型已废弃。主要用于标明将来会废弃的方法，方便调用者及时查看。

```objc
- (void)testDeprecated __attribute__((deprecated));
```

使用有明确弃用描述的方法：

```objc
- (void)testDeprecated OBJC_DEPRECATED("This method is no longer supported");

```

### noreturn
告知编译器，该方法不会反悔（会直接结束）。编译器可忽略方法调用完成的处理，进行相应优化。多用于强制程序退出、死循环等情况。

### pure 与 const
用于函数的返回值仅与入参有关，并且函数状态但也。`pure` 除了与入参有关外，还与全局变量有关。

这两个函数均属于函数式编程，且 `const` 比 `pure` 更为严格一些。

[twitter blog](https://blog.twitter.com/2014/attribute-directives-in-objective-c)（需翻墙）给出的建议：

* 虽然对于 `runtime` 来说，加不加 `const` 和 `pure` 关系不大，但是这对提高接口可读性帮助非常大。
* 建议给不需要传入任何参数的方法加上该属性。正因为不需要入参，所以无论何时返回值都是相同的，那么完全可以对返回值进行缓存，之后调用时，直接返回缓存的结果即可。比如，单例的初始方法。
* 如果 **Objective-C** 某方法用 `pure` 或 `const` 修饰了，并且调用非常频繁，那么应该考虑将其设计为 **C** 的接口，可优化函数开销。
* 即便 `pure` 与 `const` 这么好用，但是一旦用错，会造成 **bug** 不易发现。

#### const 注意事项
Twitter 文中提到的例子：

```objc
// 定义枚举 TestEnum
typedef NS_ENUM(NSInteger, TestEnum) {
    TestEnum1,
    TestEnum2,
    TestEnum3
};

// 函数声明
FOUNDATION_EXTERN NSString * EnumToString(TestEnum testEnum);

// 函数实现
NSString * EnumToString(TestEnum testEnum) {
    switch (testEnum) {
        case TestEnum1:
            return @"TestEnum1";
        case TestEnum2:
            return @"TestEnum2";
        case TestEnum3:
            return @"TestEnum3";
        default:
            return @"";
    }
}
```
以上将 enum 转为 string 的函数返回值，仅与传入的 enum 有关，所以完全可以给该函数加上 const：

```objc
FOUNDATION_EXTERN NSString * EnumToString(TestEnum testEnum) OS_CONST;
```

一下实现就会出现问题：

```objc
NSString * EnumToString(TestEnum testEnum) {
    return [NSString stringWithFormat:@"TestEnum%ld", testEnum + 1];
}
```

> 当使用 const 与 pure 时，编译器会将 NSString 缓存到持久内存中，此时引用计数为无穷大。

这里我们动态创建了返回值，意味着该方法中的返回值地址每次都变，所以在编译器返回缓存地址的时候，地址所对应的对象很有可能已经释放了，导致程序 crash。

规则：若果方法或函数使用 `const`，那么返回值一定要是常量，若放回值为引用，则引用也要是常量（地址不变）。

### warn_unused_result
返回值为使用时警告。系统提供宏：`OS_WARN_RESULT` 。

---


