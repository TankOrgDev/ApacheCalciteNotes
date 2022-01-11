# Apache Calcite Overview

## Schema

　Schema의 정의에 대해서 알아보기 전에, 사용자 정의 테이블 구현을 먼저 진행한다.
Apache Calcite의 **AbstractTable**을 extends해 사용자 정의 테이블을 만들수 있다.
다음 두가지 정보를 테이블에 포함할 것이다.
1. Field Name and type
   - 테이블의 row type을 생성하는데 필요한 정보(expression 파생시 필요하다)
2. Optional Statistic
   - 쿼리 플래닝 할때 유용한 정보로 row count, collations, unique table key 등이 존재한다.
   - 예제에서는 row count 정보만을 다루기로 한다
```java
public class SimpleTableStatistic implements Statistic {

    private final long rowCount;
    
    public SimpleTableStatistic(long rowCount) {
        this.rowCount = rowCount;
    }
    
    @Override
    public Double getRowCount() {
        return (double) rowCount;
    }
}
```
```java
public class SimpleTable extends AbstractTable {

    private final String tableName;
    private final List<String> fieldNames;
    private final List<SqlTypeName> fieldTypes;
    private final SimpleTableStatistic statistic;

    private RelDataType rowType;

    private SimpleTable(
        String tableName, 
        List<String> fieldNames, 
        List<SqlTypeName> fieldTypes, 
        SimpleTableStatistic statistic
    ) {
        this.tableName = tableName;
        this.fieldNames = fieldNames;
        this.fieldTypes = fieldTypes;
        this.statistic = statistic;
    }
    
    @Override
    public RelDataType getRowType(RelDataTypeFactory typeFactory) {
        if (rowType == null) {
            List<RelDataTypeField> fields = new ArrayList<>(fieldNames.size());

            for (int i = 0; i < fieldNames.size(); i++) {
                RelDataType fieldType = typeFactory.createSqlType(fieldTypes.get(i));
                RelDataTypeField field = new RelDataTypeFieldImpl(fieldNames.get(i), i, fieldType);
                fields.add(field);
            }

            rowType = new RelRecordType(StructKind.PEEK_FIELDS, fields, false);
        }

        return rowType;
    }

    @Override
    public Statistic getStatistic() {
        return statistic;
    }
}
```
```java
public class SimpleTable extends AbstractTable implements ScannableTable {
    @Override
    public Enumerable<Object[]> scan(DataContext root) {
        throw new UnsupportedOperationException("Not implemented");
    }
}
```
　우리의 테이블은 Apache Calcite의 **ScannableTable** 인터페이스 역시 implements하는데, Enumerable 최적화 규칙을 사용 시
실패하는 경우가 있기 때문에 데모 목적으로 사용한다.
```java
public class SimpleSchema extends AbstractSchema {

    private final String schemaName;
    private final Map<String, Table> tableMap;

    private SimpleSchema(String schemaName, Map<String, Table> tableMap) {
        this.schemaName = schemaName;
        this.tableMap = tableMap;
    }

    @Override
    public Map<String, Table> getTableMap() {
        return tableMap;
    }
}
```
　마지막으로 **AbstractSchema**를 정의해 사용자 정의 스키마를 만든다. tableName과 Table 정보를 가진 map을 통해
Calcite는 semantic validation을 실시한다.

---

## Optimizer

　최적화 프로세스는 다음 순서로 진행된다.
1. 쿼리 스트링을 Syntax analysis 통해 Abstract Syntax Tree(AST)로 만든다.
2. AST에 대한 Semantic analysis 진행
3. AST를 relational tree로 변환
4. Relational tree 최적화

### Configuration

　Apache Calcite의 쿼리 최적화 클래스 다수는 configuration이 필요하다. 그러나 모든 객체에 사용할 수 있는 configuration 클래스는
존재하지 않기 때문에, 여러 공통 configuration을 하나의 객체에 저장해두고 필요할 때 마다 configuration value를 복사하는 방식으로
진행하게 된다.
```text
Properties configProperties = new Properties();

configProperties.put(CalciteConnectionProperty.CASE_SENSITIVE.camelName(), Boolean.TRUE.toString());
configProperties.put(CalciteConnectionProperty.UNQUOTED_CASING.camelName(), Casing.UNCHANGED.toString());
configProperties.put(CalciteConnectionProperty.QUOTED_CASING.camelName(), Casing.UNCHANGED.toString());

CalciteConnectionConfig config = new CalciteConnectionConfigImpl(configProperties);
```

### Syntax Analysis

　우선적으로 우리는 쿼리를 파싱하게되는데 그 결과는 AST형태로, 모든 노드들은 **SqlNode**의 subclass가 된다.
우리는 앞서 정의한 configuration을 파서에 셋팅하고 **SqlParser**를 통해 파싱을 진행한다.
만약 사용자 정의 SQL 문법이 있다면, parser factory를 따로 지정해줘야함을 인지하자.
```text
public SqlNode parse(String sql) throws Exception {
    SqlParser.ConfigBuilder parserConfig = SqlParser.configBuilder();
    parserConfig.setCaseSensitive(config.caseSensitive());
    parserConfig.setUnquotedCasing(config.unquotedCasing());
    parserConfig.setQuotedCasing(config.quotedCasing());
    parserConfig.setConformance(config.conformance());

    SqlParser parser = SqlParser.create(sql, parserConfig.build());

    return parser.parseStmt();
}
```

### Semantic Analysis

