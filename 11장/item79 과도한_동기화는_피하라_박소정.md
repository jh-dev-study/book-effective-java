# item79. 과도한 동기화는 피하라

### 🙅‍♂️응답 불가와 안전 실패를 피하려면 동기화 메서드나 동기화 블록 안에서는 제어를 절대로 클라이언트에게 양도하면 안된다!

## 동기화(Synchronize)

"여러개의 쓰레드가 한 개의 리소스를 사용하려고 할 때 ,

사용 하려는 쓰레드를 제외한 나머지들이 접근하지 못하게 막는것"

→ Thread-safe

동기화 방법

- 메소드 자체 Synchronized로 선언하는 방법
- 블록으로 객체를 받아 lock을 거는 방법 - syncrhonized(this)

### 멀티스레드 프로세스는 동시성 또는 병렬성으로 실행된다.

동시성 : 하나의 코어에서 여러개의 프로세스가 번갈아 가면서 실행됨

병렬성 : 멀티 코어에서 개별 스레드를 동시에 실행

## 교착 상태

한정된 자원을 여러 곳에서 사용할 때 발생할 수 있는 현상

![item79%20%E1%84%80%E1%85%AA%E1%84%83%E1%85%A9%E1%84%92%E1%85%A1%E1%86%AB%20%E1%84%83%E1%85%A9%E1%86%BC%E1%84%80%E1%85%B5%E1%84%92%E1%85%AA%E1%84%82%E1%85%B3%E1%86%AB%20%E1%84%91%E1%85%B5%E1%84%92%E1%85%A1%E1%84%85%E1%85%A1%20e37b2138b4704c21a9d60101526477fe/deadlock.png](item79%20%E1%84%80%E1%85%AA%E1%84%83%E1%85%A9%E1%84%92%E1%85%A1%E1%86%AB%20%E1%84%83%E1%85%A9%E1%86%BC%E1%84%80%E1%85%B5%E1%84%92%E1%85%AA%E1%84%82%E1%85%B3%E1%86%AB%20%E1%84%91%E1%85%B5%E1%84%92%E1%85%A1%E1%84%85%E1%85%A1%20e37b2138b4704c21a9d60101526477fe/deadlock.png)

출처 : [https://www.geeksforgeeks.org/deadlock-in-java-multithreading/](https://www.geeksforgeeks.org/deadlock-in-java-multithreading/)

💡주의

- 동기화(synchronized) 된 코드 블럭 안에서는 재정의 가능한 메서드를 호출해선 안된다.
- 클라이언트가 넘겨준 함수객체를 호출해서도 안된다.
- 이런 메서드는 동기화도니 클래스 관점에서 외계인 메서드(alien method)라고 칭한다.(무슨일을 할지 모르니, 이 메서드가 예외를 발생시키거나, 교착상태를 만들거나, 데이터를 훼손시킬 수 있다.)

---

case 1) ConcurrentModificationExcption 이 발생하는 경우

```java
public class ObservableSet<E> extends ForwardingSet<E>{
    public ObservableSet(Set<E> set) {
        super(set);
    }

    private final List<SetObserver<E>> observers = new ArrayList<>();

    public void addObserver(SetObserver<E> observer){
        synchronized (observers){
            observers.add(observer);
        }
    }
		// 외계인 메서드
    private void notifyElementAdded(E element){
        synchronized (observers){
            for(SetObserver observer: observers){
                observer.added(this,element);
            }
        }
    }

    public boolean removeObserver(SetObserver<E> observer){
        synchronized (observers){
            return observers.remove(observer);
        }
    }

    @Override
    public boolean add(E element) {
        boolean added = super.add(element);
        if(added)notifyElementAdded(element);
        return added;
    }

    public static void main(String[] args) {
        ObservableSet<Integer> set = new ObservableSet<>(new HashSet<>());

//        set.addObserver(((set1, element) -> System.out.println(element)));
//
//        for (int i = 0; i < 100; i++) {
//            set.add(i);
//        }// 100까지 출력

				// 23 일 때 , 지우는 조건 추가
        set.addObserver(new SetObserver<Integer>() {
            @Override
            public void added(ObservableSet<Integer> set, Integer element) {
                System.out.println(element);
                if (element == 23)
                    set.removeObserver(this);
            }
        });
        for (int i = 0; i < 100; i++) {
            set.add(i);
        }
    }

}
```

