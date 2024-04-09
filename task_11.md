### 2. Уровень классов
#### 2.1. Класс слишком большой
В качестве примера можно привести класс `SymExecContext`, который по сути является одним большим состоянием, которое передается из функции в функцию в ходе символьного исполнения:  
```python
class SymExecContext:
	def __init__(self, inputSymbols = {}, constraints = [], startAddress = None):
		...
	def setImportSymbolsForState(self, state = None):
		...
	def registerEmulatedFunctions(self, emulatedSymbols):
		...
	def getCallEmulator(self):
		...
	def getSymbolsRef(self):
		...
	def getRegs(self):
		...
	def replaceInitialConstraints(self, constraints):
		...
	def addConstraints(self, constraints):
		...
	def save(self, baseFilePath):
		...
	def restore(self, baseFilePath):
		...
```
На текущей стадии у нас уже имеется достаточно большой набор методов, который возвращает элементы состояния: регистры (`getRegs`), символы  (`getSymbolsRef`),  реализует методы работы с ограничениям (`replaceInitialConstraints`, `addConstraints`). И данный класс имеет тенденцию к разрастанию в случае  необходимости работы с иными элементами контекста. Проблема в том, что в этом классе во-первых хранится некоторое глобальное состояние и имеются методы для сохранения/изменения отдельных элементов этого состояния/контекста. Решение видится в том, чтобы оставить необходимые элементы `save`/`restore`, а отдельные элементы контекста предоставлять по какому-то атрибуту, например по имени. В этом случае, данный класс отвечает только за  хранение состояния, не зная о том, какие элементы в нем имеются, что отвечает его прямому назначению. При этом, логично его стоит переименовать в `SymExecState`.  

#### 2.2. Класс слишком маленький
```python
class ExpressionSimplifierZ3:
	def __init__(self):
		pass

	def run(self, expr):
	    z3expr = Translator.to_language('z3').from_expr(expr)
	    z3exprSimp = z3.simplify(z3expr)
	    return z3exprSimp
```
Данный класс упрощает выражение и возвращает его формате солвера z3. Однако по сути это поведение не нужно выделять в отдельный класс достаточно лишь функции, которая выполняет это действие. 

#### 2.3. В классе есть метод, который выглядит более подходящим для другого класса. 
Подобное поведение можно наблюдать в приведенном выше классе `SymExecContext`:
```python
class SymExecContext:
	...
	def setImportSymbolsForState(self, state = None):
		...
	def registerEmulatedFunctions(self, emulatedSymbols):
		...
	...
```
Выделенные методы не должны  относиться к данному классу, который отвечает за хранение (сохранение/восстановление)  состояния символьного исполнения. Указанные методы относятся к компоненту, который реализует эмуляцию импортируемых функций и api-вызовов, что никак не относится к области ответственности класса SymExecContext.    
#### 2.4. Класс хранит данные, которые загоняются в него в множестве разных мест в программе. 
В данном случае в дополнение к вышеупомянутым методам `setImportSymbolsForState`, `registerEmulatedFunctions` можно добавить так же следующие методы: 
```python
class SymExecContext:
	...
	def replaceInitialConstraints(self, constraints):
		...
	def addConstraints(self, constraints):
	...
```
Таким образом, мы имеем 4 метода, которые позволяют изменить внутреннее состояние класса, что по факту и происходит в разных местах программы. Хотя, данный класс в принципе подразумевает возможность изменения состояния, полагаю, что имеет смысл данное поведение сделать в некотором декларативном стиле (в дополнение к обозначенному в пункте 2.1. решению). Т.е. когда ты хочешь явно изменить состояние, необходимо вызвать конструктор, который это сделает например на основе старого состояния и новых измененных  элементов этого состояния.   
#### 2.5. Класс зависит от деталей реализации других классов.
Рассмотрим следующий пример:   
```python
class CallEmulator:
	...
	def saveCallCounters(self, fileName):
        with open(fileName, "wb") as outfile:
            pickle.dump(self.handlerCallsCounters, outfile)
	...

class SymExecContext:
	def save(self, fileName):
		...
		self.callEmulator.saveCallCounters(self.callCountersFileName)
```
В данном случае класс `SymExecContext` зависит деталей того, каким образом сохраняются счетчики количества вызовов функций, которые поддерживаются классом `CallEmulator`. Т.е. вне зависимости от возможных способов сохранения внутреннего состояния, мы можем сохранить  указанные счетчики только в файл. Наиболее приемлемым вариантом кажется отделить логику сохранения от класса `CallEmulator`, использовав например visitor или mixin, либо передавать в метод  `saveCallCounters` отдельный класс, который скрывает в себе логику сериализации (что по сути  является реализацией того же "посетителя").    

#### 2.6. Приведение типов вниз по иерархии (родительские классы приводятся к дочерним).
```python

class Sequence:
	...
	
class ConstrainedSequence:
	...
	def getConstraints(self) 
		...
	
class SequenceOperation:
	...
	def run(self, sequence:Sequence):
		...
	
class ConstrainedSequenceOperation:
	def run(sequence:ConstrainedSequence):
		if isinstance(sequence,ConstrainedSequence):
			constraints = sequence.getConstraints()
			# perform operation
			...
		else:
			raise ValueError("Expected constrained sequence")
 
```

