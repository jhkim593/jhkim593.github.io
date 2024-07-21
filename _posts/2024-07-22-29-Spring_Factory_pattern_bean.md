---
layout: post
title: "Factory Pattern 적용을 통한 if - else 구문 제거"
author: "jhkim593"
tags: Spring
---
## 기존 문제

프로젝트를 진행하면서 상황에 따라 다른 비즈니스 로직을 적용해야 하는 경우가 많은데 일반적으로 if-else문을 통해 구현하게 됩니다. 예를 들면 다음과 같습니다.

```java
@Service
@RequiredArgsConstructor
public class RunService {
  private final ToolARuuner toolARuuner;
  private final ToolBRuuner toolBRuuner;
  private final ToolCRuuner toolCRuuner;

  public void run(ToolType toolType){
      if(ToolType.A.getType() == toolType ){
      //처리 로직
      toolARunner.run();
      } else if(ToolType.B.getType() == toolType ){
      //처리 로직
      toolBRunner.run();
      } else if(ToolType.C.getType() == toolType){
      //처리 로직
      toolCRunner.run();
      }
      ...
    }
}
```

기존에는 위 예시와 같이 if - else 문을 통해 파라미터로 받은 ToolType에 따라 다른 처리 로직을 타도록 구현했습니다.

하지만 해당 구조는 기존에 사용하던 ToolType이 제거되거나 새로운 ToolType이 추가 된다고 하면 **서비스 로직의 if - else문도 그에 맞게 모두 수정**해야했습니다. 위 예시처럼 한곳이라면 괜찮겠지만 ToolType을 사용하는 로직이 점점 많아질 수록 변경에 쉽게 대응할 수 없습니다.

<br>
## Factory Pattern 적용
새로운 ToolType이 추가되거나 기존 ToolType이 삭제된다고 해도 기존 서비스 로직에 변경이 없도록 하기 위해 **팩토리 패턴**을 적용했습니다.

먼저 도구별 처리 메소드를 인터페이스로 추상화 합니다.

```java
public interface ToolRunner {
    void run() throws Exception;
    ToolType getToolType();
}
```

<br>
다음 각 도구별로 구현체를 생성합니다.

```java
//ToolA
@Component
public class ToolARuuner implements ToolRunner{
    @Override
    public void run(){
	    ...
    }
    @Override
    public ToolType getToolType() {
        return ToolType.A;
    }
}

//ToolB
@Component
public class ToolBRuuner implements ToolRunner{
    @Override
    public void run(){
	    ...
    }
    @Override
    public ToolType getToolType() {
        return ToolType.B;
    }
}

//ToolC
@Component
public class ToolCRuuner implements ToolRunner{
    @Override
    public void run(){
	    ...
    }
    @Override
    public ToolType getToolType() {
        return ToolType.C;
    }
}
```
도구별 각각 구현체를 생성했으며 @Component 어노테이션을 통해 빈으로 등록했습니다.
getToolType은 각 도구의 ToolType을 return 하도록 구현했습니다.

<br>
이어서 상황에 따라 다른 객체를 return할 Factory class를 생성해보겠습니다.
```java
@Component
@RequiredArgsConstructor
public class ToolRunnerFactory {
    private final Map<ToolType, ToolRunner> toolRunnerMap = new HashMap<>();

    public ToolRunnerFactory (List<ToolRunner> toolRunners) {
        toolRunners.forEach(i -> toolRunnerMap.put(i.getToolType(), i));
    }
    public ToolRunner get(ToolType toolType) {
        return toolRunnerMap.get(toolType);
    }
}
```
ToolARuuner, ToolBRuuner, ToolCRuuner를 **모두 빈으로 등록했기 때문에 생성자에서 각 구현체를 List로 받을 수 있습니다.** 파라미터로 받은 ToolRunner 구현체를 순회하며 ToolType을 key값으로한 toolRunnerMap을 설정합니다. 이후에 get 메소드를 통해 설정된 toolRunnerMap을 통해 ToolType에 맞는 객체를 return하게됩니다.

<br>
이제 위 팩토리 클래스를 적용해 기존 RunService 코드를 개선 해보겠습니다.
```java
@Service
@RequiredArgsConstructor
public class RunService {
	private final ToolRunnerFactory toolRunnerFactory;
	//개선
	public void run(ToolType toolType) throws Exception {
      toolRunnerFactory.get(toolType).run();
  }
```
RunService 에서 ToolRunnerFactory를 통해 동적으로 조건에 맞는 구현체 메소드를 호출할 수 있도록 개선할 수 있었습니다.

<br>
## 결론

Factory 클래스를 적용해보면서 느낀 해당 방식의 장단점은 다음과 같습니다.

### 장점

- 길게 작성된 if - else문 보다 가독성이 좋습니다.
- ToolType이 추가 , 삭제된다고 해도 Service , Factory 클래스 모두 수정하지 않아도 됩니다.

### 단점

- 등록된 bean을 대상으로 하기 때문에 Spring Framework에 의존적입니다.
- interface 및 factory 클래스 추가로 구조가 복잡해집니다.
