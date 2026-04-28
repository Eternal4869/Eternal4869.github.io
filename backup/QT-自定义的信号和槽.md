# QT-自定义的信号和槽

> 实现该需求: 下课后，老师饿了，学生请客。

> 使用的关键字: emit

> 使用的函数: connect()

## 具体示例

### 头文件

`student.h`

```C++
#ifndef STUDENT_H
#define STUDENT_H

#include <QObject>

class student : public QObject
{
    Q_OBJECT
public:
    explicit student(QObject *parent = nullptr);

signals:

public slots:
    // 早期版本必须写在此处，5.4版本之后可以写在public或者全局下
    // 返回值为void，需要声明，也需要实现
    // 可以有参数，可以重载
    void treat();
};

#endif // STUDENT_H
```

`teacher.h`

```C++
#ifndef TEACHER_H
#define TEACHER_H

#include <QObject>

class teacher : public QObject
{
    Q_OBJECT
public:
    explicit teacher(QObject *parent = nullptr);

signals:
    // 自定义信号写在 signal 下
    // 返回值为void，只需声明，无需实现
    // 可以有参数，可以重载
    void hungry();
};

#endif // TEACHER_H
```

`mywidget.h`

```C++
#ifndef MYWIDGET_H
#define MYWIDGET_H

#include <QWidget>
#include "teacher.h"
#include "student.h"

class myWidget : public QWidget
{
    Q_OBJECT

public:
    myWidget(QWidget *parent = nullptr);
    ~myWidget();

private:
    teacher *t;
    student *s;
    void classIsOver();
};
#endif // MYWIDGET_H
```

### 源文件

`student.cpp`

```C++
#include "student.h"
#include <QDebug>

student::student(QObject *parent)
    : QObject{parent}
{}

void student::treat()
{
    qDebug() << "treat() ...";
}
```

`teacher.cpp`

```C++
#include "teacher.h"

teacher::teacher(QObject *parent)
    : QObject{parent}
{}
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
    connect(t, &teacher::hungry, s, &student::treat);

    // 调用下课（必须先连接再触发）
    classIsOver();
}

void myWidget::classIsOver()
{
    // 下课函数，触发老师饿了
    emit t->hungry();
}

myWidget::~myWidget() {}
```

## 运行结果

![Image](https://github.com/user-attachments/assets/1e045c0c-b5f5-452a-b57c-6de5f49cf078)
