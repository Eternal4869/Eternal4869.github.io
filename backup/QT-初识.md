# QT项目下的文件

>QT creator创建的 `.pro` 文件并未列出

1. 头文件 
 
- `mywidget.h`

```C++
#ifndef MYWIDGET_H
#define MYWIDGET_H

#include <QWidget>

class myWidget : public QWidget
{
    Q_OBJECT

public:
    myWidget(QWidget *parent = nullptr);
    ~myWidget();
};
#endif // MYWIDGET_H
```

2. 源文件

- `main.cpp`

```C++
#include "mywidget.h"

#include <QApplication>

int main(int argc, char *argv[])
{
    QApplication a(argc, argv);
    // window
    myWidget w;
    // make window shown
    w.show();
    // make a into cycle
    return a.exec();
}
```

- `mywidget.cpp`

```C++
#include "mywidget.h"
#include <QPushButton>

myWidget::myWidget(QWidget *parent)
    : QWidget(parent)
{
    // 创建按钮
    QPushButton *button = new QPushButton;
    // 设置父窗口
    button->setParent(this);
    // 设置文本
    button->setText("Hello");
    QPushButton *button2 = new QPushButton("Second", this);
    // 移动按钮
    button2->move(400, 300);
    // 设置窗口大小（mywidget的大小）
    // resize(500, 400);
    // 设置窗口为固定大小（不可伸缩）
    setFixedSize(800, 500);
    // 设置窗口标题
    setWindowTitle(" ");
    // 设置窗口为无边框样式
    setWindowFlags(Qt::FramelessWindowHint);
    // 移动窗口至 `0, 0` 的位置
    this->move(0,0);
}

myWidget::~myWidget() {}
```