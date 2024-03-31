### Пример 1

В приведенной строке одновременно вычисляется размер ожидаемых данных и валидируется размер буфера: 
```C
if (buf_size <= ((sizeof(DEVICE_TO_HOST_START_OF_MSG) + sizeof(DEVICE_TO_HOST_END_OF_MSG)) / sizeof(uint32_t) + answer_size_in_dwords) * sizeof(uint32_t))
{
	  ...
}
```
После преобразования код выглядят так: 
```C
uint32_t total_answer_size_in_dwords = (sizeof(DEVICE_TO_HOST_START_OF_MSG) + sizeof(DEVICE_TO_HOST_END_OF_MSG)) / sizeof(uint32_t) + answer_size_in_dwords;

if (buf_size <= total_answer_size_in_dwords * sizeof(uint32_t))
{
	  ...
}
```

### Пример 2
В приведенной строке одновременно вычисляются критерии допустимости записи данных на устройство и проверяются указанные критерии: 
```C
if (!(state->regs.cmsr & READ_TRANSFER_ENABLE) || (state->regs.cmsr & DEVICE_TO_HOST_FIFO_EMPTY))
{
	...
}
```
Код после преобразования: 
```C
bool is_read_enabled = (state->regs.cmsr & READ_TRANSFER_ENABLE);
bool is_fifo_empty = (state->regs.cmsr & DEVICE_TO_HOST_FIFO_EMPTY);

if (!is_read_enabled || is_fifo_empty)
{
	...
}
```

### Пример 3
В одной и той же строке происходит загрузка файла с данными и итерация по загруженным данным:
```python
for traceItem in json.load(stateFile)['log']:
	...
```

Код после преобразования: 
```python
traceData = json.load(stateFile)
traceLog = traceData['log']
for traceItem in traceLog:
	...
```

### Пример 4
В одной и той же строке копируется словарь и происходит итерация по его элементам: 
```python
for symbol in functionsHandlers.copy():
       handlerCallsCounters[symbol] = 0

```
Код после преобразования:
```python
functionsHandlerSnapshot = functionsHandlers.copy()
for symbol in functionsHandlerSnapshot:
       handlerCallsCounters[symbol] = 0
```

### Пример 5
По аналогии с п. 4 в одной и той же строке происходит копирование структуры данных и её использование: 
```python
if inputSymbols == {}:
	setDestinationToState(self.machine.mn.regs.regs_init.copy(), startAddress)
```

Код после преобразования:
```python
if inputSymbols == {}:
	initial_symbols = self.machine.mn.regs.regs_init.copy()
	setDestinationToState(initial_symbols, startAddress)
```
