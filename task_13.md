
### Пример 1. RAII для Handle
Рассмотрим класс, который инкапсулирует в себе логику работы с устройством в ОС Windows:
```C++

class Device
{
private:
	HANDLE hDevice;
	HANDLE hDeviceBlocking;
	HANDLE hDeviceMutex;
private:
	void AcquireDeviceMutex(int waitTimeout);
	void ReleaceDeviceMutex();

public:
	Device(LPWSTR deviceName);
	~Device();
	...
};


Device::Device(LPWSTR devicePath)
{
    this->hDevice = CreateFile(devicePath,
        GENERIC_READ | GENERIC_WRITE,
        FILE_SHARE_READ | FILE_SHARE_WRITE,
        NULL,
        OPEN_EXISTING,
        FILE_FLAG_OVERLAPPED,
        NULL);
	
    if (this->hDevice == INVALID_HANDLE_VALUE)
        throw std::runtime_error(GetLastErrorAsString("Error creating non blocking handle file for device"));

    this->hDeviceBlocking = CreateFile(devicePath,
        GENERIC_READ | GENERIC_WRITE,
        FILE_SHARE_READ | FILE_SHARE_WRITE,
        NULL,
        OPEN_EXISTING,
        NULL,
        NULL);
    if (this->hDeviceBlocking == INVALID_HANDLE_VALUE)
        throw std::runtime_error(GetLastErrorAsString("Error creating blocking  for device"));
		
		...
		
    HANDLE mutexHandle = CreateMutexA(NULL, FALSE, mutexName);
    if (handle == INVALID_HANDLE_VALUE)
            throw std::runtime_error(GetLastErrorAsString("Error creating mutexes for  device:"));
    
	this->hDeviceMutex = mutexHandle;

};

Device::~Device()
{
	if (this->hDevice != INVALID_HANDLE_VALUE)
		CloseHandle(this->hDevice);

    if (this->hDeviceBlocking != INVALID_HANDLE_VALUE)
        CloseHandle(this->hDeviceBlocking);
	
	if (this->hDeviceMutex != INVALID_HANDLE_VALUE)
        CloseHandle(this->hDeviceMutex);
};

```

Мы видим, что данный класс использует для работы несколько дескрипторов: дескрипторы устройства, открытые в блокирующем и неблокирующем режиме (`hDevice`, `hDeviceBlocking`), а также дескриптор мьютекса (`hDeviceMutex`) для синхронизации доступа к устройству. Каждый из этих дескрипторов создается в конструкторе и освобождается в деструкторе, таким образом соблюдая принцип RAII. Однако, указанный код можно  еще  сократить, обернув в RAII-обертки сами указанные дескрипторы: 
```C++
class Handle
{
	private: 
		HANDLE _handle;
	public:
		Handle()
		{
			this->_handle = INVALID_HANDLE_VALUE;
		}
		Handle(HANDLE handle):_handle(handle)
		{
		};
		~Handle()
		{
			if (this->_handle != INVALID_HANDLE_VALUE)
				CloseHandle(this->_handle);
		}
		HANDLE Get()
		{
			return this->_handle;
		};
		HANDLE Set(HANDLE handle)
		{
			this->_handle = handle;
		};
}
```
Таким образом, код класса `Device` примет следующий вид: 
```C++
class Device
{
private:
	Handle hDevice;
	Handle hDeviceBlocking;
	Handle hDeviceMutex;
	...
}

Device::Device(LPWSTR devicePath)
{
     auto hDev = CreateFile(devicePath,
        GENERIC_READ | GENERIC_WRITE,
        FILE_SHARE_READ | FILE_SHARE_WRITE,
        NULL,
        OPEN_EXISTING,
        FILE_FLAG_OVERLAPPED,
        NULL);
	
	this->hDevice.Set(hDev);
	auto hDevBlock = CreateFile(devicePath,
        GENERIC_READ | GENERIC_WRITE,
        FILE_SHARE_READ | FILE_SHARE_WRITE,
        NULL,
        OPEN_EXISTING,
        NULL,
        NULL);
	this->hDeviceBlocking.Set(hDevBlock);		
		...
		
    HANDLE mutexHandle = CreateMutexA(NULL, FALSE, mutexName);
    if (handle == INVALID_HANDLE_VALUE)
            throw std::runtime_error(GetLastErrorAsString("Error creating mutexes for  device:"));
    
	this->hDeviceMutex.Set(mutexHandle);

};

Device::~Device()
{
};
```

