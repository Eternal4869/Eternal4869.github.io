# QT-自定义信号和槽的重载问题

> 当自定义的信号和槽发生重载时，会引发编译问题。

## 具体示例

头文件

`student.h`

```C++
    // 带菜名的
    void treat(QString foodName);
```

`teacher.h`

```C++
    // 带菜名的
    void hungry(QString foodName);
```

源文件

`student.cpp`

```C++
void student::treat(QString foodName)
{
    // QString -> char*: .toUtf8().data()
    qDebug() << "treat() with" << foodName.toUtf8().data();
}
```

`mywidget.cpp`

```C++
#include "mywidget.h"
#include "teacher.h"
#include "student.h"
#include <QPushButton>

myWidget::myWidget(QWidget *parent)
    : QWidget(parent)
{
    // 指定父窗口为this，将其绑定至对象树上，随窗口一同销毁
    this->t = new teacher(this);
    this->s = new student(this);

    // 老师饿了，学生请客
    void (teacher:: *tSignal) (QString) = &teacher::hungry;
    void (student:: *sSlot) (QString) = &student::treat;
    connect(t, tSignal, s, sSlot);

    // 调用下课（必须先连接再触发）
    classIsOver();
}

void myWidget::classIsOver()
{
    // 下课函数，触发老师饿了
    // emit t->hungry();
    emit t->hungry("food1");
}

myWidget::~myWidget() {}
```

## 运行结果

![Image](https://github.com/user-attachments/assets/391de7cb-3453-4374-b58d-0e56359e6e63)