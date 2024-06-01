### Введение
В этом отчете объединены несколько примеров, выделенных в результате переработки  скриптов для анализа двоичного кода, которая была сделана в рамках и с учетом статьи "Семь проектных ошибок", а также предыдущих занятий. Данные примеры сформированы на основе рефакторинга единственного компонента, который отвечает за подсчет  параметров в вызываемой функции и показывает его в развитии - решил, что это будет достаточно наглядно, прежде всего для меня, при возвращении к своим заметкам позже.  

### Пример 1
Рассмотрим в данном примере компонент, который на основе анализа регистров, которые записываются перед вызовом функции, вычисляет количество её параметров:
```python
...

class FuncParametersAnalyser:
	def __init__(self):
		self.funcs_prototypes = {}
		
	def analyse_block(self, block):
		last_instr = block.instrs[-1]
		
		if not last_instr.is_subcall():
			return None, None
		
		called_func_addr = last_instr.get_call_dst()
		if called_func_addr in self.funcs_prototypes:
			return None, None

		written_regs = machine.X86_64.regs.copy()
		
		for instr in block.instrs[-1::-1]
			if instr.type != machine.X86_64.MOV and instr.type != machine.X86_64.LEA: continue
			written_regs[instr.op1] = 1
		
		num_params = count_params(written_regs)
		return called_func_addr, num_params
		

	def analyse_func(self, addr):
		blocks = disasm_blocks(addr)
		if blocks is not None:
			return
			
		for block in blocks:
			called_func_addr, num_params = self.analyse_block(block)
			if called_func_addr is None:
				contunue
			self.funcs_prototypes[called_func_addr] = num_params
	
...
```
В первоначальной реализации данный класс получает на вход адрес функции, хотя технически это может быть адрес любого базового блока с инструкциями. Далее  выполняется анализ всех базовых блоков, достижимых из указанного (с использованием инструкций условного и безусловного перехода), по следующей схеме: если  последняя инструкция базового блока - `call`, то выполняется анализ предшествующих инструкций, на предмет того, какие регистры записываются. На основании этой информации делается вывод о количестве параметров вызываемой функции.  

Одним из аспектов, подлежащих содержательному рефакторингу является поведение данного компонента, который получает параметры только тех функций, которые были вызваны в исходной, адрес которой поступил в качестве параметра к методу  `analyse_func`. 
В целом для решения задач на текущем этапе, этого хватает в полной мере, однако в дальнейшем могут понадобиться иные его применения. Поэтому, здесь можно видоизменить код, отделив явно логику отбора функций для анализа и непосредственно логику анализа параметров. После рефакторинга код принимает такой вид:    
```python

...	
	
def count_func_params_in_caller_block(prevent_analysis_predicate, block):
	last_instr = block.instrs[-1]
		
	if not last_instr.is_subcall():
		return None, None
		
	called_func_addr = last_instr.get_call_dst()
	if not prevent_analysis_predicate(called_func_addr):
		return None, None

	written_regs = machine.X86_64.regs.copy()
		
	for instr in block.instrs[-1::-1]:
		if instr.type != machine.X86_64.MOV and instr.type != machine.X86_64.LEA: continue
        written_regs[instr.op1] = 1
	
    num_params = count_params_stdcall(written_regs)
	return called_func_addr, num_params


class FuncParametersCounter:
	def __init__(self, params_count_func):
		self.funcs_params_count = {}
        self.params_count_func = params_count_func
        self.prevent_analysis_predicate = lambda x: x not in self.funcs_params_count

    def count_func_params(addr):
        pass

    def get_params_counts(self):
        self.funcs_params_count


class SubacallsParametersCounter(FuncParametersCounter):        
	def count_func_params(addr):
		blocks = disasm_blocks(addr)
		if blocks is not None:
			return
			
		for block in blocks:
			called_func_addr, prototype = self.params_count_func(self.prevent_analysis_predicate, block)
			if called_func_addr is None:
				contunue
			self.funcs_params_count[called_func_addr] = prototype


class FuncParametersCounterBySingleRef(FuncParametersCounter):
    def count_func_params(addr):
        references = get_references_to_addr(addr)
        referenced_code_blocks = get_code_blocks_by_refs(references)
        if len(referenced_code_blocks) == 0:
            return None
       	
        called_func_addr, prototype = self.params_count_func(self.prevent_analysis_predicate, block)
		if called_func_addr is None:
			contunue
		self.funcs_params_count[called_func_addr] = prototype
...
```
Здесь, в измененной версии в  дополнение добавлен еще один класс `FuncParametersCounterBySingleRef`, который считает количество параметров в самой функции, адрес которой передается в к вызову `get_params_counts()`. 