```java
@FunctionalInterface
public interface SetObserver<E> {

    void added(ObservableSet<E> set, E element);
}
```

case2) 쓸데 없이 백그라운드 스레드를 사용하는 관찰자(교착 상태)

```java
set.addObserver(new SetObserver<>() {
    public void added(ObservableSet<Integer> s, Integer e) {
        System.out.println(e);
        if(e == 23) {
            ExecutorService exec = Executors.newSingleThreadExecutor();
            try {
                exec.submit(() -> s.removeObserver(this)).get(); //여기서 lock
               													 //하지만 메인 스레드 작업을 기다림 
            } catch(ExecutionException | InterruptedException ex) {
                throw new AssertionError(ex);
            } finally {
                exec.shutdown();
            }
        }
    }
}); 
```

### 락의 재진입

락에 대한 자세한 설명 →[https://parkcheolu.tistory.com/2](https://parkcheolu.tistory.com/24)4

자바의 고유 락은 재진입 가능하다. 재진입 가능하다는 것은 락의 획득이 호출 단위가 아닌 스레드 단위로 일어난다는 것을 의미한다. 이미 락을 획득한 스레드는 같은 락을 얻기 위해 대기할 필요 없다.

```java
public class Reentrancy {
  public synchronized void a() {
    System.out.println("a");
    // b가 synchronized로 선언되어 있지만 a진입시 이미 락을 획득하였으므로,
    // b를 호출할 수 있다.
    b();
  }
  public synchronized void b() {
    System.out.println("b");
  }
  public static void main(String[] args) {
    new Reentrancy().a();
  }
}
```

만약 재진입이 불가능하다면? → 교착 상태에 빠질것

### 문제점

안전실패 (데이터 변모)의 문제

교착 상태는 막을 수 있지만, 같은 락에서 보호 받고 있는 데이터에 대해 개념적으로 관련이 없는 다른 작업이 진행 중이어도 락 획득을 성공한다.

그러므로 데이터의 훼손가능성이 증가한다.

락의 재진입 문제 해결 방법 -1)

```java
private void notifyElementAdded(E element) {
    List<SetObserver<E>> snapshot = null;
    synchronized(observers) {
        snapshot = new ArrayList<>(observers);
    }
    for (SetObserver<E> observer : snapshot) {
        observer.added(this, element); //동기화 블록 바깥으로
    }
}
```

락의 재진입 문제 해결 방법 2)

자바의 동시성 컬렉션 라이브러리 CopyOnWriteArrayList 를 활용

내부를 변경하는 작업은 항상 복사본을 만들어 수행 

내부의 배열은 수정되지 않아서 순회할 때 락이 필요 없다.

```java
private final List<SetObserser<E>> observers = new CopyOnWriteArrayList<>();

public void notifyElementAdded(E element) {
    for (SetObserver<E> observer : observers) {
         observers.added(this, element);
    }
}
```

---

## 동기화의 성능

자바의 동기화 비용은 빠르게 낮아져 왔지만, 과도한 동기화를 피하는일은 오히려 과거 어느 때보다 중요하다.

멀티코어가 일반화된 오늘날 과도한 동기화가 초래하는 진짜 비용은 락을 얻는데 드는 CPU 시간이 아니다.

서로 스레드끼리 경쟁하는 Race Condition에 낭비가 발생한다.

- 병렬로 실행할 기회를 잃는다.
- 모든 코어가 메모리를 일관되게 보기위한 지연시간이 진짜 비용
- 가상머신의 코드최적화를 제한하는 점도 숨은 비용

---

## 정리

- 기본 규칙은 동기화 영역에서 가능한 한 일을 적게하는 것
- 오래 걸리는 작업이라면 동기화 영역 밖으로 옮기는 방법을 찾아보자.
- 여러 스레드가 호출할 가능성이 있는 메서드가 정적 필드를 수정한다면 그 필드를 사용하기 전에 반드시 동기화 해야 한다.
- 교착상태와 데이터 훼손을 피하려면 동기화 영역 안에서 외계인 메서드를 절대 호출하지 말자
- 동기화 영역 안에서 작업은 최소한으로 줄이자