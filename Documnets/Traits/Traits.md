# Traits

## Abstract

　물리적 속성은 더 많은 대안을 탐색할 수 있게하는 최적화 과정의 필수요소이다.
Apache Calcite에서는 **Convention**, **Collation**(정렬 순서)과 함께 제공된다.
여기서는 traits이라는 custom property를 cost-based optimizer 사용하여 정의하고, 등록하고 적용하는 방법에 대해 정리할 것이다.

---

## Physical Properties

　쿼리 최적화는 Scan, Project, Filter, Join과 같은 여러 연산자와 함께 동작한다. 
최적화를 진행하는 과정에서, 연산자는 입력의 특정 상황, 컨디션을 만족해야될 필요성이 있다.
그 컨디션이 만족되는지 아닌지를 확인하기 위해서 연산자는 물리적 속성(physical properties)을
노출할 수 있다. 여기서 말하는 물리적 속성은 연산자와 관련된 plain values에 해당한다.
연산자는 입력 속성과 원하는 속성을 비교하고, 특별한 적용 연산자를 통해 원하는 속성을 적용 시킬 수 있다.  

```text
t1 JOIN  t2 ON t1.a = t2.b
```
　위의 JOIN 연산을 보면, t1.a와 t2.b 입력에 대해 각각 정렬되어 있다면 우리는 merge join을 사용할 수 있다.
따라서, 우리는 모든 연산자에 대한 **collation property**(정렬 속성)을 정의할 수 있고, 생성되는 행들에 대한 
정렬 순서를 설명할 수 있다.
```text
Join[t1.a = t2.b]
    Input[t1]   [Sorted by t1.a]
    Input[t2]   [NOT SORTED]
```
　Merge join 연산은 입력의 t1.a와 t2.b에 대해 정렬을 수행할 수 있다. 그렇기 때문에
첫 번째 입력인 t1.a에 대해서는 이미 정렬되어 있으므로 그대로 두고, 두 번째 입력인 t2.b는
정렬되어 있지 않으므로 **Sort 연산을 주입**하여 Merge join을 가능하게 만들 수 있다.
```text
Join[t1.a = t2.b]
    Input[t1]   [Sorted by t1.a]
    Sort[t2.a]  [Sorted by t2.b
        Input[t2]   [NOT SORTED]
```

---

## Apache Calcite API

　Apache Calcite에서는 Property(속성)들을 **RelTrait**과 **RelTraitDef** 클래스로 정의한다.
**RelTraits**은 속성의 **구체적인 값**을 나타내며, **RelTraitDef**는 속성의 이름, 예상되는 java class,기본 값, 적용하는 방법을
설명하는 **property definition**(속성 정의)이다. 이 속성 정의는 **RelOptPlanner.addRelTraitDef**를 통해
플래너에 등록되고, **플래너**는 모든 연산자가 기본값 여부와는 상관없이 등록된 **속성 정의에 대해 특정한 값을 가지도록** 한다.  

　노드의 모든 속성들은 **변경불가**한 데이터 구조인 **RelTraitSet**로 구성된다. 이 클래스 내 semantic을 복사하여 속성을
추가하거나 업데이트 할 수 있는 메서드(**RelOptNode.getTraitSet**)를 통해 구현된 연산자의 속성에 접근할 수 있다.

　플래닝 중 연산자에게 특정 속성을 정의하기 위해서는 몇가지 규칙을 따라야 한다.
1. 노드의 현재 속성을 getTraiSet 메서드를 통해 가져온다
2. 새로운 RelTraiSet 인스턴스를 생성하고 속성을 업데이트한다
3. RelOptRule.convert 메서드 호출을 통해 속성을 정의한다