Use cases  данных классов выглядит следующим образом (взято из Unit-тестов): 
```python
...
def TestParamCounterBySingleRef(self):
	paramCounter = FuncParametersCounterBySingleRef(count_func_params_in_caller_block)
	paramCounter.count_func_params(self.test_func_addr)
	result = paramCounter.get_params_counts()
	self.assertEqual(result[self.test_func_addr], self.test_func_params)

...
def TestSubcallsParamsCounter(self):
	paramCounter = SubacallsParametersCounter(count_func_params_in_caller_block)
	paramCounter.count_func_params(self.test_func_addr)
	result = paramCounter.get_params_counts()
	self.assertEqual(result, self.test_func_subcall_params)
```
  

### Пример 2

Данный пример является продолжением примера № 1, исходный код здесь соответствует коду после рефакторинга. 

Продолжая анализировать получившийся компонент дальше, а также пытаясь дать классам более ясные имена,  пришел к выводу, что непосредственно  вычисление количества параметров функций и хранение результатов вычисления - это по сути разные и не зависящие друг от друга операции.  Таким образом, представляется  логичным отделить поведение сохранения состояния от непосредственно вычисления функций. Без значительного изменения прототипа функции вычисления количества параметров, это можно сделать добавлением функции обратного вызова, которая будет обрабатывать результаты, например сохранять их:  

```python

class FuncParametersCounter:
	def __init__(self, params_count_func, prevent_analysis_predicate = lambda x: False, process_result_callback = lambda x,y: (x,y)):
        self.params_count_func = params_count_func 
        self.prevent_analysis_predicate = prevent_analysis_predicate
        self.process_result_callback = process_result_callback 


class SubcallsParametersCounter(FuncParametersCounter):
    def count_func_params(addr):
		blocks = disasm_blocks(addr)
        if blocks is not None:
			return  
	
		for block in blocks:
			called_func_addr, num_params = self.params_count_func(self.prevent_analysis_predicate, block)
            self.process_result_callback(called_func_addr, num_params)
        
        return result


class FuncParametersCounterBySingleRef(FuncParametersCounter):
    def count_func_params(addr):
        references = get_references_to_addr(addr)
        referenced_code_blocks = get_code_blocks_by_refs(references)
        if len(referenced_code_blocks) == 0:
            return None, None
       	
        called_func_addr, num_params = self.params_count_func(self.prevent_analysis_predicate, block)
        self.process_result_callback(called_func_addr, num_params)

```
Использование данных классов видоизменится следующим образом: 
```python
def TestParamCounterBySingleRef(self):
    analysis_res = {}
    save_func = lambda addr,num_counts : analysis_res[addr] = num_counts  
	prevent_analysis_predicate = lambda addr: addr in analysis_res
    
    paramCounter = FuncParametersCounterBySingleRef(count_func_params_in_caller_block, prevent_analysis_predicate, save_func)
	paramCounter.count_func_params(self.test_func_addr)
	self.assertEqual(analysis_res[self.test_func_addr], self.test_func_params)

...
def TestSubcallsParamsCounter(self):
    analysis_res = {}
    save_func = lambda addr,num_counts : analysis_res[addr] = num_counts  
	prevent_analysis_predicate = lambda addr: addr in analysis_res

	paramCounter = SubacallsParametersCounter(count_func_params_in_caller_block, prevent_analysis_predicate, save_func)
	paramCounter.count_func_params(self.test_func_addr)
	result = paramCounter.get_params_counts()
	self.assertEqual(analysis_res, self.test_func_subcall_params)

```
### Пример 3

