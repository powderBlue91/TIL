# 불변객체란?
불변객체란 객체 인스턴스의 상태를 변경할 수 없는 객체를 의미한다.

# 불변객체를 보장하는 방법들
- 필드를 final로 선언
- 방어적 복사
- Collection.unmodifiableList 메소드 사용

위 방법들이 있다. 하지만 이를 남용하기보단 주의사항과 어떤 경우에 사용해야하는지 알고있는것이 중요하다.

# Case
### Case1 | 객체 필드가 primitive type일 경우
이경우엔 필드에 final 키워드만 붙여주면 해당 객체는 불변객체를 보장한다. 
final 필드를 사용하기때문에 setter 메소드 또한 존재할 수 없다.

```
public class Lotto {
    private final int number;
}
```

### Case2 | 객체 필드가 reference type일 경우
이경우엔 참조 필드에 final 키워드를 쓰더라도 참조 필드의 get, set 메소드에 의해 
참조 필드의 상태값을 수정할 수 있다. 그러므로 참조 필드 내부의 상태값도 final을 붙여서 불변 객체로 만들어야한다.

```
public class Lotto {
    private final Number number;
}
```

### Case3 | 객체 필드가 Collection일 경우
이 경우에 Collection에 final 키워드를 쓰고 다음과 같이 생성자에서 다른 Collection에 의해 초기화 된다고 가정해보자.

```
public class Bucket {

    private final List<Data> list;
    
    public Bucket(List<Data> tmpList) {
        this.list = tmpList;
    }
}

```

위와 같이 사용한다면 객체의 Collection은 얕은복사로 주소의 참조를 복사하게 되고, 만약 매개변수(tmpList) Collection 원소들을 add, remove 하게 되면
객체의 Collection(list)의 원소들도 자동으로 add, remove하게 된다. 이는 불변성을 깨뜨린다.
그래서 이때는 생성자에서 ***방어적 복사로 필드를 초기화***해줘야한다. 그러면 기존 Collection와의 참조를 끊고 복사되기 때문에 
외부에서 Collection에서 수정하더라도 내부 필드는 변경되지 않는다 ***(* 얕은 복사, 방어적 복사, 깊은 복사의 차이점을 알아야한다)***

```
public class Bucket {

    private final List<Data> list;
    
    public Bucket(List<Data> list) {
        this.list = new ArrayList<>(list);
    }
}
```

그런데 여기서 Collection를 반환하는 getter 메서드가 있다면 어떻게 될까? 

```
public class Bucket {

    private int data1;
    private final List<Data> list;
    public Bucket(List<Data> list) {
        this.list = new ArrayList<>(list);
    }
    
    public List<Data> getBucket() {
        return this.list;
    }
}
```

반환해서 add, remove 등의 행위를 한다면 이또한 객체 불변성을 깨뜨린다.

이런 경우에도 방어적 복사를 사용해서 반환하면 객체의 불변성을 지킬 수 있다.
또는 Collections.unmodifiableList()로 반환하면 add, remove같은 메소드를 쓰면 예외를 반환하기때문에 이또한 불변성을 지킬 수 있는 방법이다.

```
public List<Data> getBucket() {
        return new ArrayList<>(this.list);
    }
```

```
public List<Data> getBucket() {
        return Collections.unmodifiableList(this.list);
    }
```
  
# 정리
정리하자면 불변객체를 만드는 방법은
1. 모든 필드를 final로 만든다.
2. 필드가 레퍼런스 타입이라면 레퍼런스 타입 내부 필드도 final 키워드를 써야한다.
3. 생성자나 getter 메서드에 방어적 복사를 사용한다. (getter 메서드에는 Collections.unmodifiableList()도 가능)

https://steady-coding.tistory.com/559
