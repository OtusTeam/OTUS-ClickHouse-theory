## Введение
Существует два типа функций:  
- **Регулярные функции** — применяются к каждой строке отдельно (результат не зависит от других строк).  
- **Агрегатные функции** — накапливают значения из разных строк (зависят от всего набора строк).  

## Regular functions (Обычные функции)

### Особенности
- **Сильная типизация** — нет неявных преобразований между типами. Каждая функция работает с определённым набором типов.  
- **Устранение общих подвыражений** — выражения с одинаковым AST выполняются один раз.  
- **Типы результатов** — функция возвращает одно значение. Тип результата определяется типами аргументов, а не их значениями.  
- **Константы** — некоторые функции работают только с константами для некоторых аргументов (постоянство важно только в пределах одного запроса).  
- **Обработка NULL** — если хотя бы один аргумент NULL, результат NULL (особое поведение можно задать самостоятельно).  
- **Постоянство** — функции не изменяют значения аргументов. Порядок вычисления не имеет значения.  
- **Функции высшего порядка** — принимают лямбда-функции в качестве аргументов. Используется оператор `->` для передачи лямбда-функции.  

### Назначение
Помогают решать ежедневные задачи, раскрывая мощный SQL-диалект ClickHouse. Бывают базовые и сложные функции. Частота использования зависит от знаний и решаемой задачи.  

### Категории функций
Arithmetic, Arrays, arrayJoin, UDF, Bit, Bitmap, Comparison, Conditional, Dates and Times, Dictionaries, Distance, Embedded Dictionaries, Geo, Encoding, Encryption, Files, Hash, IN Operator, IP Addresses, Introspection, JSON, Logical, Machine Learning, Maps, Mathematical, NLP (experimental), Nullable, Other, Random Numbers, Replacing in Strings, Rounding, Searching in Strings, Splitting Strings, Strings, Time Series, Time Window, Tuples, Type Conversion, ULID, URLs, UUIDs, uniqTheta.

## Aggregate functions (Агрегатные функции)
Функции, которые получают результирующее значение путём вычисления на множестве значений. В ClickHouse работают обычным образом, как ожидается.  

### Категории агрегатных функций
- **Стандартные**: count, min, max, sum, avg, stddevPop, stddevSamp, varPop, varSamp, covarPop, covarSamp, any, anyLast, argMin, argMax, avgWeighted, groupArray, groupUniqArray, groupArrayInsertAt, groupArrayMovingAvg, groupArrayMovingSum, sumWithOverflow, sumMap, minMap, maxMap.  
- **Специфичные для ClickHouse**: anyHeavy, topK, topKWeighted, groupBitAnd, groupBitOr, groupBitXor, groupBitmap, groupBitmapAnd, groupBitmapOr, groupBitmapXor, uniq, uniqExact, uniqCombined, uniqCombined64, uniqHLL12, uniqTheta, quantile, quantiles, quantileTiming, quantileTimingWeighted, quantileDeterministic, quantileTDigest, quantileTDigestWeighted, quantileBFloat16, quantileBFloat16Weighted, simpleLinearRegression, stochasticLinearRegression, stochasticLogisticRegression, categoricalInformationValue, skewPop, skewSamp, kurtPop, kurtSamp, sequenceMatch, sequenceCount, windowFunnel, retention, uniqUpTo, sumMapFiltered, sumMapFilteredWithOverflow, sequenceNextNode.  

### Параметрические агрегатные функции
Могут принимать не только столбцы аргументов, но и набор параметров-констант для инициализации. Синтаксис: две пары скобок (первая — для параметров, вторая — для аргументов).  
Примеры: histogram, sequenceMatch(pattern)(timestamp, cond1, cond2, ...), sequenceCount(pattern)(time, cond1, cond2, ...), windowFunnel, retention, uniqUpTo(N)(x), sumMapFiltered, sumMapFilteredWithOverflow, sequenceNextNode.  

### Комбинаторы агрегатных функций
К имени агрегатной функции может быть добавлен суффикс, изменяющий принцип её работы.  

### Grouping: ROLLUP и CUBE
Модификаторы GROUP BY для вычисления промежуточных итогов.  
- **ROLLUP** — вычисляет промежуточные итоги на каждом уровне агрегирования по упорядоченному списку столбцов.  
- **CUBE** — вычисляет промежуточные итоги по всем возможным комбинациям указанных столбцов.  

## UDF (User Defined Functions)

### Executable User Defined Functions
ClickHouse может вызывать внешнюю исполняемую программу или скрипт для обработки данных.  
- Конфигурация размещается в XML-файлах, путь указывается в параметре `user_defined_executable_functions_config`.  
- Команда должна читать аргументы из STDIN и выводить результат в STDOUT, обрабатывая аргументы итеративно.  

#### Пример конфигурации XML
```xml
<functions>
  <function>
    <type>executable</type>
    <name>test_function_python</name>
    <return_type>String</return_type>
    <argument>
      <type>UInt64</type>
      <name>value</name>
    </argument>
    <format>TabSeparated</format>
    <command>test_function.py</command>
  </function>
</functions>
```

#### Пример скрипта Python
```python
#!/usr/bin/python3
import sys
if __name__ == '__main__':
    for line in sys.stdin:
        print("Value " + line, end='')
        sys.stdout.flush()
```

### SQL User Defined Functions
Создаются из лямбда-выражения. Выражение может включать параметры функции, константы, операторы или другие вызовы функций.  

#### Ограничения
- Имя функции должно быть уникальным среди пользовательских и системных функций.  
- Рекурсивные функции не допускаются.  
- Все переменные должны быть указаны в списке параметров.  

#### Примеры
```sql
CREATE FUNCTION linear_equation AS (x, k, b) -> k*x + b;
SELECT number, linear_equation(number, 2, 1) FROM numbers(3);

CREATE FUNCTION parity_str AS (n) -> if(n % 2, 'odd', 'even');
SELECT number, parity_str(number) FROM numbers(3);
```
