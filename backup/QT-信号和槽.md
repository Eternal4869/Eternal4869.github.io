# QT-信号与槽

> connect(信号的发送者, 信号, 信号的接收者, 信号的处理-槽函数);

信号槽的优点:
- 松散耦合: 信号发送端和接收端本身并无关联，通过 `connect()` 函数将两端耦合在一起。

## 具体示例

`mywidget.cpp`

```C++
#include "mywidget.h"
#include <QPushButton>

myWidget::myWidget(QWidget *parent)
    : QWidget(parent)
{
    QPushButton *button = new QPushButton;
    button->setParent(this);
    button->setText("Close");
    setFixedSize(800, 500);
    setWindowTitle(" ");
    setWindowFlags(Qt::FramelessWindowHint);
    this->move(0,0);
    // 点击按钮关闭窗口
    // connect(信号发送者,信号,信号的接收者,信号的处理-槽函数);
    // 参数1 信号的发送者
    // 参数2 发送的信号（函数的地址）
    // 参数3 信号的接收者
    // 参数4 处理的槽函数
    connect(button, &QPushButton::clicked, this, &QWidget::close);
}

myWidget::~myWidget() {}
```