iOS 处理文本的方式在 iOS7 以后发生了改变。

iOS6 及以前， UIKit 中和文本相关的类，主要是用 Webkit 和 core  graphics 中的 string drawing 方法来实现绘制， 类的层级图如： 

![ios6_text](./images/ios6_text.png)

可以看出 UITextView 是基于 webkit 实现的。 


iOS7 扁平化设计后， 类的层级结构发生了一些变化：
![ios7_text](./images/ios7_text.png)



主要有三个类：

1. NSTextStorage 实际上是 NSMutableAttributedString 的子类。 持有 NSLayoutManager 数组，主要目的是在底层数据发生变化后，通知 layoutmanager 来进行布局变化。
2. NSLayoutManager 负责在一个或者多个 NSTextContainer 对象中进行渲染内容（NSTextStorage）。 把unicode字符转化成字形然后渲染到屏幕上。
	字形和字符不是一对一的关系：
	![n_to_one](./images/n_to_one.png)	 	![one_to_n](./images/one_to_n.png)	 	
3. NSTextContainer 定义文本渲染的区域。可以使用 UIBezierPath 对象，定义排除的区域。


类比 MVC ，三个类的位置如：

![textkit_mvc](./images/textkit_mvc.png)