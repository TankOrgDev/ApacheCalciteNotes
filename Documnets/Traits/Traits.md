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
1. 노드의 현재 속성을 getTraitSet 메서드를 통해 가져온다
2. 새로운 RelTraitSet 인스턴스를 생성하고 속성을 업데이트한다
3. RelOptRule.convert 메서드 호출을 통해 속성을 정의한다

　내부적으로, Apache Calcite는 대상이되는 연산자의 윗부분에 **AbstractConverter** 연산자를 추가해서 속성들을 정의한다.
```text
AbstractConverter   [Sorted by a]
    Input[t2]       [NOT SORTED]
```
AbstractConverter를 Sort와 같은 구현된 연산자노드로 바꾸기 위해선 ExpandConversionRule 규칙을 최적화에 포함해야한다.
이 Rule이 적용되면 AbstractConverter는 우리가 정의한 trait을 충족하기 위해 확장하는 작업을 수행하고, 위의 경우
만족되지 못한 속성이 Sort 하나이므로, Sort로 확장된다.
```text
Sort[t2.a]      [Sorted by a]
    Input[t2]   [NOT SORTED]
```

--- 

## Custom Property

　Property의 목적과 Calcite API를 이해하게ㄷ되면 사용자 지정 property의 정의, 등록 및 적용이 가능하다.
예를들어 모든 관계 연산자가 두 가지 방법 중 하나로 노드 간에 분산될 수 있는 분산 데이터베이스가 있다고 가정하자.
1. PARTITIONED
   - Relation이 분할되며 모든 tuple(row)은 노드 중 하나에 존재.
   - 대표적인 분산 데이터 구조
2. SINGLETON
   - Relation은 단일 노드에 존재.
   - 예를들어 최종 결과를 사용자 어플리케이션에 전달하는 커서
아래 설명에서는 최상위 연산자가 언제나 SINGLETON 분산을 보장하고, 단일 노드에 결과를 전달하는 것을 설명한다.

### Enforcer
　먼저 SINGLETON 분산을 보장하기 위해,모든 노드를 단일 노드로 옮길 필요가 있다.
분산 데이터베이스에서 데이터 이동 연산자는 Exchange라고 하며, Calcite에서 사용자 지정 연산자를 위한
최소한의 구현은 생성자와 복사를 정의하는 것이다.
```java
public class ExchangeRel extends SingleRel {
    public RedistributeRel(
        RelOptCluster cluster,
        RelTraitSet traits,
        RelNode input
    ) {
        super(cluster, traits, input);
    }

    @Override
    public RelNode copy(RelTraitSet traitSet, List<RelNode> inputs) {
        return new ExchangeRel(getCluster(), traitSet, inputs.get(0));
    }
}
```

### Trait
　다음으로 custom trait과 trait을 정의해야한다.
1. 특성은 getTraitDef 메소드에서 공통 특성 정의 인스턴스를 참조해야한다.
2. satisfies 메소드를 override하여, 현재 특성이 목표로하는 특성을 충족하는지 확인해야한다.
3. 특성 정의는 getTraitClass 메소드에서 trait의 java class를 선언해야한다
4. 특성 정의는 getDefault 메소드에서 기본값을 선언해야한다.
5. 현재 특성이 목표하는 특성을 충족하지 않은 경우 Enforcer를 생성하기 위해 호출하는 메소드 변환을 구현해야한다.
그렇지 않으면 null을 반환하도록 해야한다.

```java
public class Distribution implements RelTrait {

    public static final Distribution ANY = new Distribution(Type.ANY);
    public static final Distribution PARTITIONED = new Distribution(Type.PARTITIONED);
    public static final Distribution SINGLETON = new Distribution(Type.SINGLETON);

    private final Type type;

    private Distribution(Type type) {
        this.type = type;
    }

    @Override
    public RelTraitDef getTraitDef() {
        return DistributionTraitDef.INSTANCE;
    }

    @Override
    public boolean satisfies(RelTrait toTrait) {
        Distribution toTrait0 = (Distribution) toTrait;

        if (toTrait0.type == Type.ANY) {
            return true;
        }

        return this.type.equals(toTrait0.type);
    }

    enum Type {
        ANY,
        PARTITIONED,
        SINGLETON
    }
}
```
  
