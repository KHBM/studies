# 4. 두 다형성의 만남

## 제네릭 클래스와 상속
- 제네릭 클래스를 상속과 함께 사용하는 방법
	- 제네릭 클래스가 있을 때 타입들 사이의  서브타입 관계는 어떻게 될까.
		- class ArrayList<T> { T get(Int idx) { ...}}
		- class LinkedList<T> { T get(Int idx) {...}}
	- 이들을 다루기 위한 추상 클래스를 하나 만드는 것이 좋다.
		- abstract class List<T> { T get(Int idx); ... }
	- 위 2개 리스트는
 		- class ArrayList<T> extends List<T> { T get(Int idx) { ...}}
		- class LinkedList<T> extends List<T> { T get(Int idx) {...}}
	- 가 된다.
		- 관계 정의에서 추상 클래스에서도 T 가 나타나는 이유다.
## 타입 매개변수 제한
- 매개변수가 특정한 능력을 지녀야 하는 경우에 매개변수를 특정 능력을 지닌 타입 하위로 제한하는 기능이다.
```
T elder<T <: Person>(T p, T q) {
	return (p.age >= q.age) ? p : q;
}
```
- 매개변수가 Person 의 상한 제약을 추가한다. 따라서 T 매개변수는 age 필드를 소유하는 것이 확정적이다.
- 상한 제약은 T 의 최대 타입은 Person 이라고 한정하는 것이다.
- 언어가 구조를 드러내는 타입을 제공한다면 아래의 경우로도 작성이 가능하다
```
T elder<T <: { Int age; }>(T p, T q) {
	return (p.age >= q.age) ? p : q;
}
```
## 재귀적 타입 매개변수 제한
- 타입 매개변수가 자기 자신을 제한하는 데 사용될 수 있다.
- 이를 재귀적 타입 매개변수 제한이라 부른다.
```
Void sort<T <: Comparable<T>> (List<T> list) {
 ...
}
```
- T 가 반드시 Comparable<T> 의 서브타입이어야 한다는 뜻이다.
## 가변성
- 제네릭 타입 사이의 서브타입 관계를 추가로 정의하는 기능이다.
- List<Person> 과 List<Student> 사이의 관계를 정의한다.
- B 가 A 의 서브타입일 때, 읽기만 가능한 List1 에 대해 List1<B> 는 List1<A> 의 서브타입이다. 그러나 원소 추가가 가능한 List2 에 대해선 성립하지 않는다.
- 이것을 확인하면 제네릭 타입과 타입 인자의 서브타입 관계가 보전되지만(List1) 그렇지 않은 경우(List2)도 있다는 내용이다.
- 첫 번째 가변성은 보전하는 것으로, 공변이라 부른다.
- 두 번째 가변성은 무시하는 것으로, 불변이라 부른다.
- 이 외, 함수 타입에 대해 T => T 고찰할 때
	- T => S 는 두 개의 매개변수를 가진 제네릭 타입이다.
	- 함수 타입은 매개변수 타입의 서브타입 관계를 뒤집고 결과 타입의 서브타입 관계를 유지한다고 했다.
		- 여기서 함수 타입의 매개변수 타입에서 새로운 가변성이 나타난다.
		- 뒤집힌다.
		- 이를 반변이라 부른다.
- 최종 결론에 도달하면
	- 제네릭 타입을 G, 매개변수를 T 라 할 때
		- G 가 T 를 출력에만 사용하면 공변
		- 입력에만 사용하면 반변
		- 출력과 입력 모두에 사용하면 불변이다.
### 파라미터 정의 시 가변성 지정하기
- 매개변수 지정 시 아무 것도 지정하지 않으면 기본적으로 불변이다.
- out 은 출력에만 사용하는 경우로, 공변이다.
```
abstract class ReadOnlyList<out T> {
	Int length();
	T get(Int idx);
}
```
- in 을 붙이면 반변이다.
```
abstract class Map<in T, S> {
	Int size();
	S get(T t);
	Void add(T t, S s);
}
```
- B가 A의 서브타입일 때, Map<A, C> 가 Map<B, C> 의 서브타입니다.

### 사용할 때 가변성 지정하기
- 제네릭 타입을 사용할 때 가변성을 지정하는 경우, 제네릭 타입을 정의할 때는 가변성을 지정할 수 없다.
- 제네릭 타입을 사용할 때 가변성을 불변 대신 공변이나 반변으로 지정하려면 새로운 종류의 타입을 사용해야 한다.
	- 새로운 타입들은 타입 인자 앞에 out 또는 in 을 붙여서 만들 수 있다.
	- List<out Person>, List<in Student>, Map<Person, out Person> Map<out Person, in Person> 등이 가능하다.
```
List<out Person> people = ...;
people.length();
peopole.get(...);
```
- 이렇게 하면 출력 기능만 허용된다. 입력 기능 메서드 사용시 컴파일 에러가 난다.
- B가 A의 서브타입일 때, List<out B> 는 List<out A>의 서브타입이다.
- 이 관계는 연쇄 작용이 발생한다.
- List<B>가 List<out B>의 서브타입이므로, List<B>는 List<out A> 의 서브타입니다.
```
List<in Person> people = ...;
people.length();
people.add(...);
```
- 입력 기능만 허용된다.
- 출력 기능 사용시 컴파일 에러가 난다.
- B가 A의 서브타입일 때 List<in A>가 List<in B> 의 서브타입이다. 연쇄 작용에 의해
- List<A>가 List<in B> 의 서브타입이다.
