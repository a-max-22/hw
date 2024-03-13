### Реализация Миксинов в Python. 
Использование миксинов в Python синтаксически  аналогично использованию множественного наследования: 
```python
class DerivedClass(Mixin1, Mixin2, BaseClass)
	pass
```
Особенность миксина состоит в том, что он инстанцируется отдельно, и не используется изолированно. Как правило, этот класс реализует некоторое свойство, которое мы можем добавлять ("подмешивать") к другим классам, которые, в том числе, могут быть не связаны между собой. Из особенностей использования можно подчеркнуть важность порядка перечисления миксинов в определении класса, в случае, если миксины реализуют функции с одинаковыми именами. 
### Простейший учебный пример 
Простейший учебный  пример использования миксинов, который я использовал для того, чтобы понять суть этого приёма, приведен ниже: 

```python

class Container:
	def __init__(self):
		pass

	def GetDataCopy(self):
		 pass

	def AddData(self, newData):
		 pass
		

class AccumulatingContainer(Container):
	def __init__(self):
		self.data = bytearray()
	
	def GetDataCopy(self):
		 self.data.copy()

	def AddData(self, newData):
		 self.data.extend(newData)


class ReplacingContainer(Container):
	def __init__(self):
		self.data = bytearray()
	
	def GetDataCopy(self):
		 self.data.copy()

	def AddData(self, newData):
		 self.data = bytearray(newData)

class Base64EncodeableMixin:
	def encode(self):
		return base64.b64encode(super().GetDataCopy())

class EncodeableAccumulatingContainer(Base64EncodeableMixin, AccumulatingContainer)
	pass
	
class EncodeableReplacingContainer(Base64EncodeableMixin, ReplacingContainer)
	pass

```
В данном случае у нас имеются некоторые контейнеры, которые реализуют различные стратегии добавления новых данных - один из них накапливает все вновь добавляемые данные, другой - заменяет старые данные на вновь добавленные. Предположим, что у нас возникла необходимость расширить функциональность контейнера возможностью предоставлять данные в закодированном виде. Это можно сделать с помощью миксина `Base64EncodeableMixin`, который добавляет дополнительный метод encode, в котором реализует необходимые действия по кодированию содержимого. 
Возможно, это не совсем оптимальный вариант реализации возможности кодирования содержимого контейнера. Ведь строго говоря нам ничего не мешает просто получить от него  актуальные данные и передать их в функцию кодирования, применив по сути простую композицию функций. Однако, в качестве некоторой демонстрации данный код вполне может подойти. 

### Практический пример
На практике применил миксины в следующем варианте. В движке символьного исполнения miasm, который , имеется класс `Expr`, который представляет собой некоторое выражение, получаемое в результате символьного исполнения. Есть также класс `Constraint` - ограничения на значение  этого выражения. С использованием солвера z3 можно проверять, выполняется оно для данного выражения или нет. 
Для удобства задания выражений вручную мной применялся класс  `ExpressionConstructor`, который позволяет конструировать выражения в более удобной форме, применяя вместо громоздких конструкций типа `sum = ExprOp('+', ExprId('RAX', 64) , ExprId('RBX', 64))` выражения: 
```
rax = ExpressionConstructor('RAX')
sum = rax + 64
```

Со временем возникла необходимость генерировать ограничения с использованием такого же компактного синтаксиса, наподобие такого: 

```
rax = ExpressionConstructor('RAX')
constraint = ( (rax + 64) != 0 )

```

Для этого надо было расширить ExpressionConstuctor новыми операторами `__ne__` и `__eq__`, для чего я и решил применить миксины. В итоге получился следующий код, задача была решена:  

```python
import miasm.expression.expression as m2_expr

class CondConstraint(object):

    """Stand for a constraint on an Expr"""

    # str of the associated operator
    operator = ""

    def __init__(self, expr):
        self.expr = expr

    def __repr__(self):
        return "<%s %s 0>" % (self.expr, self.operator)

    def to_constraint(self):
        """Transform itself into a constraint using Expr"""
        raise NotImplementedError("Abstract method")


class CondConstraintZero(CondConstraint):

    """Stand for a constraint like 'A == 0'"""
    operator = m2_expr.TOK_EQUAL

    def to_constraint(self):
        return m2_expr.ExprAssign(self.expr, m2_expr.ExprInt(0, self.expr.size))



class CondConstraintNotZero(CondConstraint):

    """Stand for a constraint like 'A != 0'"""
    operator = "!="

    def to_constraint(self):
        cst1, cst2 = m2_expr.ExprInt(0, 1), m2_expr.ExprInt(1, 1)
        return m2_expr.ExprAssign(cst1, m2_expr.ExprCond(self.expr, cst1, cst2))


class ExpressionConstructor:
    def __init__(self, symbol):
        if isinstance(symbol, Expr):
            self.symbol = symbol

        self.symbol = ExprId(symbol, size = 64)
    
    def __add__(self, other):
        if isinstance(other, Expr):
            return self.symbol + other
        return self.symbol + ExprId(other, size = 64)

    def __neg__(self):
        return -self.symbol        

    def getExpr(self):
        return self.symbol

...

class ConditionalsMixin:
    def __eq__(self, other):
        if isinstance(other, Expr):
            eq = ExprOp('==', super().getExpr(), other)
        else:
            eq = ExprOp('==', super().getExpr(), Expr(other, size = 64))
        return CondConstraintNotZero(eq)

    def __ne__(self, other):
        if isinstance(other, Expr):
            eq = ExprOp('==', super().getExpr(), other)
        else:
            eq = ExprOp('==', super().getExpr(), Expr(other, size = 64))
        return CondConstraintZero(eq)

class ConditionalExprConstructor(ConditionalsMixin, ExpressionConstructor):
    pass

...

rax = ConditionalExprConstructor('RAX')
constraint1 = (rax != 0)
constraint2 = (rax == 0)

rax_ = ExprConstructor('RAX')
# ошибка, данный оператор не реализован 
constr1 = (rax_ != 0)
constr2 = (rax_ != 0)

```

### Вывод
Миксины представляют собой интересный и удобный инструмент для расширения функциональности уже имеющихся классов. Особенно это актуально, когда ты работаешь с довольно большой устоявшейся кодовой базой. Это позволяет довольно безопасно дополнять её нужным функционалом и аккуратно тестировать его.  
