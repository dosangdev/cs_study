# 12장. 프로세스 동기화

## 12-1. 동기화란
#### 프로세스 동기화의 의미
> 프로세스들 사이의 수행 시기를 맞추는 것. 실행 순서 제어, 상호 배제가 있다.

##### * 프로세스뿐만 아니라 스레드도 동기화의 대상이다. 

##### 1. 실행 순서 제어를 위한 동기화<br>
   > 올바른 순서대로 실행하기
  - Writer 프로세스가 book.txt를 생성하기 전에 Reader 프로세스가 이것을 읽는 것은 잘못된 실행 순서이다.
   
##### 2. 상호 배제를 위한 동기화<br>
   > 한 자원에 동시에 접근하지 않도록 하기
   
   - 생산자와 소비자 문제
```cpp
#include <iostream>
#include <queue>
#include <thread>

void produce();
void consume();

int sum = 0; // 총합 : 공유 자원

int main() {

    std::cout << "초기 합계: " <<  sum << std::endl;
    std::thread producer(produce); // producer 프로세스가 produce 함수를 수행하도록
    std::thread consumer(consume); // consumer 프로세스가 consume 함수를 수행하도록

    producer.join(); // 스레드 실행
    consumer.join(); // 스레드 실행
    
    std::cout << "producer, consumer 스레드 실행 이후 합계: " <<  sum << std::endl;
    
    return 0;
}

void produce() {
    for(int i = 0; i < 100000; i++) { // 10만 번
        sum++; // 합계를 1만큼 증가
    }
}

void consume() {
    for(int i = 0; i < 100000; i++) { // 10만 번
        sum--; // 합계를 1만큼 감소
    }
}
```
```
초기 합계: 0
producer, consumer 스레드 실행 이후 합계: -32689 // 실행할 때마다 값이 바뀜
```
스레드 실행 후 합계가 0이 아닐뿐더러, 실행할 때 마다 값이 변한다. 생산자와 소비자가 동시에 총합을 수정했기 때문에 이런 일이 발생했다. 

이런 일이 발생하지 않도록, **상호 배제**가 될 수 있게 동기화를 해주어야 한다. 한 쪽의 작업이 끝나고 나서, 다른 쪽의 작업이 시작되게 해야 한다.

#### 공유 자원, 임계 구역
> 공유 자원 : 여러 프로세스가 공유하는 자원
> 임계 구역(critical section) : 공유 자원 중, 동시에 실행하면 안되는 자원에 접근하는 코드 영역

- 임계 구역의 코드는 반드시 한번에 한 프로세스만 실행할 수 있어야 한다.
- 레이스 컨디션 : 임계 구역에서 둘 이상의 프로세스가 실행되어 문제가 발생하는 경우. 데이터의 일관성*이 깨진다. ex) 생산자와 소비자 문제

##### * 데이터의 일관성 : 같은 시간에 조회한 데이터는 같은 내용이어야 한다. 즉 같은 시점에서는, 어떤 곳에서든지 데이터의 값이 항상 동일해야 한다는 것.

#### 상호 배제를 위한 동기화의 세 가지 원칙
1. 상호 배제 : 임계 구역에 진입하는 프로세스는 한 번에 하나만 가능.
2. 진행 : 임계 구역에 진입한 프로세스가 없을 때는, 임계 구역에 진입하고자 하는 프로세스는 들어갈 수 있다.
3. 유한 대기 : 임계 구역에 들어가기 위해 무한정 대기가 발생하면 안된다.


## 12-2.동기화 기법
#### 뮤텍스 락
> 뮤텍스 락 (**MUT**ual **EX**clusive **LOCK**) : 공유 자원에 걸어놓은 자물쇠

- 구성 요소 : 뮤텍스 락, acquire 함수, release 함수