Заменив "сырой" дескриптор на обертку, мы получили некоторый дискомфорт, связанный с тем, что для передачи значений в API-функцию теперь надо вызывать метод `hDevice.Get()`. Однако, при этом, мы получаем некоторую защиту от утечки ресурсов, если забудем в деструкторе закрыть соответствующий дескриптор.   Подобные обертки могут быть распространены на другие типы ресурсов. Также присутствуют уже готовые реализации, например https://github.com/okdshin/unique_resource. 

### Пример 2. Синхронизация в драйвере. Обертка через макрос
Типичная ситуация при разработке драйвера состоит в том, что имеется некоторая функция, которая должна быть синхронизирована, например с обработчиком прерываний. Для этих целей создается  spinlock, который захватывается каждый раз при входе в функцию и освобождается при выходе из неё. В итоге код функции на языке "С" может выглядеть  примерно так: 
```C

NTSTATUS ReadMessageFromDevice(PDEVICE_CONTEXT deviceContext, OutMessage * outMessageBuf, size_t outMessageBufCapacity, ULONG_PTR* numBytesReceived)
{
	...
	WdfSpinLockAcquire(deviceContext->hardwareAccessSpinLock);

	USHORT flags = READ_PORT_USHORT((PUSHORT)&deviceContext->registers->io_flags);
	if (IsRecvFifoEmpty(flags))
	{
		WdfSpinLockRelease(deviceContext->hardwareAccessSpinLock);
		status = STATUS_UNSUCCESSFUL;
		return status;
	};
	
	...
	status = WaitForTransmitFifoEmpty(deviceContext);
	if (!NT_SUCCESS(status))
	{
		WdfSpinLockRelease(deviceContext->hardwareAccessSpinLock);
		return STATUS_UNSUCCESSFUL;
	}
	...
	
	status = ReadBuf(deviceContext, outMessageBuf, outMessageBufCapacity, numBytesReceived);
	if (!NT_SUCCESS(status))
	{
		WdfSpinLockRelease(deviceContext->hardwareAccessSpinLock);
		return status;
	}
	WRITE_PORT_UCHAR((PUCHAR)&deviceContext->registers->io_status, 0);
	WdfSpinLockRelease(deviceContext->hardwareAccessSpinLock);	
	return STATUS_SUCCESS;
};
```
По итогу в каждой ветке, где функция может завершиться неудачно мы обязаны дублировать процедуру освобождения spinlock:  `WdfSpinLockRelease`. И необходимость в этом будет характерна для любой функции, при выполнении которой нужно будет захватывать блокировку. 
При этом захват и освобождение spinlock можно отделить от логики работы самой функции, обернув  вызов в соответствующий макрос: 
```C
#define CallLocked(funcName, status, lockName, deviceContext, ...) \
	WdfSpinLockAcquire(deviceContext->lockName);\
	status = funcName (deviceContext, ##__VA_ARGS__);\
	WdfSpinLockRelease(deviceContext->lockName);\
```

После чего функции, требующие синхронизации,  становится компактнее. Ниже приведена исходная функция после изменения: 

```C
NTSTATUS ReadMessageFromDevice(PDEVICE_CONTEXT deviceContext, OutMessage * outMessageBuf, size_t outMessageBufCapacity, ULONG_PTR* numBytesReceived)
{
	...

	USHORT flags = READ_PORT_USHORT((PUSHORT)&deviceContext->registers->flags);
	if (IsRecvFifoEmpty(flags))
		return STATUS_UNSUCCESSFUL;
	
	...
	status = WaitForTransmitFifoEmpty(deviceContext);
	if (!NT_SUCCESS(status))
		return STATUS_UNSUCCESSFUL;
	
	...
	
	status = ReadData(deviceContext, outMessageBuf, outMessageBufCapacity, numBytesReceived);
	if (!NT_SUCCESS(status))
		return status;

	WRITE_PORT_UCHAR((PUCHAR)&deviceContext->registers->io_reg, 0);
	return STATUS_SUCCESS;
};
```

 Вызываться она будет следующим образом:  
```C
...
CallLocked(ReadMessageFromDevice, status, hardwareAccessSpinLock, deviceContext,
		   outMessageBuf, outMessageBufCapacity, numBytesReceived);
...
```
Вызов выглядит не очень компактно и красиво, однако данный приём позволяет избежать потенциальных ошибок, связанных с тем, что мы забываем освободить spinlock в какой-либо ветке исходной функции. На мой взгляд это разумная цена, которую можно заплатить для того, чтобы избавиться от многих потенциальных часов отладки в будущем.  

