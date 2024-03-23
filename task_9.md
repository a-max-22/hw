### 1.1. Методы, которые используются только в тестах 
Просматривая свой код, нашел один пример такого стиля - наличие метода `GetCov` в ранее упомянутом в задании № 4 классе `Coverage`, который реализует логику работы с покрытем кода в ходе фаззинга: 
```C++
class Coverage
{
protected:
	CoverageMap _coverage;
	
public:
	Coverage()
	{
	}
	Coverage(const Coverage & coverage)
	{		
		this->_coverage = coverage._coverage;
	}
	Coverage(const Coverage && coverage)
	{	
		this->_coverage = std::move(coverage._coverage);
	}
	
	Coverage(const CoverageMap & coverageMap):_coverage(coverageMap)
	{	
	}
	
	virtual ~Coverage()
	{
	}
	
	// в новом коде просто исключил данный метод
	CoverageMap & GetCov()
	{
		return _coverage;
	}
		
	virtual bool IsEmpty()
	{
		return _coverage.size() == 0;
	}
	
	virtual bool Contains(const Coverage & other)
	{	
		if (other._coverage.size() != _coverage.size())
			return false;
		
		auto thisIter = _coverage.begin();
		auto otherIter = other._coverage.begin();
		for ( ; (thisIter != _coverage.end()) && (otherIter != other._coverage.end()); thisIter++, otherIter++)
		{
			if (*thisIter < *otherIter)
				return false;
		}
		return true;
	}
	
	friend Coverage operator+(const Coverage & l, const Coverage & r)
	{		
		if (l._coverage.size()!=r._coverage.size())
			throw_runtime_error("ASSERT: coverage sizes doesn't match");
		
		Coverage result(l);
		auto lIter = result._coverage.begin();
		auto rIter = r._coverage.begin();
		for ( ; (lIter != result._coverage.end()) && (rIter != r._coverage.end()) ; lIter++, rIter++)
			*lIter = (std::max)(*lIter, *rIter);
		
		return result;
	}
	
	Coverage& operator+=(const Coverage & r)
	{		
		if (this->_coverage.size()!=r._coverage.size())
			throw_runtime_error("ASSERT: coverage sizes doesn't match");
				
		auto lIter = this->_coverage.begin();
		auto rIter = r._coverage.begin();
		for ( ; (lIter != this->_coverage.end()) && (rIter != r._coverage.end()) ; lIter++, rIter++)
			*lIter = (std::max)(*lIter, *rIter);
		
		return *this;
	}
	...
};

```

В итоге код был изменен следующим образом: метод GetCov был убран. Тесты были переписаны так, чтобы не получать сведения о состоянии Coverage через метод GetCov, а использовать уже реализованные операторы сравнения, чтобы проверять корректность поведения. Таким образом, тесты стали более гибкими и приблизились к тому, чтобы тестировать ожидаемое внешнее поведение класса, а не его внутреннюю реализацию.
На одном из этапов написания тесты выглядели таким образом: 
```C++
TEST_F(CoverageTest, EqualityTest)
{	
	Coverage coverage1({1,2,3});
	CoverageMap map({1,2,3});
	EXPECT_EQ(coverage1.GetCov(), map);
};
```
После переработки тесты стали выглядет так (в таком виде они приведены в задании 4): 
```C++
TEST_F(CoverageTest, EqualityTest)
{	
	Coverage coverage1({1,2,3});
	Coverage coverage2({1,2,3});
	EXPECT_EQ(coverage1, coverage2);
	EXPECT_EQ(coverage2, coverage1);	
	EXPECT_FALSE(coverage1!=coverage2);
};
```

### 1.2. Цепочки методов
Нашел подобное в одном из классов, который был создан для работы состоянием в ходе символьного выполнения программы.  
```python
class ExecutionStateManager:
    def __init__(self):
        self.vmInstrTracker = VirtualInstrTracker(vmCharacteristics)
    ...
    
    def isExecutionStoppedDueToControlTransfer(self):
        return self.vmInstrTracker.getCurrentState().wasLastInstructionControlTransfer()

``` 
Цепочка методов реализуется в функции `isExecutionStoppedDueToControlTransfer`. В данном случае данный метод по сути является избыточным, так как вместо него можно возвращать копию состояния объекта `vmInstrTracker`, который предоставляет возможность получить информацию о характеристиках той точки, в которой остановилось выполнение: 
```python
class ExecutionStateManager:
    def __init__(self):
        self.vmInstrTracker = VirtualInstrTracker(vmCharacteristics)
    ...
    
    def getExecutionState(self):
        return self.vmInstrTracker.getCurrentState()

...
execState = manager.getExecutionState()
if execState.wasLastInstructionControlTransfer():
	...
``` 

### 1.3. У метода слишком большой список параметров
```python
...
class VirtualInstructionsTracker(SymExecEventsSubscriber):
    def __init__(self, vmHandlersTablePtr, vmHandlersTableSize, vmContextBase, controlTransferInstrucitons = [], dataFlowVarsOffsets = [], ) -> None:
        super().__init__()
        self.vmHandlers = makeVmInstructionsHandlersTable(vmHandlersTablePtr, vmHandlersTableSize)
        self.controlTransferInstrucitons = frozenset(controlTransferInstrucitons)
        self.dataFlowVarsOffsets = dataFlowVarsOffsets
        self.vmContextBase = vmContextBase
...
```

