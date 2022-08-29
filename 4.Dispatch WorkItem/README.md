```
func sync(execute block: () -> Void)

func async(group: DispatchGroup? = nil,
           qos: DispatchQoS = .unspecified,
           flags: DispatchWorkItemFlags = [],
           execute work: @escaping () -> Void)
```

지금까지 디스패치 큐(DispatchQueue)에 클로저 형태로 작업을 보냈다. 반복되는 작업을 디스패치 큐에 매번 클로저의 형태로 보내는 것보다는 작업을 캡슐화한 디스패치 워크 아이템(Dispatch WorkItem)을 사용하는 것을 추천한다.

# **Dispatch WorkItem**

디스패치 워크 아이템은 클래스로 작업을 캡슐화할 뿐만 아니라 아래의 추가적인 기능을 제공한다.

-   `perform()`
-   `notify()`
-   `wait()`
-   `cancel()`

## **사용**

```
let task = DispatchWorkItem(qos: .background) {
    print("Task 시작")
    sleep(3)
    print("Task 완료")
}

let queue = DispatchQueue.global()
queue.async(execute: task)
```

디스패치 워크 아이템 객체를 만들어 `excute` 매개변수 보내면 된다. 위의 코드는 아래 코드와 동일하다.

```
queue.async(qos: .background) {
    print("Task 시작")
    sleep(3)
    print("Task 완료")
}
```

## **Perfom 메서드**

`perform` 메서드는 현재 스레드에서 디스패치 워크 아이템을 동기적으로 실행한다. 따라서 메인 메서드에서 주의해서 사용해야 한다.

```
let task = DispatchWorkItem(qos: .background) {
    print("Task 시작")
    sleep(3)
    print("Task 완료")
}
task.perform()
```

## **notify 메서드**

디스패치 그룹의 `notify` 메서드와 비슷한 기능을 한다. `notify` 메서드를 호출한 아이템이 끝난 후 실행할 작업과 큐를 지정할 수 있다. 

```
let task1 = DispatchWorkItem(qos: .background) {
    print("\(currentTimeString()) : Task1 시작")
    sleep(3)
    print("\(currentTimeString()) : Task1 완료")
}

let task2 = DispatchWorkItem(qos: .background) {
    print("\(currentTimeString()) : Task2 시작")
    sleep(1)
    print("\(currentTimeString()) : Task2 완료")
}

DispatchQueue.global().async(execute: task1)
task1.notify(queue: .global(), execute: task2)

/*
2022-08-29 23:43:59.938 : Task1 시작
2022-08-29 23:44:03.017 : Task1 완료
2022-08-29 23:44:03.019 : Task2 시작
2022-08-29 23:44:04.116 : Task2 완료
*/
```

## **wait 메서드**

`wait` 메서드를 호출한 디스패치 워크 아이템이 끝날 때까지 최대 매개변수로 보낸 시간만큼 현재 큐를 블록 처리한다. 동기적인 순서로 아이템이 실행되어야 할 때 사용하면 된다. Task 1 시작이 출력된 후 Task 1이 완료될 때까지 최대 2초까지 기다린 후 Task 2 시작이 출력된다.

```
let task1 = DispatchWorkItem(qos: .background) {
    print("\(currentTimeString()) : Task1 시작")
    sleep(3)
    print("\(currentTimeString()) : Task1 완료")
}

let task2 = DispatchWorkItem(qos: .background) {
    print("\(currentTimeString()) : Task2 시작")
    sleep(1)
    print("\(currentTimeString()) : Task2 완료")
}

let que = DispatchQueue(label: "com.young.concurrent", attributes: .concurrent)
que.async(execute: task1)
task1.wait(timeout: .now() + 2)
que.async(execute: task2)

/*
2022-08-30 00:13:47.106 : Task1 시작
2022-08-30 00:13:49.125 : Task2 시작
2022-08-30 00:13:50.173 : Task1 완료
2022-08-30 00:13:50.219 : Task2 완료
*/
```

## **cancel 메서드**

작업이 아직 시작이 안된 경우에 메서드를 호출하면 해당 작업이 큐에서 제거되며, 작업이 시작된 후에 호출하면 `isCancelled` 프로퍼티가 true로 설정된다.  현재 실행 중인 작업을 멈추는 것이 아닌 `isCancelled` 프로퍼티만 변경된다.

task2가 시작되기 전에 `cancel` 메서드가 실행되어 `notify` 메서드가 바로 실행된다.

```
let task1 = DispatchWorkItem(qos: .background) {
    print("\(currentTimeString()) : Task1 시작")
    sleep(3)
    print("\(currentTimeString()) : Task1 완료")
}

let task2 = DispatchWorkItem(qos: .background) {
    print("\(currentTimeString()) : Task2 시작")
    sleep(1)
    print("\(currentTimeString()) : Task2 완료")
}

task1.notify(queue: .main) {
    print("task1 완료")
}
task2.notify(queue: .main) {
    print("task2 완료", task2.isCancelled)
}

let que = DispatchQueue(label: "com.young.concurrent", attributes: .concurrent)
que.async(execute: task1)
task2.cancel()
que.async(execute: task2)

/*
2022-08-30 00:26:07.188 : Task1 시작
task2 완료 true
2022-08-30 00:26:10.293 : Task1 완료
task1 완료
*/
```

task2가 실행된 후 `cancel` 메서드를 실행하는 경우에 실행결과는 다양하지만 모든 경우에 공통적으로 작업이 실행된 후 `cancel` 될 경우 작업은 실행되고 `isCancelled` 프로퍼티가 변경된 다는 것을 알 수 있다. 

경우 3은 작업을 모두 비동기적으로 보냈기 때문에 근소한 차이로 `cancel` 메서드가 task2가 실행되기 전에 먼저 실행된 경우이다.

```
let task1 = DispatchWorkItem(qos: .background) {
    print("\(currentTimeString()) : Task1 시작")
    sleep(3)
    print("\(currentTimeString()) : Task1 완료")
}

let task2 = DispatchWorkItem(qos: .background) {
    print("\(currentTimeString()) : Task2 시작")
    sleep(1)
    print("\(currentTimeString()) : Task2 완료")
}

task1.notify(queue: .main) {
    print("task1 완료")
}
task2.notify(queue: .main) {
    print("task2 완료", task2.isCancelled)
}

let que = DispatchQueue(label: "com.young.concurrent", attributes: .concurrent)
que.async(execute: task1)
que.async(execute: task2)
task2.cancel()

/* 경우 1
2022-08-30 00:28:19.649 : Task2 시작
2022-08-30 00:28:19.649 : Task1 시작
2022-08-30 00:28:20.757 : Task2 완료
task2 완료 true
2022-08-30 00:28:22.713 : Task1 완료
task1 완료
*/

/* 경우 2
2022-08-30 00:29:20.426 : Task1 시작
2022-08-30 00:29:20.426 : Task2 시작
2022-08-30 00:29:21.531 : Task2 완료
task2 완료 true
2022-08-30 00:29:23.532 : Task1 완료
task1 완료
*/


/* 경우 3
2022-08-30 00:29:01.784 : Task1 시작
task2 완료 true
2022-08-30 00:29:04.883 : Task1 완료
task1 완료
*/
```