
### Пример № 1
Было: 
```python
class Storage:
	def __init__(self, size):
		assert size <= 0,  "Error,size of the storage must be greater than zero"
		self.size = size
		...

	def insert(self, item):
		...
```
Стало: 
```python
class Storage:
	STORAGE_CREATE_OK = 0
	STORAGE_CREATE_ERR = 1
	
	def __init__(self, size):
		self.size = size
		...
	...

def MakeStorage(size):
	if size <=0:
		return None
	
	return Storage(size)
	
```

В данном случае мы вынесли проверку размера хранилища из конструктора во внешнюю функцию, `MakeStorage`, которая сейчас отвечает за проверку входных данных и не создает хранилища, если они не удовлетворяют заданным условиям к его размеру. Конечно это не гарантирует вызов конструктора напрямую на уровне языка, но даже в таком виде позволяет избавиться от недопустимых состояний в самом классе при его создании. 

### Пример № 2
Было: 
```C

NTSTATUS DeviceInsertPendingIoRequest(PDEVICE_CONTEXT devCtx, WDFREQUEST Request, WDFREQUEST* requestPtr, ULONG* ioctlCodePtr)
{
    
    ULONG ioctl = WdfRequestGetIoCtlCode(Request);
    WdfInterruptAcquireLock(devCtx->interrupt);

    if (*requestPtr != NULL)
	{
		WdfInterruptReleaseLock(devCtx->interrupt);
		return STATUS_UNSUCCESSFUL;
	}
	
    *requestPtr = Request;
    if (ioctlCodePtr != NULL)
	{
		
       ASSERT(*ioctlCodePtr == 0);
	   *ioctlCodePtr = ioctl;
    }
    
	WdfInterruptReleaseLock(devCtx->interrupt);
    return STATUS_SUCCESS;
}


BOOLEAN DeviceCompletePendingIoRequestWithInformation(PDEVICE_CONTEXT devCtx, WDFREQUEST *request, ULONG* ioctlCodePtr,
    NTSTATUS completionStatus, ULONG_PTR numBytesTransmitted)
{

    WdfInterruptAcquireLock(devCtx->interrupt);
    BOOLEAN result;
    if (*Request == NULL)
    { 
		WdfInterruptReleaseLock(devCtx->interrupt);
		return FALSE;
    }
    
	WDFREQUEST requestToComplete = *request;
    *request = NULL;

    if (ioctlCodePtr != NULL)
	{
		ASSERT(*ioctlCodePtr != 0);
        *ioctlCodePtr = 0;
	}
    WdfInterruptReleaseLock(devCtx->interrupt);
    WdfRequestCompleteWithInformation(requestToComplete, completionStatus, numBytesTransmitted);        
    
    return TRUE;
}
```
Стало:
```C

VOID PendingRequestDataSet(PENDING_REQUEST_DATA* pendingRequestDataPtr, WDFREQUEST Request, ioctlCode)
{
	   pendingRequestDataPtr->request = Request;
       pendingRequestDataPtr->ioctl = ioctlCode;
}

void PendingRequestDataClear(PENDING_REQUEST_DATA* requestData)
{
    requestData->request = NULL;
    requestData->ioctl = 0;
}

NTSTATUS DeviceInsertPendingIoRequest(PDEVICE_CONTEXT devCtx, WDFREQUEST Request, PENDING_REQUEST_DATA* pendingRequestDataPtr)
{
    
    ULONG ioctlCode = WdfRequestGetIoCtlCode(Request);
    NTSTATUS status;

    WdfInterruptAcquireLock(devCtx->interrupt);
    if (pendingRequestDataPtr->request != NULL)
    {
       status = STATUS_UNSUCCESSFUL;
    }
    else
    {
		PendingRequestDataSet(pendingRequestDataPtr, Request, ioctlCode);
		status = STATUS_SUCCESS;
    }
    WdfInterruptReleaseLock(devCtx->interrupt);

    return status;
}

BOOLEAN DeviceCompletePendingIoRequestWithInformation(PDEVICE_CONTEXT devCtx, PENDING_REQUEST_DATA * requestData,
    NTSTATUS completionStatus, ULONG_PTR numBytesTransmitted)
{

    WdfInterruptAcquireLock(devCtx->interrupt);

    if (requestData->request == NULL)
    { 
        WdfInterruptReleaseLock(devCtx->interrupt);
		return FALSE
    }
	
    WDFREQUEST requestToComplete = requestData->request;
    PendingRequestDataClear(requestData);

    WdfInterruptReleaseLock(devCtx->interrupt);

    WdfRequestCompleteWithInformation(requestToComplete, completionStatus, numBytesTransmitted);

    return TRUE;
}

```
В данном случае нам надо было поддерживать в согласованном состоянии два поля: `request` (обрабатываемый в текущий момент запрос ввода/вывода к драйверу) и `ioctlCode` (код запроса операции ввода/вывода). Когда запрос обработан, мы удаляем его из контекста - присваиваем значение NULL соответствующему полю структуры (который передаем в функцию по ссылке, как указатель), одновременно обнуляя и `ioctlCode`.  При этом, ситуацию, когда у нас значения рассогласованы, мы оборачиваем в ASSERT.  Чтобы избавиться от ASSERT, я объединил эти поля в единую структуру  - `PENDING_REQUEST_DATA`, запись в которую осуществляется исключительно  с использованием отдельно реализованных функций  -  `PendingRequestDataSet`, `PendingRequestDataClear` . Таким образом, мы можем гарантировать, что оба данных поля будут всегда согласованы, и  необходимость использования ASSERT отпадает.