> acquire 함수 : 락이 잠겨있다면 락이 열릴 때까지 반복적으로 확인하고, 락이 열려있다면 본인이 임계 구역에 진입한 뒤 락을 잠그는 함수<br>
> release 함수 : 잠긴 락을 해제하는 함수
- 상호 배제 과정
  - 락을 획득할 수 없다면 무작정 기다리면서 락이 열렸는지 반복적으로 확인(바쁜 대기)
  - 락을 획득할 수 있다면 임계 구역을 잠근 뒤 임계 구역에서 작업
  - 작업이 끝나면 임계 구역의 잠금을 해제
 
```c
acquire();
// 임계 구역
release();
```
c 예시 코드
```c
 01 #include <stdio.h>
 02 #include <pthread.h>
 03
 04 #define NUM_THREADS 4
 05 pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
 06
 07 int shared = 0; // 공유 자원
 08
 09 void *foo()
 10 {
 11     pthread_mutex_lock(&mutex); // acquire() : c에서 제공하는 락 잠금 함수 - 이미 잠겨있다면 계속 확인
 12     for (int i = 0; i < 10000; ++i) { // 임계 구역
 13         shared += 1;
 14     }
 15     pthread_mutex_unlock(&mutex); // release() : 락 해제
 16     return NULL;
 17 }
 18
 19 int main()
 20 {
 21     pthread_t threads[NUM_THREADS]; // NUM_THREADS만큼 스레드 생성
 22
 23     for (int i = 0; i < NUM_THREADS; ++i) {
 24         pthread_create(&threads[i], NULL, foo, NULL); // foo라는 작업에 스레드를 할당
 25     }
 26
 27     for (int i = 0; i < NUM_THREADS; ++i) {
 28         pthread_join(threads[i], NULL); // 스레드 실행
 29     }
 30
 31     printf("final result is %d\n", shared);
 32
 33     return 0;
 34 }
```

#### 세마포
> 공유 자원이 여러 개인 버전의 뮤텍스 락

- 구성 요소 : 사용 가능한 공유 자원의 개수를 뜻하는 전역 변수 S, wait 함수, signal 함수

1) 상호 배제 과정
  - S가 2일 경우
  - 프로세스 A : S를 1 감소시키고, S가 0이므로 임계 구역 진입하여 작업.
  - 프로세스 B : S를 1 감소시키고, S가 -1이므로 대기 큐에 삽입되고 대기 상태로 전환.
  - 프로세스 A : 작업 완료 후 임계 구역에서 빠져나오면서 S를 1 증가시키고 대기 큐에 있는 프로세스 B를 대기 큐에서 제거한 뒤 준비 상태로 전환시킴.
  - 프로세스 B : S를 1 감소시키고, S가 0이므로 임계 구역 진입.
    
```c
wait();
// 임계 구역
signal();
```

```c
wait() {
  S--;
  if ( S < 0) { // S가 0보다 작으면
    /* 해당 프로세스를 대기 큐에 삽입하고, 대기 상태 전환 */
  };
```
```c
signal() {
  S++;
  if( S <= 0 ) {
    /* 대기 큐에서 대기 중인 프로세스를 큐에서 제거하고 준비 상태로 전환시킴 */
  };
}
```
Python 예시 코드
```python
  1 from threading import Thread, Semaphore
  2
  3 num = 0 # 공유 자원
  4 sem = Semaphore(1) # 전역 변수 S
  5
  6 def foo(sem):
  7     global num
  8     sem.acquire() # python에서의 wait 함수
  9     for _ in range(100000): # 임계 구역
 10         num += 1
 11     sem.release() # python에서의 signal 함수
 12
 13 if __name__ == '__main__':
 14     t1 = Thread(target=foo, args=(sem,))
 15     t2 = Thread(target=foo, args=(sem,))
 16     t1.start()
 17     t2.start()
 18     t1.join()
 19     t2.join()
 20     print(num)
```

2) 순서 제어 과정
  - S = 0 설정
  - 먼저 실행할 프로세스 뒤어 signal 함수
  - 나중에 실행할 프로세스 앞에 wait 함수
  
