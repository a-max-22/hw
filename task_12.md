### Избавиться от точек генерации исключений 
#### Пример 1. 
Рассмотрим следующий пример:  
```python
class FixedArray:
	def __init__(self, size):
		if (size <= 0):
			raise ValueError("Invalid array size")
		...
```

Здесь в случае передачи неправильного параметра в конструктор, возникает исключение.  Для того, чтобы от него избавиться возможно создавать структуру данных с валидным, заданным по умолчанию параметром. Однако, чтобы не прятать ошибку, можно создавать объект данного класса в отдельной функции: 
```python
class FixedArray:
    @classmethod
    def create_fixed_array(fixed_array_class, size):
        if (size <= 0):
            return None
        else:
            return fixed_array_class(size)

	def __init__(self, size):
		self.size = size
		self.storage = []
		...
```

#### Пример 2. 
Рассмотрим для примера следующий класс
```python
class SymExecEngine:
	def __init__(self, state):
		self.dest_key = 'dst'
		self.state  = state
		...
		
	def set_state(self, new_state):
		self.state = new_state
		...

	def get_state(self, new_state):
		return self.state
		...

	# запустить символьное выполнение блока
	def run(self):
		new_state, next_destination = exec_block(state)
		if (next_destination is None):
			return False
		new_state[self.dest_key] = next_destination
		self.state = new_state
		return True

	# вычислить следующий адрес для выполнения
	# на основании дополнительных сведений 
	# и ограничений, полученных в процессе исполнения 
	def resolve_destination(self, constraint):
		if (self.dest_key not in self.state):
			raise RuntimeError("No destination in current state")
		resolved_dst = resolve_constraints(self.state[self.dest_key], constraint)
		if resolved_dst is None:
			return False
		
		self.state[self.dest_key] = resolved_dst
		return True
	...
```
В данном случае,  движок символьного исполнения предоставляет возможность вычислить следующий адрес для выполнения, с учетом дополнительных ограничений/условий или значений, которые вызывающая сторона может ему предоставить. При этом, валидная последовательность вызовов выглядит так:
```python

engine = SymExecEngine(state)
while (not stop_condition):
	if (engine.run()):
		continue 
	# если здесь вызывать engine.set_state(new_clean_state)
	# возникнет исключение, так как в новом состоянии не будет ключа 'dst'
	constraints = context.get_constraints()	
	if (not engine.resolve_destination(constraints)):
		break

```

Однако, если вызвать метод `set_state` между `run` и `engine.resolve_destination(constraints)` до вызова `run` у нас возникнет исключение, так как в данном случае новое  состояние может не содержать элемента с ключом `dst`.  При этом, мы не можем заменить `dst` на какое-то значение по умолчанию, так как это усугубит ситуацию - будет выполнен код  по неизвестному адресу, что скроет ошибку, но не устранит её. 
Возможное решение -  сделать  метод `resolve_destination` внутренним, добавив дополнительные аргументы в метод `run()`:
```python
def run(self, constraints):
		new_state, next_destination = exec_block(state)
		if (next_destination is None):
			resolved_dst = self.resolve_destination(constraints)
			...
		new_state[self.dest_key] = next_destination
		self.state = new_state
		return True
```
В таком случае, правильная последовательность вызовов будет гарантирована. 

### Отказаться от конструкторов без параметров
#### Пример 1. 
В данном случае класс SymExecContext конструируется с помощью методов `add*`, которые добавляют к нему различные атрибуты: 
```python
class SymExecContext:
	def __init__(self):
		...
	
	def add_machine(self, machine)
		self.machine = machine

	def add_engine(self, engine)
		self.engine = engine

	def execute(self, addr)
		# исключение, если метод add_machine не вызван
		self.engine.run(self.machine)
		...
```
Данное временное решение, можно заменить на передачу параметров через конструктор, и создавать новый экземпляр, в случаях, когда надо менять составляющие класса: 
```python
class SymExecContext:
	def __init__(self, machine, engine):
		self.machine = machine
		self.engine = engine
	...
```

