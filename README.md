# Spring_Cloud_Config
Spring Cloud Config

Zuul API Gateway를 구현할 때 기본적으로 Ribbon-Eureka를 같이 사용하는 것을 권장한다.  
당장에 Netflix Zuul과 관련된 서적을 봐도 Eureka 없이 Ribbon을 구현하는 예제는 찾아보기 힘들다.  
하지만 생각보다 많은 프로젝트에서 Eureka는 잘 사용되지 않는 것 같다. 그래서 Eureka 없이 Ribbon을 구현하면서 겪은 것을 정리해서 공유하고자 한다.  

## 1초도 안되는 짧은 타이밍에 들어오는 요청. 1초가 넘어가는 딜레이.  
Eureka 없이 로드밸런서를 구현하는 것은 예상보다 쉬웠다. Ribbon은 정상적으로 1번 서버와 2번 서버를 번갈아가며 요청을 전달했다. 문제는 이 다음이다.  
이번엔 2번 서버가 정지된 상태로 요청을 보냈다. 이상하게도 Zuul은 정상적으로 1번 서버를 부르다가도 간헐적으로 500 Error를 뱉어냈다.  
여기저기 찾아본 끝에 ``zuul: retryable: true``로 설정하니까 500 Error는 사라졌다. 대신에 그 자리에 1초가 넘어가는 딜레이가 생겼다.  
아무래도 내가 바꾼건 "2번 서버를 (죽었다고 판단하지 않고) 부른다"에서 "2번 서버를 불러도 답이 없으면 1번 서버를 부른다" 뿐인 것 같다.  

여기서 두가지 의문이 생겼다. 왜 간헐적으로 일어나는 것일까? 왜 2번 서버를 죽었다고 판단하지 않는 걸까?  

1. 왜 간헐적으로 일어나는 것일까?  
해답을 알기 위해 여러 차례 요청을 보내본 결과 Ribbon이 서버리스트를 갱신하고 있을 때 요청이 들어오면 1초 딜레이가 생긴다는 걸 알 수 있었다.  
(Ribbon은 일정 주기마다 각각의 서버에 Ping을 날려서 상태를 확인한다.)

2. 왜 2번 서버를 죽었다고 판단하지 않는 걸까?  
1번 의문을 해결하면서 당황스러웠던 점은 Ribbon이 서버리스트를 갱신하고 있을 때에만 2번 서버를 살아있다고 잘못 인식한다는 점이었다.  
(평소에는 2번 서버가 죽었다고 판단하고 절대 요청을 2번 서버로 전달하지 않는다.)  
2번 의문에 대한 해답은 조금 더 깊이 파볼 필요가 있었다.

## 원인은 DynamicServerListLoadBalancer  
디버깅 모드로 어느 시점에 서버리스트가 정상적으로 업데이트가 되지 않는지를 찾아보았다.  
그 결과 ``DynamicServerListLoadBalancer`` 클래스의 ``updateAllServerList()`` 메소드에 문제가 있다는 것을 알게 되었다.

## updateAllServerList()가 가진 문제점은?  
이 메소드는 각각의 서버에 Ping을 쏘고 그 결과를 서버리스트에 저장하는 역할을 한다.  
이상한 점은 Ping을 쏘기 전에 서버 상태를 전부 true(=살아있음)로 바꾼다는 것이다.  
```java
    protected void updateAllServerList(List<T> ls) {
        // other threads might be doing this - in which case, we pass
        if (serverListUpdateInProgress.compareAndSet(false, true)) {
            try {
                for (T s : ls) {            // for문으로 모든 서버 상태를 true로 변경
                    s.setAlive(true);
                }
                setServersList(ls);         // 변경된 값을 allServerList에 저장
                super.forceQuickPing();     // 각각의 서버에 Ping을 보내 서버 상태를 체크해서 allServerList를 업데이트
            } finally {
                serverListUpdateInProgress.set(false);
            }
        }
    }
```
어떤 의도로 코드가 이렇게 작성되었는지는 정확히 모르겠지만 서버 상태를 전부 true로 바꾸는 구문에는 분명 문제가 있었다. 그래서 결국 해당 구문을 수정하기로 했다.  

## 어떻게 수정할 것인가?  
서버가 초기 구동되었을 때 ``allServerList``가 초기화되어 있지 않기 때문에 ``setServersList(ls);``를 아예 지워버리면 에러가 생긴다.  
그래서 간단하게 "``allServerList``가 null이거나 비어있을 때만 true로 초기화한다" 구문만 추가했다.  
```java
    protected void updateAllServerList(List<T> ls) {
        // other threads might be doing this - in which case, we pass
        if (serverListUpdateInProgress.compareAndSet(false, true)) {
            try {
                if (allServerList == null || allServerList.size() == 0) {   // 서버가 초기 구동될 때에만 실행되도록 변경
	                for (T s : ls) {
	                    s.setAlive(true);
	                }
                	setServersList(ls);
                }
                super.forceQuickPing();
            } finally {
                serverListUpdateInProgress.set(false);
            }
        }
    }
```

## 또다른 문제
처음에는 임의의 ``CustomLoadBalancer`` 클래스를 만들어서 ``DynamicServerListLoadBalancer``를 상속받아 ``updateAllServerList()``만 오버라이딩을 할 생각이었다. 하지만 ``updateAllServerList()``의 ``super.forceQuickPing();`` 구문이 문제였다.  
상속관계가 A->B->C 이고 A 클래스의 메소드를 B 클래스에서 오버라이딩을 하지 않는다면 C 클래스에서 A 클래스의 메소드를 사용할 수 있다.
하지만 B클래스에 해당되는 ``DynamicServerListLoadBalancer``는 ``forceQuickPing()``을 아무것도 수행하지 않는 빈 메소드로 오버라이딩을 해놓았다.  
```
    public void forceQuickPing() {  // 이것 때문에 상속받아 사용할 수가 없다...
    }
```
``super.forceQuickPing();``이 빈 메소드를 불러와서 실행을 하기때문에 ``DynamicServerListLoadBalancer`` 클래스를 상속받아 쓰기는 불가능했고 ``DynamicServerListLoadBalancer``의 부모인 ``BaseLoadBalancer``를 상속받아서 ``DynamicServerListLoadBalancer``와 똑같지만 ``updateAllServerList()`` 부분을 수정한 클래스를 생성해서 사용했다.

# 관련된 Github 이슈들
https://github.com/spring-cloud/spring-cloud-netflix/issues/2485  
https://github.com/Netflix/ribbon/issues/368  
https://github.com/Netflix/ribbon/issues/354  