　Semantic analysis의 최종 목적은 AST가 유효함을 보장하는 것이다. 이는 다음을 포함한다.
- 객체와 함수의 한정자
- 데이터 타입 참조
- sql 문법의 정확성

　유효성 검사는 **SqlValidatorImpl**을 통해 진행되며 Calcite에서 가장 복잡한 클래스 중 하나이다.
이 클래스를 실행하기 위해선 몇 개의 객체가 필요로 하는데, 우선적으로 **RelDataTypeFactory**라는 SQL type에 대한 정의이다.
여기서는 built-in factory를 사용했지만, 필요하다면 사용자가 직접 datatype에 관한 factory를 생성해 진행할 수 있다.
```text
RelDataTypeFactory typeFactory = new JavaTypeFactoryImpl();
```

　다음으로 **Prepare.CatalogReader** 객체를 만들게 되는데, 데이터베이스 객체에 접근하기 위한 객체이다.
여기서 우리가 만든 Schema가 사용되는데, 카탈로그 리더는 파싱 중에 객체의 이름에 대한 일관성을 유지하기 위해 
우리가 만든 configuration 객체를 사용한다.
```text
SimpleSchema schema = ... // Create our custom schema

CalciteSchema rootSchema = CalciteSchema.createRootSchema(false, false);
rootSchema.add(schema.getSchemaName(), schema);

Prepare.CatalogReader catalogReader = new CalciteCatalogReader(
    rootSchema,
    Collections.singletonList(schema.getSchemaName()),
    typeFactory,
    config
);
```
　이후 **SqlOperatorTable**을 정의하여 SQL 함수와 연산자에 대한 라이브러리를 추가해준다.
```text
SqlOperatorTable operatorTable = ChainedSqlOperatorTable.of(
    SqlStdOperatorTable.instance()
);
```
　Validaton을 위한 supporting 객체들이 모드 만들어지면 SqlValidatorImpl을 인스턴스화 하여 validation을 진행 할 수 있다.
validator 인스턴스는 AST를 relational tree로 변환하는데도 사용되어진다.
```text
SqlValidator.Config validatorConfig = SqlValidator.Config.DEFAULT
    .withLenientOperatorLookup(config.lenientOperatorLookup())
    .withSqlConformance(config.conformance())
    .withDefaultNullCollation(config.defaultNullCollation())
    .withIdentifierExpansion(true);

SqlValidator validator = SqlValidatorUtil.newValidator(
    operatorTable, 
    catalogReader, 
    typeFactory,
    validatorConfig
);

SqlNode sqlNode = parse(sqlString);
SqlNode validatedSqlNode = validator.validate(node);
```

### Conversion to a Relational Tree

　AST는 문법 구조가 복잡하기 때문에 쿼리 최적화를 하기에는 적절하지 않다. 따라서 Calcite에서는 relational operator tree인
**SqlNode(Scan,Project,Filter,Join, etc..)** 를 통해 보다 편리하게 최적화를 진행하게된다.
 **SqlToRelConverter**라는 SqlValidation과 마찬가지로 굉장히 복잡한 클래스를 통해 AST를 realtional tree로 바꾸게 된다.
  
　우리가 converter를 만들면서 planner를 선택하게 되는데 아래에서는 cost-based planner인 **VolcanoPlanner**를 사용한다.
VolcanoPlanner를 만들기 위해선 이전에 만들었던 configuration과 **RelOptCostFactory**라는 플래너가 cost를 계산하게 해주는
클래스가 필요하다. Built-in으로 내장된 row count를 통한 카디널리티로 비용을 산출하는 방법은 상용화 프로그램에서 사용하기엔
부족할 수 있음을 인지해야한다.
  
　추가적으로 VolcanoPlanner가 지켜야하는 physical poroperty도 설정해 줘야한다. 모든 속성은 **RelTraitDef**에서 확장된
디스크립터를 사용하며, 아래에서는 **ConventionTraitDef**라는 관계형 노드의 실행 관계 백엔드와 관련된 trait을 등록할 것이다.
```text
VolcanoPlanner planner = new VolcanoPlanner(
    RelOptCostImpl.FACTORY, 
    Contexts.of(config)
);

planner.addRelTraitDef(ConventionTraitDef.INSTANCE);
```

　Trait이 설정된 planner가 생성되면 변환과 최적화에서 사용되는 공통 컨텍스트 객체인 **RelOptCluster**를 만들어준다.
```text
RelOptCluster cluster = RelOptCluster.create(
    planner, 
    new RexBuilder(typeFactory)
);
```

　그리고 나서 configuration의 설정과 함께 converter를 만들어준다.
```text
SqlToRelConverter.Config converterConfig = SqlToRelConverter.configBuilder()
    .withTrimUnusedFields(true)
    .withExpand(false) 
    .build();

SqlToRelConverter converter = new SqlToRelConverter(
    null,
    validator,
    catalogReader,
    cluster,
    StandardConvertletTable.INSTANCE,
    converterConfig
);
```

　Converter가 만들어지게 되면 이제 relational tree를 만들수 있다.
 Conversion을 진행하면 Calcite는 Logical relational operator를 생성하게 되고, 이는 추상적이며
 특정 실행 백엔드를 대상으로 하지않는다. 이러한 이유로 논리적 연산자의 convention은 언제나 Convention.NONE으로 설정한다.
 최적화 중 physical operator로 변환되며, 그때는 Convention.NONE과 다른 trait들이 적용되어 있다.
 
### Optimization