또한 현재 property가 목표로 하는 property를 충족하지 않는 경우, ExchangeRel enforcer를 생성하는 변환함수를 정의해야한다.
```java
public class DistributionTraitDef extends RelTraitDef<Distribution> {

    public static DistributionTraitDef INSTANCE = new DistributionTraitDef();

    private DistributionTraitDef() {
        // No-op.
    }

    @Override
    public Class<Distribution> getTraitClass() {
        return Distribution.class;
    }

    @Override
    public String getSimpleName() {
        return "DISTRIBUTION";
    }

    @Override
    public RelNode convert(
        RelOptPlanner planner,
        RelNode rel,
        Distribution toTrait,
        boolean allowInfiniteCostConverters
    ) {
        Distribution fromTrait = rel.getTraitSet().getTrait(DistributionTraitDef.INSTANCE);

        if (fromTrait.satisfies(toTrait)) {
            return rel;
        }

        return new ExchangeRel(
            rel.getCluster(),
            rel.getTraitSet().plus(toTrait),
            rel
        );
    }

    @Override
    public boolean canConvert(
        RelOptPlanner planner,
        Distribution fromTrait,
        Distribution toTrait
    ) {
        return true;
    }

    @Override
    public Distribution getDefault() {
        return Distribution.ANY;
    }
}
```

### Putting It All Together

　우선 PARTITIONED와 SINGLETON 분산을 가진 테이블을 만들어주고 플래너 인스턴스에 custom trait을 등록한다.
```text
// Table with PARTITIONED distribution.
Table table1 = Table.newBuilder("table1", Distribution.PARTITIONED)
  .addField("field", SqlTypeName.DECIMAL).build();

// Table with SINGLETON distribution.
Table table2 = Table.newBuilder("table2", Distribution.SINGLETON)
  .addField("field", SqlTypeName.DECIMAL).build();

Schema schema = Schema.newBuilder("schema").addTable(table1).addTable(table2).build();

VolcanoPlanner planner = new VolcanoPlanner();

planner.addRelTraitDef(ConventionTraitDef.INSTANCE);
planner.addRelTraitDef(DistributionTraitDef.INSTANCE);
```

앞서 언급한 ExpandConversionRule을 적용하고 최상위 노드에 SINGLETON 분산 trait을 적용하고 플래닝을 진행한다.
```text
// Use the built-in rule that will expand abstract converters.
RuleSet rules = RuleSets.ofList(AbstractConverter.ExpandConversionRule.INSTANCE);

// Prepare the desired traits with the SINGLETON distribution.
RelTraitSet desiredTraits = node.getTraitSet().plus(Distribution.SINGLETON);
        
// Use the planner to enforce the desired traits
RelNode optimizedNode = Programs.of(rules).run(
    planner,
    node,
    desiredTraits,
    Collections.emptyList(),
    Collections.emptyList()
);
```

---

## Summary

　Physical property는 여러 다른 방법을 찾을 수 있는 쿼리 최적화의 중요한 컨셉이다. 위의 정리에서는 custom physical property를
calcite내에 정의하는 방법과 custom RelTraitDef와 relTrait를 만드는 방법, 플래너에 등록하여 property의 목표하는 값으로
정의하여 custom operator를 사용하는 법을 다뤄보았다.

　그러나 연산자간의 property의 전파와 관련된 중요한 문제가 존재하며 calcite는 이 부분이 잘 되지 않기에
여러 해결책을 마련해두고 가장 최선의 방법을 고르길 제안한다.