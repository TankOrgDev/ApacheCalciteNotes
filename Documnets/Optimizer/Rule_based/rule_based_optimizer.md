# Rule based optimization

## Transformation

　최적화기는 여러 개의 동등한 플랜 중에 하나의 최적화 방법을 선택해야만 하며, 직관적으로는
모든 가능한 입력에 대한 결과가 같다면 동일한 플랜이라고 할 수 있다. 우리는 동등한 실행결과를 얻는
실행 계획을 얻기위해, 하나 또는 그 이상의 변환을 원래의 플랜에 적용하게 된다.

　그 중 어떤 변환은 플랜의 큰 부분을 차지하거나 전체 플랜에 대해 수행될 수 있다.
예를 들어, join 순서의 구현을 위해 동적 프로그래밍을 통해 모든 조인의 경우의 수를 가지고
가장 좋은 것을 선택할 수 있는 것 처럼 말이다.

<p align="center"><img src="./img/join_order.png" width="80%"></p>

　또 어떤 변환은 상대적으로 격리되어 실행될 수 있다. Filter 연산을 Aggregation 아래로 push down
 시키는 변환을 고려해보면, 전역적인 변환이 가해지는 것이 아닌 특정 격리된 부분에서만 변환이 수행된다.

<p align="center"><img src="./img/filter_pushdown.png" width="50%"></p>

## Rules

　모든 최적화기는 어떠한 변환을 시행할 시점과 그 변환으로 생성되는 새로운 등가계획을 처리하는 방법에
대해 몇가지 알고리즘을 따르게 되어있다. 따라서 변환의 갯수가 늘어날 수록, 단순히 어떤 플랜에 대해
 이 변환을 적용할 수 있는지 if-else 식으로 비교하여 확인하는 것은 상당히 불편한 일이 될 것이다.  
 　따라서 공통된 부분을 인터페이스화하고 플랜의 일정 부분에 변환을 적용할 수 있는지 여부를
 정의하는 패턴을 정의하게 되는데, 이런 **패턴과 변환의 한 쌍**을 우리는 **Rule** 이라고한다.
규칙을 추상화하게 되면 독립적으로 최적화 논리를 개발하고 플러거블하게 분할하여 최적화 개발을 단순화 할 수 있다.
이런 규칙들을 이용해 계획을 생성하는것을 **rule based optimization** 이라고 한다.

<p align="center"><img src="./img/rule_base.png" width="70%"></p>

## In Apache Calcite

　Calcite에서는 두 가지 rule base 최적화기를 제공한다.
- HepPlanner
  - Heuristic optimizer로 더 이상 변환이 불가능할 때까지 규칙을 하나씩 적용한다
- VolcanoPlanner
  - cost-based optimizer이지만, rule-based로 동작하며 추가적으로 데이터 구조에 넣어
  비용을 산출하는 최적화기이다. 
  - 하향식으로 동작

Rule Interface를 통해 패턴을 확인하는데, onMatch() 함수를 사용자가 직접 구현하는 식으로
패턴을 설정하고, 메서등의 파라미터로 변환된 기능을 등록할 수 있는 기능을 제공한다.
```java
class CustomRule extends RelOptRule {
    CustomRule() {
        super(pattern_for_the_rule);
    }
    
    void onMatch(RelOptRuleCall call) {
        RelNode equivalentNode = node;
        
        // Register the new equivalent node in MEMO
        call.transformTo(equivalentNode);
    }
}
```

