# Metrics Query Expression(MQE) Syntax
MQE is a string that consists of one or more expressions. Each expression could be a combination of one or more operations. 
The expression allows users to do simple query-stage calculation through [V3 APIs](./query-protocol.md#v3-apis).    

```text
Expression = <Operation> Expression1 <Operation> Expression2 <Operation> Expression3 ...
```

The following document lists the operations supported by MQE.

## Metrics Expression
Metrics Expression will return a collection of time-series values.

### Common Value Metrics
Expression:
```text
<metric_name>
```

For example: 
If we want to query the `service_sla` metric, we can use the following expression:
```text
service_sla
```

#### Result Type
The `ExpressionResultType` of the expression is `TIME_SERIES_VALUES`.

### Labeled Value Metrics
For now, we only have a single anonymous label with multi label values in a labeled metric. 
To be able to use it in expressions, define `_` as the anonymous label name (key).

Expression:
```text
<metric_name>{_='<label_value_1>,...'}
```
`{_='<label_value_1>,...'}` is the selected label value of the metric. If is not specified, all label values of the metric will be selected. 

For example:
If we want to query the `service_percentile` metric with the label values `0,1,2,3,4`, we can use the following expression:
```text
service_percentile{_='0,1,2,3,4'}
```

If we want to rename the label values to `P50,P75,P90,P95,P99`, see [Relabel Operation](#relabel-operation).

#### Result Type
The `ExpressionResultType` of the expression is `TIME_SERIES_VALUES` and with labels.

## Binary Operation 
Binary Operation is an operation that takes two expressions and performs a calculation on their results.
The following table lists the binary operations supported by MQE.

Expression:
```text
Expression1 <Binary-Operator> Expression2
```

| Operator | Definition           |
|----------|----------------------|
| +        | addition             |
| -        | subtraction          |
| *        | multiplication       |
| /        | division             |
| %        | modulo               |

For example:
If we want to transform the service_sla metric value to percent, we can use the following expression:
```text
service_sla / 100
```

### Result Type
The result type of the expression please refer to the following table.

### Binary Operation Rules
The following table listed if the difference result types of the input expressions could do this operation and the result type after the operation.
The expression could on the left or right side of the operator. 
**Note**: If the expressions on both sides of the operator are the `TIME_SERIES_VALUES with labels`, they should have the same labels for calculation.

| Expression              | Expression                | Yes/No | ExpressionResultType     |
|-------------------------|---------------------------|--------|--------------------------|
| SINGLE_VALUE            | SINGLE_VALUE              | Yes    | SINGLE_VALUE             |
| SINGLE_VALUE            | TIME_SERIES_VALUES        | Yes    | TIME_SERIES_VALUES       |
| SINGLE_VALUE            | SORTED_LIST/RECORD_LIST   | Yes    | SORTED_LIST/RECORD_LIST  |
| TIME_SERIES_VALUES      | TIME_SERIES_VALUES        | Yes    | TIME_SERIES_VALUES       |
| TIME_SERIES_VALUES      | SORTED_LIST/RECORD_LIST   | no     |                          |
| SORTED_LIST/RECORD_LIST | SORTED_LIST/RECORD_LIST   | no     |                          |

## Aggregation Operation
Aggregation Operation takes an expression and performs aggregate calculation on its results.

Expression:
```text
<Aggregation-Operator>(Expression)
```

| Operator | Definition                                        | ExpressionResultType |
|----------|---------------------------------------------------|----------------------|
| avg      | average the result                                | SINGLE_VALUE         |
| count    | count number of the result                        | SINGLE_VALUE         |
| latest   | select the latest non-null value from the result  | SINGLE_VALUE         |
| sum      | sum the result                                    | SINGLE_VALUE         |
| max      | select maximum from the result                    | SINGLE_VALUE         |
| min      | select minimum from the result                    | SINGLE_VALUE         |

For example:
If we want to query the average value of the `service_cpm` metric, we can use the following expression:

```text
avg(service_cpm)
```

### Result Type
The different operator could impact the `ExpressionResultType`, please refer to the above table.

## Function Operation
Function Operation takes an expression and performs function calculation on its results.

Expression:
```text
<Function-Operator>(Expression, parameters)
```

| Operator | Definition                                                                | parameters                                                         | ExpressionResultType          |
|----------|---------------------------------------------------------------------------|--------------------------------------------------------------------|-------------------------------|
| abs      | returns the absolute value of the result                                  |                                                                    | follow the input expression |
| ceil     | returns the smallest integer value that is greater or equal to the result |                                                                    | follow the input expression |
| floor    | returns the largest integer value that is greater or equal to the result  |                                                                    | follow the input expression |
| round    | returns result round to specific decimal places                           | `places`: a positive integer specific decimal places of the result | follow the input expression |

For example:
If we want to query the average value of the `service_cpm` metric in seconds, 
and round the result to 2 decimal places, we can use the following expression:

```text
round(service_cpm / 60 , 2)
```

### Result Type
The different operator could impact the `ExpressionResultType`, please refer to the above table.

## TopN Operation
TopN Operation takes an expression and performs TopN calculation on its results.

Expression:
```text
top_n(<metric_name>, <top_number>, <order>)
```

`top_number` is the number of the top results, should be a positive integer.

`order` is the order of the top results. The value of `order` can be `asc` or `des`.

For example:
If we want to query the top 10 services with the highest `service_cpm` metric value, we can use the following expression:

```text
top_n(service_instance_cpm, 10, des)
```

### Result Type
According to the type of the metric, the `ExpressionResultType` of the expression will be `SORTED_LIST` or `RECORD_LIST`.

## Relabel Operation
Relabel Operation takes an expression and replace the label values to new label values on its results.

Expression:
```text
relabel(Expression, _='<new_label_value_1>,...')
```

`_` is the new label valuess of the metric after the label is relabeled, the order of the new label values should be the same as the order of the label values in the input expression result.

For example:
If we want to query the `service_percentile` metric with the label values `0,1,2,3,4`, and rename the label values to `P50,P75,P90,P95,P99`, we can use the following expression:

```text
relabel(service_percentile{_='0,1,2,3,4'}, _='P50,P75,P90,P95,P99')
```

### Result Type
Follow the input expression.

## AggregateLabels Operation
AggregateLabels Operation takes an expression and performs aggregate calculation on its `Labeled Value Metrics` results. It aggregates a group of `TIME_SERIES_VALUES` into a single `TIME_SERIES_VALUES`.

Expression:
```text
aggregate_labels(Expression, parameter)
```

| parameter | Definition                                          | ExpressionResultType |
|-----------|-----------------------------------------------------|----------------------|
| avg       | calculate avg value of a `Labeled Value Metrics`    | TIME_SERIES_VALUES   |
| sum       | calculate sum value of a `Labeled Value Metrics`    | TIME_SERIES_VALUES   |
| max       | select maximum value from a `Labeled Value Metrics` | TIME_SERIES_VALUES   |
| min       | select minimum value from a `Labeled Value Metrics` | TIME_SERIES_VALUES   |

For example:
If we want to query all redis command total rate, we can use the following expression(`total_commands_rate` is a metric which recorded every command rate in labeled value):

```text
aggregate_labels(total_commands_rate, SUM)
```

### Result Type
The ExpressionResultType of the aggregateLabels operation is TIME_SERIES_VALUES.
