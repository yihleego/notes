# String 字符串

- `String`是`final`修饰的类，不可变，不可继承。
- `StringBuilder`可变，线程不安全，性能好。
- `StringBuffer`可变，线程安全，性能差。
- 判断字符串值是否相等用`equals()`方法，判断对象地址是否相等用`==`。
- Java 7 版本`switch`支持字符串。
- Java 7 版本`intern()`方法，如果常量池中已存在，则返回常量池中的字符串，否则常量池引用指向当前字符串在堆中的地址。  
  Java 6 版本`intern()`方法，如果常量池中已存在，则返回常量池中的字符串，否则复制一份字符串到常量池中，并返回复制的字符串。
- `String.trim()`方法移除字符串首尾的空白字符（空格、tab键、换行符），Java 11 版本新增`String.strip()`方法支持移除`Unicode`空白字符。
- 可以自定义`java.lang.String`类并编译成功，但不会被加载使用。
- Java 9 版本使用`byte[]`替换`char[]`来储存字符串，并新增`coder`字段区分编码类型（`LATIN1`、`UTF16`），目的是为了节约空间。
