---
title: "Comparable와 Comparator"
categories:
- JAVA
date: 2023-04-25
toc: true
toc_label: CATEGORY
toc_sticky: true
---

자바에서 객체를 정렬하기 위해 `Arrays.sort()`나 `Collections.sort()`를 사용한다. 

하지만 정렬에 조건을 추가하고 싶을 때 `Comparable`과 `Comparator`  인터페이스를 사용해야 한다.

## Comparable

Comparable 인터페이스는 객체 스스로가 정렬에 필요한 기준을 설정할 수 있다.

`Comparable` 인터페이스는 클래스를 선언할때 상속하여 `CompareTo()`함수에 정렬 조건을 정의할 수 있다.

```java
public class Person implements Comparable<Person> {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
		@Overriede
    public int compareTo(Person other) {
        return this.age - other.age;  // 나이로 오름차순
    }

    public static void main(String[] args) {
        List<Person> people = new ArrayList<>();
        people.add(new Person("A", 25));
        people.add(new Person("B", 30));
        people.add(new Person("C", 20));
        Collections.sort(people);
        System.out.println(people); // [C(20), A(25), B(30)]
    }
```

### `CompareTo`가 오름차순인 이유

`CompareTo`를 사용한 정렬은 `Collections.sort()`, `Arrays.sort()`와 같이 오름차순으로 정렬된다.

오름차순(1,2,3,4, …)를 이용하여 return 값이 음수이면 현재 객체가 다른 객체보다 작다고 인식되고, return 값이 양수이면 다른 객체보다 크다고 인식되어 정렬된다.

예를들어 this.age = 1 이고 other.age = 3이면 return값이 -2음수이기 때문에 [1,3]순으로 정렬된다.

나이로 내림차순(4,3,2,…)으로 정렬하고 싶다면 this와 other의 자리를 바꾸면 된다.

```java
public int compareTo(Person other) {
return other.age - this.age;
}
```

위의 예처럼 this.age = 1, other = 3일 때 return값이 2 양수이기 때문에 [3,1]로 정렬된다.

## Comparator

Comparator 인터페이스는 객체 정렬기준을 **별도로** 정의할 수 있다.

 `Comparator` 클래스 **외부에서** 선언하고 `compare`함수에 정렬 조건을 구현한다.  

```java
public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }

    public static void main(String[] args) {
        List<Person> people = new ArrayList<>();
        people.add(new Person("A", 25));
        people.add(new Person("B", 30));
        people.add(new Person("C", 20));
        Comparator<Person> comparator = new Comparator<Person>() {
           public int compare(Person p1, Person p2) {
					    return p1.getAge() - p2.getAge();
}
        };
				Collections.sort(people, comparator);
	      System.out.println(people); // [C(20), A(25), B(30)]
}
```

### `compare`가 오름차순인 이유

`compare`도 `compareTo()`와 같이 기본적으로 오름차순으로 정렬한다.

return값이 음수일 때 p1이 p2보다 작고, 양수일 때 p1이 p2보다 크다고 인식하여 정렬한다.

p1.age = 3  p2.age = 2일 때 return 값은 3-2 = 1 양수이므로 p1과 p2의 위치가 **바뀌어서** [p2, p1]으로 정렬된다.

반대로 p1.age = 2, p2.age = 3 일때 return 값은 2-3 = -1 음수이므로 위치가 **바뀌지 않고** [p1, p2]오름차순으로 정렬된다.

## 정리

`Comparable` 인터페이스는 클래스 내부에 정렬 기준을 정의할 때 사용되며, 

`Comparator` 인터페이스는 클래스의 정렬 기준을 변경하거나, 다른 기준으로 객체를 정렬하고 싶을 때 사용한다. 

Comparable 인터페이스를 구현하여 기존의 정렬 기준이 있지만, 다른 기준으로 정렬하고 싶을 때 `Comparator` 선언하여 정렬 기준을 변경할 수 있다.