### Пример № 3
Было: 
```python

def start_connection(addr:str):
	if not is_addr_valid(addr):
		raise ValueError("Invalid address provided %s" % addr)
	...

```
Стало: 
```python
def start_connection(remoteHost:RemoteHost):
	status = send_handshake(remoteHost)
	...
```
В данном случае мы избавляемся от проверки корректности переданного адреса удаленного хоста, с которым хотим связаться, через выделение дополнительной сущности RemoteHost, который осуществляет все необходимые проверки. Это пересекается с техникой отказа от использования примитивных типов и переходом к более сложным. 

### Пример № 4
Было: 
```C
	class UniformlyDistributedInteger : public RandomInteger
	{
	private:
		std::mt19937 _generator;
	public:
		UniformlyDistributedInteger()
		{
			std::random_device dev;			
			_generator = std::mt19937(dev());
		}
		virtual ~UniformlyDistributedInteger()
		{
		}
		virtual int Get(int min, int max)
		{
			if (min > max)
				throw_runtime_error("ASSERT:invalid range for random generator");
			
			std::uniform_int_distribution<> distr(min, max);
			int result = distr(_generator);
			return result;
		}
	};
```

Стало: 
```C
	class UniformlyDistributedInteger : public RandomInteger
	{
	private:
		std::mt19937 _generator;
	public:
		UniformlyDistributedInteger(int min, int max): _min (min), _max(max)
		{
			std::random_device dev;			
			_generator = std::mt19937(dev());
		}
		virtual ~UniformlyDistributedInteger()
		{
		}
		virtual int Get(Interval<int> &interval)
		{		
			std::uniform_int_distribution<> distr(interval.min, interval.max);
			return distr(_generator);
		}
	};
	...
```

В данном случае мы заменили параметры min и max отельной сущностью `Interval`, которая представляет собой интервал, ограниченный максимальным и минимальным значениями. Проверка корректности интервала задается в самой реализации данного класса, что позволяет нам  избавиться от ASSERT.

### Пример № 5
Было: 
```python

def enum_nested_calls(func_addr:int):
	try:
		instrs = get_instructions(addr)
	except e:
		print("Invalid address provided")
		return
	
	for instr in instrs:
		... 

```
Стало: 
```python
def enum_nested_calls(func:Function):
	for instr in func.instructions:
		... 
```
В данном случае по аналогии с примером 3 мы избавляемся от проверки корректности переданного адреса функции,  выделяя дополнительный тип `Function`, который будет  реализовывать проверки корректности при своем создании.

## Выводы
Главная мысль, которую удалось усвоить из этого занятия -  избавление от ASSERT-ов и логов  это не метод проектирования как таковой. В конце концов, например, некоторые правки, приведенные выше в примерах 3,4,5 используют уже изученные техники, в частности, избавление от примитивных типов. Но свой код имеет смысл анализировать на ASSERT и дополнительные логи, так как это выполняет роль хорошего сигнала о том, что над тем местом, где они используются следует еще раз подумать, чтобы придти к лучшему с точки зрения дизайна решению.   