### Пример 3. Разделение логики эмулируемой функции и соглашения  о вызове
В ходе эмуляции бинарного программного кода  часто попадаются уже известные api-функции, выполнять код которых нецелесообразно, так как их функционал и так известен. В этом случае можно эмулировать их вызовы, подставляя в качестве выходных параметров те или иные значения. Типовым примером будет вот такой обработчик функции malloc (для краткости приведен только один):
```python
def malloc_handler(jitter):
    log("malloc_handler:", jitter.cpu.RAX, jitter.cpu.RBX, jitter.cpu.RCX)

    size = jitter.cpu.RCX
    
    #логика функции malloc
    addr = winobjs.heap.alloc(jitter, size)
    jitter.vm.set_mem(addr, b'\x00'*size)
    #логика функции malloc завершается
    
    jitter.cpu.RAX = addr
    jitter.cpu.RIP = int.from_bytes( jitter.vm.get_mem(jitter.cpu.RSP, 8),  byteorder = 'little')
    jitter.pc = jitter.cpu.RIP
    jitter.cpu.RSP += 8
    
    log("malloc_handler: addr = ", hex(addr))
    return True

```
Данный обработчик полностью имитирует выполнение функции malloc, выделяя память в контексте эмулятора. Следующим примером будет обработчик функции strlen:
```python
def strlen_handler(jitter):
    log("strlen_handler:", jitter.cpu.RAX, jitter.cpu.RBX, jitter.cpu.RCX)

    str_addr = jitter.cpu.RCX
	#логика функции strlen
    str_len = get_len_str(str_addr)
    jitter.cpu.RAX = str_len
    #логика функции strlen завершается
    
    jitter.cpu.RIP = int.from_bytes( jitter.vm.get_mem(jitter.cpu.RSP, 8),  byteorder = 'little')
    jitter.pc = jitter.cpu.RIP
    jitter.cpu.RSP += 8
    
    log("strlen_handler: addr = ", hex(addr))
    return True
```
 В данных обработчиках можно выделить две составляющие: 
 - код, реализующий непосредственно логику эмулируемой функции
 - код, реализующий логику соглашения о вызове (чтение входных параметров, установка результирующих значений, модификация стека и.т.д.)
 В данном случае они не разделены явно, что приводит к написанию лишнего кода и не позволяет использовать те же обработчики для других соглашений о вызове.
Явно абстрагировав код, который реализует соглашение о вызове, мы получим следующее:
```python
...
def stcall_handler(handler_spec, jitter):
    
    params_count = handler_spec['params_count']
    func_name = handler_spec['func_name']
    handler_func = handler_spec['handler_func']
    
    # в этом коде мы отдельно выделили вызов функции, реализующей
    # логику эмулируемой функции 
    parameters = stdcall_get_params(params_count)
    log(func_name, parameters)
    func_result = handler_func(jitter, parameters)
    
    # ниже мы реализуем логику соглашени о вызове
    jitter.cpu.RAX = func_result
    jitter.cpu.RIP = int.from_bytes( jitter.vm.get_mem(jitter.cpu.RSP, 8),  byteorder = 'little')
    jitter.pc = jitter.cpu.RIP
    jitter.cpu.RSP += 8
    
    return True


def malloc_handler(jitter, params):
	size = params[0]
	new_mem = winobjs.heap.alloc(jitter, size)
    jitter.vm.set_mem(addr, b'\x00'*size)
    return new_mem
    
def strlen_handler(jitter, params):
	str_addr = params[0]
	str_len = get_len_str(str_addr)
    return str_len

...
malloc_handler_spec['func_name'] = 'malloc'
malloc_handler_spec['params_count'] = 1
malloc_handler_spec['handler_func'] = malloc_handler

malloc_handler_spec['func_name'] = 'strlen'
malloc_handler_spec['params_count'] = 1
malloc_handler_spec['handler_func'] = strlen_handler
```

Таким образом в обработчик эмулируемой функции мы передаем её спецификацию, которая содержит в себе сведения о количестве её параметров, о названии и об обработчике, который реализует логику эмуляции. Таким образом, разделив логику соглашения о вызове и логику самой функции мы получаем возможность использовать одни и те же обработчики в разных контекстах, с разными соглашениями. К тому же код становится более читабельным. 

### Вывод
Из приведенных примеров видно, что описанный приём в некоторых случаях помогает увеличивать читабельность кода, но это не основное преимущество. Основное преимущество состоит в том, что выделение подобных управляющих конструкций помогает избежать некоторых ошибок, что сильно экономит время, которое может быть потенциально затрачено на их поиск в будущем.