В данном случае конструктор класса `VirtualInstructionsTracker` (отвечает за отслеживание инструкций виртуальной машины в хоте символьного исполнения кода) принимает большое количество параметров, которые по сути связаны между собой и описывают характеристики анализируемой "виртуальной машины". 

Таким образом, представляется  логичным объединить эти параметры в одну структуру, и передавать уже её в качестве параметра:

```python
class VirtualInstructionsTracker(SymExecEventsSubscriber):
	## ввел вместо множества связанных  параметров структуру VirtualMachineCharacteristics
	## в которой указанные параметры сосредоточены
    def __init__(self, vmCharacteristics:VirtualMachineCharacteristics) -> None:
        super().__init__()
        c = vmCharacteristics
        self.vmHandlers = makeVmInstructionsHandlersTable(c.vmHandlersTablePtr, c.vmHandlersTableSize)
        self.controlTransferInstrucitons = frozenset(c.controlTransferInstrucitons)
        self.dataFlowVarsOffsets = c.dataFlowVarsOffsets
        self.vmContextBase = c.vmContextBase
...
```
### 1.4. Странные решения. Много методов, решающих одни и те же задачи.
Элементы подобного стиля обнаружил в коде, который приводился в решении прошлого задания, а именно в классе ExpressionsConstructor, который отвечает за удобное задание  выражений: 
```python
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
```

В данном случае мы видим, что операторы `__add__`, `__eq__`, `__ne__` хоть и различаются по тому, какую операцию реализуют, внутри выполняют одно и то же действие - проверяют тип аргумента и в зависимости от этого возвращают соответствующий класс. Эту логику можно вынести в отдельный метод или функцию, чтобы регулировать это поведение в единой точке. 
```python

def MakeExpr(arg):
    if isinstance(arg, Expr):
        return arg
    else 
        return ExprId(other, size = 64)

class ExpressionConstructor:
    def __init__(self, symbol):
        self.symbol = MakeExpr(symbol)
    
    def __add__(self, other):
        return self.symbol + MakeExpr(other)

    def __neg__(self):
        return -self.symbol        

    def getExpr(self):
        return self.symbol

...

class ConditionalsMixin:
    def _makeEqExpr(self, other):
        if isinstance(other, Expr):
            eq = ExprOp('==', super().getExpr(), other)
        else:
            eq = ExprOp('==', super().getExpr(), Expr(other, size = 64))
        return eq
         
    def __eq__(self, other):
        return CondConstraintNotZero(self._makeEqExpr(other))

    def __ne__(self, other):
        return CondConstraintZero(self._makeEqExpr(other))
```

В данном случае код, отвечающий за проверку типа аргумента и за создание экземпляра выражения вынесен в отдельную функцию `MakeExpr` и метод `_makeEqExpr`, таким образом устранив предпосылки к возможной рассогласованности, если это поведение будет как-то меняться далее.  
### 1.5. Чрезмерный результат
```python
	def run_state(self, initialState):
	    ...    
	    runner = BlockRunner(machine = Machine("x86_64"), symbolicExecutionEngine = SymbolicExecutionEngine, executionContext = self)
	
	    if initialState is None:
	        initialState = runner.makeBlankState()
	
	    currentState = initialState.copy()
	
	    while currentState is not None:       
	        currentState = runner.runStateUntilDstUnknown(state = currentState, start = None)
	        if currentState is None:
	            break
	
	        if not self.isExecutionStoppedDueToControlTransfer() and currentState is not None:
	            break
	        isDestinationHandled, constraints = callEmulator.handleControlTransfer(currentState)
	        if not isDestinationHandled: break
	        self.addConstraints(constraints)
	
	    modifiedSymbols = getModifiedSymbols(initialState, currentState)
	    return currentState, modifiedSymbols

... 

def run_config(context:SymExecContext, state, config):
    context.registerEmulatedFunctions(makeEmulatedFunctionsDict())
    updatedConstraints = config.makeConstraintsList(context.getSymbolsRef(), context.getRegs())
    context.replaceInitialConstraints([constr.to_constraint() for constr in updatedConstraints])
    state = context.run_state(state, context)
    return context, state

```

В данном случае мы видим, что метод `run_state`, которая отвечает за символьное выполнение блока инструкций, используя некоторое начальное состояние `initialState`. При этом данная функция в конце своего выполнения вычисляет также список символов, которые были модифицированы в результате выполнения блока инструкций. При этом, данный список не применяется вызывающей функцией, а данные вычисления затратны и носят вспомогательный характер для визуализации изменений. 
При этом, изменение состояния может быть вычислено вне функции `run_state`, отдельным вызовом. Поэтому данные вычисления можно вынести за пределы функции `run_state`:
```python
	def run_state(self, initialState):
	    ...    
	
	    while currentState is not None:       
	        currentState = runner.runStateUntilDstUnknown(state = currentState, start = None)
			...
	
            # убрал строку: 
	    # modifiedSymbols = getModifiedSymbols(initialState, currentState)
	    # вычисления измененных символов будут производиться вовне, при необходимости
	    return currentState

... 

def run_config(context:SymExecContext, state, config)
	...
    state = context.run_state(state, context)
    return context, state

# вариант выноса вычислений за пределы метода run_state
def run_config_calc_modified_symbols(context:SymExecContext, initialState, config)
    context, finalState = run_config(context, initialState, config)
    modifiedSymbols = getModifiedSymbols(initialState, finalState)
    return context, finalState, initialState
```
