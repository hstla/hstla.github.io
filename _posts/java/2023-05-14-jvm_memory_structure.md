---
title: "JVM 메모리 구조"
categories: [java]
date: 2023-05-14
toc: true
toc_label: CATEGORY
toc_sticky: true
tags: [java, JVM]
---

JVM 메모리 구조를 알아보기 전에 JVM에 대해 알아보자!

# JVM이란?

JVM(Java Virtual Machine) 자바를 실행시키기위한 가상환경이다. 자바코드(.java)는 자바 컴파일러 javac에 의해 바이트 코드(byte code)로 변환된다. 

JVM이 중간에서 코드를 해석해 주기때문에 자바는 운영체제에 상관없이 실행시킬 수 있고 이를 플랫폼 독립성(platform independence)이라고 한다.

클래스 파일로 변경된 바이트 코드는 클래스 로더에 의해 JVM 내부로 로딩된다.

<p align = "center"><img src='/../assets/images/posts/2023-05-14/1.svg' width="200"/></p>


또한 JVM은 가비지 컬렉터(Garbage Collector)를 통해 더 이상 사용되지 않는 메모리를 해제하여, 메모리 누수를 방지하고 메모리 사용량을 최적화한다. 

c++에서는 new연산자 이후 delete연산자로 메모리해제를 명령해야하지만 java는 new연산자 이후 delete를 사용하지 않아도 Garbage Collerctor가 사용하지 않는 변수나 메서드를 해제한다.

**c++ new, delete연산자 사용**

```cpp
int main(void)
{
    char* str = new char[20];

    strcpy(str, "I am so happy.");
    cout << str << endl;

    delete[]str;

    return 0;
}
```

**자바 new 연산자 사용**

```java
public static void main(String[] args) {
		List<Integer> arr = new ArrayList<>();		
		arr.add(1);
		arr.add(2);
		arr.add(3);
		for (int i : arr) System.out.print(1+" ");
}
```

## JVM의 메모리 구조

JVM은 자바 프로그램을 실행하기 위해 따로 메모리 영역이 존재하는데 이를 **Runtime Data Area**라 한다.

Runtime Data Area는 5개의 영역으로 나뉜다.

1. Method Area
2. Heep
3. Stack
4. PC Register
5. Native Method Stack


<p align = "center"><img src='/../assets/images/posts/2023-05-14/2.svg' width="500"/></p>


**1. Method Area**

JVM이 시작될 때 생성되며, 모든 스레드가 공유되는 영역으로 JVM이 실행되는 모든 클래스에 대한 정보인 클래스 파일의 바이트 코드, 상수, 클래스 변수(static), 메소드 정보를 저장한다.

모든 스레드가 공유되는 영역이기때문에 스레드 안전성을 보장한다.

**2. Heep**

동적으로 할당되는 객체들이 저장되는 메모리 영역이다. 

Heep메모리도 Method Area와 마찬가지로 JVM이 시작될 때 생성되며, 모든 스레드가 공유하는 영역이다. JVM이 실행되는 동안 객체가 new 연산자에 의해 생성되거나 Garbage Collection에 의해 삭제되는 영역이다.

**3. JVM Stack**

각 스레드마다 생성되는 영역으로 메서드의 실행에 필요한 데이터와 메서드 호출 및 반환 값, 지역 변수 등을 저장한다. 

자료구조의 스택 영역처럼 LIFO(Last-In-First-Out) 구조를 가지며, 스택 프레임의 생성과 소멸은 메소드 호출과 반환에 의해 결정된다.

**4. PC Register**

Program Counter Register의 약어로, **현재 실행 중**인 명령어의 주소를 가리키는 레지스터이다. 

**5. Native Method Stack**

자바 코드가 아닌 네이티브 코드(C, C++ 등)을 실행하기 위한 스택이다. 네이티브 메소드 호출 시 생성되며, 해당 메소드가 실행될 때까지 유지된다.

만약 밑의 코드가 실행됐을 떄

```cpp
public static void main(String[] args) {
		Person p = new Person();
}
```

먼저 JVM이 Person() 객체를 저장하기 위해 Heep 영역에 메모리를 할당한다. 

그 후에 JVM Stack 영역에 p 변수가 생성되어 Heep 영역에 생성된 Person 객체를 참조한다.

이후 p 변수를 통해 Person 객체에 접근하여 데이터를 수정할 수 있고 사용이 종료되면 Garbage Collection에 의해 삭제된다.