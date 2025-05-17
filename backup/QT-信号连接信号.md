# QT-信号连接信号

> 窗口有一个按钮，按钮点击（信号）触发下课函数（信号），下课函数触发老师饿了，老师饿了，学生请客。

## 具体示例

`myWidget.cpp`

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

    // 按钮触发下课
    QPushButton *classOverBtn = new QPushButton("CLOSE", this);
    connect(classOverBtn, &QPushButton::clicked, this, &myWidget::classIsOver);
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

![Image](https://github.com/user-attachments/assets/c2c7e58d-c471-449e-a572-f9f567f02d2e)

按钮点击后。

![Image](https://github.com/user-attachments/assets/217b3eda-fbe4-4e7c-8f44-c214af744d5c)