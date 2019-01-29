# 设计模式引言

要上班了(WTMSB)，设计模式还没系统学过，赶紧补习一波，万一看不懂人家骚气的代码，岂不是很滑稽。

## 啥是设计模式

面向对象系统中存在很多的类以及对象之间的通信行为，这部分内容存在高度的重复性以及可扩展的需求，
这部分内容八成可以由某种设计模式来描述。
因此设计模式主要解决的是面向对象软件的可复用，不应该生搬硬套，奉为银弹。
GoF所著的《设计模式》中总结了23种常用的设计模式基础，重点要关注这些模式主要解决的问题，模式的模板以及它所带来的优点。

## 23种基础设计模式

GoF将这些设计模式以其目的分为三类：创建型、结构型、行为型：
创建型主要用于创建对象、结构型主要用于描述类与对象的组合、行为型主要描述用于描述类与对象的交互。
此外又根据实现分成了类模式以及对象模式：类模式用于描述类以及子类的关系，在C++里面，这种关系是编译时就确立的，属于静态关系；
对象模式用于处理对象间关系，具备动态性，可以在运行时变化。

|          | 创建型           | 结构型  | 行为型 |
| -------- | ---------------- | ------- | ------ |
| 类模式   | Factory Method   | Adapter(class) | Interpreter<br>Template Method |
| 对象模式 | Abstract Factory<br>Builder<br>Prototype | Adapter(obj)<br>Bridge<br>Composite<br>Decorator<br>Facade<br>Flyweight<br>Proxy    |Chain of Responsibility<br>Command<br>Iterator<br>Mediator<br>Memento<br>Observer<br>State<br>Strategy<br>Visitor|