#### Пример 2. 
Аналогичный пример: 
```C++
class DynamoRioLauncher : public Launcher
{
private:
	std::string &_dynamoRioDir;
	std::string &_instrumentationLibPath;
public:
	DynamoRioLauncher(std::string &dynamoRioDir, std::string &instrumentationLibPath): _dynamoRioDir(dynamoRioDir), 
	_instrumentationLibPath(instrumentationLibPath)
	{
		...
	};
	DynamoRioLauncher(): _dynamoRioDir(''), _instrumentationLibPath('')
	{
		...
	};
}
```

Если инициализировать класс DynamoRioLauncher с конструктором по умолчанию, будет необходимо устанавливать валидные параметры дополнительными методами. В этом случае также лучше избавиться от конструктора по умолчанию. 
### Избегать увлечения примитивными типами данных 
#### Пример 1. 
В качестве примера использования примитивных типов данных можно привести такой код: 
```C++
class DynamoRioLauncher : public Launcher
{
	...
	virtual RunResultPtr RunOnce(vector<char> & testCase) override
	{
		...
	}
}
```
В данном примере в коде фаззера  исходные данных, поступающих на вход тестируемого приложения (параметр `testCase`), передаются как массив байт (`vector<char>`). Хоть данный тип и не является примитивным в плане простоты, однако он является таковым в контексте приложения, так как находится на более низком уровне абстракции. Впоследствии данный для описания тесткейсов был введен отдельный тип `TestCase`: 
```C++
typedef vector<char> TestCase;
	...
class DynamoRioLauncher : public Launcher
{
	...
	virtual RunResultPtr RunOnce(TestCase & testCase) override
	{
		...
	}
}
```
Хотя его  внутреннее содержание не поменялось, мы можем теперь более гибко менять его структуру данных, которая его реализует.  

#### Пример 2. 
В качестве другого примера использования  примитивных  типов  данных можно привести следующую выдержку из  кода  рассмотренного ранее класса `DynamoRioLauncher`, который отвечает за запуск тестируемого приложения с использованием инструмента бинарной инструментации `DynamoRio`.    
```C
class DynamoRioLauncher : public Launcher
{
	...
	virtual RunResultPtr RunOnce(TestCase & testCase) override
	{
		...
		char result = targetClient->RecvTargetCommand();
		if (result == 'K')
			result = targetClient->RecvTargetCommand();

		if (result == 0)
		{
			_launcher->StopApp();
			hangOccured = true;
			break;
		}
		
		if (result != 'P')
			throw_runtime_error("Unexpected result from pipe! expected 'P', instead received " + result);
			
		targetClient->SendTargetCommand(startRun);

		result = targetClient->RecvTargetCommand();
		if (result == 'K') break;

		if (result == 'C')
		{
			_launcher->StopApp();
			crashOccured = true;
			break;
		}
		_launcher->StopApp();
		hangOccured = true;
	}
	...
};
```
В данном случае, общение с тестируемым приложением происходит через именованный канал, который возвращает статус  выполнения приложения, который представляет собой символ 'K', 'C', 'P' и.т.д. Впоследствии данный код был переписан, для статусов выполнения был создан отдельный тип-перечисление `TargetExecStatusAndCommands` и код принял такой вид: 
```C
typedef enum TargetExecStatusAndCommands
{
	FinishedRun = 'K';
	ReadyToRun = 'P';
	CrashOccured = 'C';	
	StartRun = 'F';
	ExitTareget = 'Q';
	TargetNotResponded = 0;
} TargetExecStatusAndCommands;


class DynamoRioLauncher : public Launcher
{
	...
	virtual RunResultPtr RunOnce(TestCase & testCase) override
	{
		...
		TargetExecStatusAndCommands status = targetClient->RecvTargetCommand();
		if (result == FinishedRun)
			result = targetClient->RecvTargetCommand();

		if (result == TargetNotResponded)
		{
			_launcher->StopApp();
			hangOccured = true;
			break;
		}
		
		if (result != ReadyToRun)
			throw_runtime_error("Unexpected result from pipe! expected 'P', instead received " + result);
			
		targetClient->SendTargetCommand(startRun);

		result = targetClient->RecvTargetCommand();
		if (result == FinishedRun) break;

		if (result == CrashOccured)
		{
			_launcher->StopApp();
			crashOccured = true;
			break;
		}
		_launcher->StopApp();
		hangOccured = true;
	}
	...
};
```

Видно, что его читаемость повысилась, стал более понятен смысл. Теперь также можно легче заменить/расширить перечень поддерживаемых комманд и статусов. 