<img width="581" alt="image" src="https://github.com/Minnie5382/devduck-cs-study/assets/97179789/9bf6d627-9b64-40f8-8cf1-cf94271d22a3">

이렇게 해주면, P1이 먼저 도착할 경우 P1이 임계 구역에 먼저 진입하게 된다. P2가 먼저 도착해도 S가 -1이므로 대기 상태로 전환되고, P1이 먼저 임계 구역이 진입한다. P1이 임계 구역에 빠져나와 signal()을 호출하면 P2의 대기 상태가 해제되어 P2가 이용할 수 있다.

#### 모니터
- 세마포는 매번 임계구역 앞뒤로 함수를 명시해주어야 하므로, 번거롭고 실수할 여지가 있다. 이를 보완한 것이 모니터이다.
- 모니터는 OS에서 지원하는 것이 아니라, 프로그래밍 언어에서 지원한다. (Java)
- 구성 요소 : 모니터(공유자원, 인터페이스), 대기 큐, 조건 변수, 조건 변수 대기 큐

1) 상호 배제 과정
  - 모니터 안에는 공유 자원과, 해당 공유 자원에 접근할 수 있는 통로인 인터페이스가 있다. 프로세스는 인터페이스를 통해서만 공유 자원에 접근할 수 있다.
  - 모니터 안에는 한번에 한 프로세스만 들어올 수 있다.
  - 모니터에 진입하고자 하는 프로세스들은 대기 큐에서 기다리고, 큐에 줄서있는 순서대로 모니터에 진입한다.
 ```java
  1 public class BoundedBuffer<E> 
  2 {
  3     private static final int BUFFER_SIZE = 5;
  4     private E[] buffer;  // 공유 자원
  5
  6     public BoundedBuffer() {
  7         count = 0;
  8         in = 0;
  9         out = 0;
 10         buffer = (E[]) new Object[BUFFER_SIZE];
 11     }
 12
 13     /* 생산자가 호출하는 코드 */
 14     public syncronized void insert(E item) { // 인터페이스
 15         while (count == BUFFER_SIZE) { // isFull
 16             try {
 17                 wait();
 18             }
 19             catch (InterruptedException ie) {}
 20         }
 21         // 임계 구역
 22         buffer[in] = item;
 23         in = (in + 1) % BUFFER_SIZE;
 24         count++;
 25
 26         notify();
 27     }
 28
 29     /* 소비자가 호출하는 코드 */
 30     public syncronized E remove() { // 인터페이스
 31         E item;
 32
 33         while (count == 0) {
 34             try {
 35                 wait();
 36             }
 37             catch (InterruptedException ie){}
 38         }
 39
 40         item = buffer[out];
 41         out = (out + 1) % BUFFER_SIZE;
 42         count--;
 43         notify();
 44
 45         return item;
 46     }
 47 }
```
java에서 모니터의 인터페이스에 해당하는 메서드에 ```syncronized``` 키워드를 붙이면, 이 키워드가 불은 메서드는 반드시 한번에 하나의 프로세스만 실행할 수 있다. 이렇게 함으로써 모니터를 구현할 수 있다.

<br>

2) 순서 제어 과정
  - 대기 큐에 있던 프로세스 A가 모니터에 진입한다. 근데 이 프로세스 A는 프로세스 B의 뒤에 실행되어야 한다.
  - 프로세스 A가 조건 변수 x에 대해 x.wait()을 호출하고, 조건 변수 x에 대한 대기 큐에 삽입된다.
  - 프로세스 A가 조건 변수 x의 대기 큐에서 대기하면, 대기 큐에 있던 프로세스 B가 모니터에 진입할 수 있게 된다.
  - 프로세스 B가 모니터에 진입하여 공유 자원에 접근한다.
  - 프로세스 B가 작업을 마치고 조건 변수 x에 대해 x.signal()을 호출한다.
  - 조건 변수 x에 대한 대기 큐에 있던 프로세스 A가 깨어나고, 프로세스 B가 모니터를 떠나면 프로세스 A가 모니터에 진입할 수 있다.
 