На следующем шаге я рассмотрел код, который непосредственно подсчитывает количество параметров функции на основе инструкций, предшествовавших её вызову: 
```python
def count_func_params_in_caller_block(prevent_analysis_predicate, block):
	last_instr = block.instrs[-1]
		
	if not last_instr.is_subcall():
		return None, None
		
	called_func_addr = last_instr.get_call_dst()
	if not prevent_analysis_predicate(called_func_addr):
		return None, None

	written_regs = machine.X86_64.regs.copy()
		
	for instr in block.instrs[-1::-1]
		if instr.type != machine.X86_64.MOV and instr.type != machine.X86_64.LEA: continue
        written_regs[instr.op1] = 1
	
    num_params = count_params_stdcall(written_regs)
	return called_func_addr, num_params
```

Здесь функция довольно сильно привязана к аппаратной платформе, а также к конкретному соглашению о вызове. Как одно из возможных решений - это выделить отдельный компонент, который будет заниматься соответствующими вычислениями для заданной платформы и соглашения о вызове.   Таким образом,  данный компонент должен уметь определять количество параметров, переданных в функцию. Для этого надо:
- знать, в каком порядке и где (регистры, стек) передаются параметры для заданного соглашения о вызове;
- понять, какие области (регистры, стек) были модифицированы перед вызовом функции
- на основании этих сведений вывести количество параметров, которые передаются функции (а также, возможно, иные сведения, как например типы передаваемых значений). 
Таким образом, можно вывести прототип компонента `CallingConventionAnalyser`, от которого будут наследоваться компоненты, которые выполняют анализ для конкретных платформ и компонентов (часть кода для компактности не приведена  ): 
```python
class CallingConventionAnalyser:
	def __init__(self):
		pass
		
	def analyze_instr(self):
		pass

	def are_params_corretctly_set(self):
		return False

	def count_params(self):
		return None

	
class StdcallX64ParamsAnalyser(CallingConventionAnalyser):	
    ...
    def analyse_instr(self, instr):
        if instr.type in self.modifier_instructions return
        
        if instr.op1.type == machine.X86_64.reg:
            self._mark_as_written_reg(instr.op1)
            return
            
        if instr.op1.type == machine.X86_64.mem and not  self._is_stack_operand(instr.op1):
            return
        
        self._mark_as_written_stack_mem(instr.op1)
        
    def are_params_corretctly_set(self):
        ...

    def count_params(self):
        ...
        

```
В таком случае функция `count_func_params_in_caller_block` примет следующий вид: 
```python
def count_func_params_in_caller_block(prevent_analysis_predicate, block):
	last_instr = block.instrs[-1]
		
	if not last_instr.is_subcall():
		return None, None
		
	called_func_addr = last_instr.get_call_dst()
	if not prevent_analysis_predicate(called_func_addr):
		return None, None
		
	machine = get_machine_for_block(block)
	params_analyser = machine.make_stdcall_analyser()
	
	for instr in block.instrs:
        params_analyser.analyse_instr(instr)
    
	if not params_analyser.are_params_corretctly_set():
        return None, None
        
    num_params = params_analyser.count_params(written_regs)
	return called_func_addr, num_params
```

Считаю, что таким образом мы достигли не только возможности реализовывать анализаторы параметров для разных соглашений о вызовах и платформ, но и большей выразительности и компактности кода. 

### Выводы
1. Подумать над кодом в декларативной парадигме (отвечая на вопрос "что?") крайне продуктивно, это помогает легче увидеть компоненты системы
2. Все же лучше потратить некоторое время на предварительное обдумывание структуры компонента, которые ты хочешь написать,  а также на его хотя бы словесное описание. Это будет значительно быстрее, чем написать что-то, а потом поэтапно рефакторить (особенно если задача не ясна до конца)
3. Имея представление о вычислительных моделях  и возвращаясь к коду, который был написан некоторое время назад (больше полутора лет), становится легче сходу видеть какие-то варианты его улучшения. 
4. Не уверен, что итоговое изменение кода - это наилучший вариант реализации этих компонентов, однако можно с уверенностью сказать, что возможность повторного их использования  и общая ясность повысилась.