В данном случае хоть класс `ConstrainedSequenceOperation` имеет метод `run`, который должен принимать, согласно своему прототипу любой объект типа `Sequence`. При этом `ConstrainedSequenceOperation` работает только с последовательностями `ConstrainedSequence` ввиду того, что ему нужен метод `getConstraints`, которого нет в родительском классе. В данном возможность выполнения операции над Sequence стоит добавить в интерфейс самого класса, передавая функцию в качестве параметра, либо добавить такую функциональность через миксин, если изменить интерфейс не представляется возможным. 

#### 2.7. Когда создаётся класс-наследник для какого-то класса, приходится создавать классы-наследники и для некоторых других классов.

Возьмем для примера такой код: 
```python
class Module:
	...

class PeModule(Module):
	...

class ElfModule(Module):
	...

class ModuleParser:
	...

class PeModuleParser(Module):
	...

class ElfModuleParser(Module):
	...
```
В данном случае у нас есть две параллельные иерархии: наследники от класса `Module` и наследники от класса `ModuleParser`. При этом, при добавлении поддержки нового формата модулей, например `Mach-O`, в иерархию необходимо добавлять также класс `MachOModuleParser`. 
В данном случае класс `ModuleParser` отвечает за построение соответствующего экземпляра класса `Module` на основе "сырых" данных (дампа памяти, файла, и.т.д.). При этом, инстанцировать пустой экземпляр класса `Module` не имеет практического смысла. Поэтому логику парсинга можно внести в конструктор соответствующего класса: 
```python
class Module:
	def __init__(self, stream):
		pass 

class PeModule(Module):
	def __init__(self, stream):
		self.moduleAttributes = ParsePeModule(stream) 
		...

class ElfModule(Module):
	def __init__(self, stream):
		self.moduleAttributes = ParseElfModule(stream)
		...
```

#### 2.8. Дочерние классы не используют методы и атрибуты родительских классов, или переопределяют родительские методы.

Рассмотрим приведенную в пункте 2.7 иерархию классов, с методом  `serialize`, который в  подклассах  `PeModule` и `ElfModule` сериализует их содержимое в файл с помощью библиотеки `pickle` : 
```python
class Module:
	...
	def serialize(self):
		...

class PeModule(Module):
	...
	def serialize(self, fileName):
		pickle.dump(self.attributes, fileName)

class ElfModule(Module):
	...
	def serialize(self, fileName):
		pickle.dump(self.attributes, fileName)

```

В данном случае переопределяется метод `serialize` базового класса, который изначально не предполагал, что в его протоип будут включаться сведения о способе сериализации. К тому же, если мы хотим добавить возможность сериализации например через protobuf, может возникнуть соблазн добавить это поведение путем наследования, например создав класс `ProtobufSerializablePeModule`, в методе `serialize` которого определить нужную логику. Для того, чтобы избежать подобного представляет логичным использовать миксины, чтобы гибко добавлять разные способы сериализации, не затрагивая исходные классы.  

### 3. Уровень приложения
#### 3.1. Одна модификация требует внесения изменений в несколько классов.

```python
class BlockRunner: 
	...
	def save(self, fileNameToSaveState):
		runState = {}
		runState['runner_state'] = self.symbolicState
		runState['strategy'] = self.strategy
		with open(fileNameToSaveState, "wb") as outfile:
			pickle.dump(runState, outfile)    

class CallEmulator:
	...
	def saveCallCounters(self, fileName):
        with open(fileName, "wb") as outfile:
            pickle.dump(self.handlerCallsCounters, outfile) 

class SymExecContext:
	...
    def save(self):
        with open(fileNameToSaveConstraints, "wb") as outfile:
            pickle.dump(self.addedConstraints, outfile)

```
В вышеприведенном примере способ сохранения состояния - сохранение в файле в формате  библиотеки pickle  размазан по многим классам, вследствие чего необходимость изменения этого способа влечет необходимость менять код всех затронутых классов.   
#### 3.2. Использование сложных паттернов проектирования там, где можно использовать более простой и незамысловатый дизайн.
Рассмотрим пример ниже: 
```python
class DefaultExecutionStrategy:
    def __init__(self) -> None:
        self.nextDst = None

    def needToContinueExecution(self):
        return False

    def getNextAddressToExecute(self):
        return self.nextDst

    def onBlockExecuted(self, state):
        dst = getDestinationFromState(state)
        self.nextDst = dst


class BlockRunner:
	...
	def run_by_execution_strategy(self, executionStrategy, max_blocks_to_execute = 90000):
		nextDst = initialAddress

		for counter in range(max_blocks_to_execute):
			irblock = self.ircfg.get_block(nextDst)
			if irblock is not None:
				self.symExecEngine.eval_updt_irblock(irblock, step=False)
				executionStrategy.onBlockExecuted(self.symExecEngine.symbols)
				if not executionStrategy.needToContinueExecution():
					break
				nextDst = executionStrategy.getNextAddressToExecute()
			
			...
```

В данном случае мы имеем класс, `BlockRunner`, который отвечает за символьное исполнение базового блока. При этом, его поведение при выполнении базового блока определяется классом   `ExecutionStrategy`, в котором обрабатывается текущий базовый блок (например - вычисляется адрес, который будет выполняться  следующим). По сути здесь в некотором виде применяется паттерн Observer. Однако в данном случае это усложняет структуру. В данном случае здесь можно обойтись без зависимости BlockRunner от ExecutionStrategy: 

```python
runner = BLockRunner(...)
strategy = ExecutionStrategy()
dest = initialAddress
while (strategy.needToContinueExecution(runner.getState()))
	runner.run_block(dest)
	executionStrategy.handleExecutionState(runner.getState())
	dest = executionStrategy.getNextAddressToExecute(runner.getState())
```
