### Итерация 1
### Код
```C++
/* 
	Класс SeedStorage реализует некоторое постоянное хранилище исходных тестовых кейсов, 
	которые используются для фаззинга. Хранилище предоставляет возможность: 
	- внести новый тестовый кейс.
	- получить из хранилища тестовый кейс.
	Каждому тестовому кейсу при внесении в хранилище  присваивается идентификатор,
	по которому его в дальнейшем можно получить из хранилища обратно. 
	Логика формирования идентификатора может отличаться в разных хранилищах, поэтому 
	код, использующий его не должен делать каких-либо предположений и использовать
	данный идентификатор только для получения тесткейса из хранилища в дальнейшем. 
*/

class SeedStorage
{
private:
	std::string _storageDir;
	int _lastId;
	
private:
	static int CheckSeedDir(std::string storageDir)
	{		
		int lastId = 0;
		
		for (auto & entry : fs::directory_iterator(storageDir))
		{
		const char* pattern = "id_";
		std::string fname = entry.path().string();
		unsigned int possibleIdPos = fname.rfind(pattern);
		if (possibleIdPos == std::string::npos)
			continue;
		std::string seedNum = fname.substr(possibleIdPos+sizeof(pattern));
		try
		{
			int curId = std::stoi(seedNum);
			if (curId>lastId)
			lastId = curId;
		}
		catch (const std::exception)
		{			
			continue;
		}
		}
		return lastId;
	};	
	static std::string GetSeedFileNameById(int id)
	{
		std::stringstream strStream;
		strStream << "id_"<<std::setfill('0') << std::setw(6) << id;
		
		return strStream.str();
	};

public:
	static const unsigned int maxSize = 1024 * 1024;
	
public:
	SeedStorage(std::string storageDir):_storageDir(storageDir)
	{
		_lastId = CheckSeedDir(_storageDir);		
	};
	virtual ~SeedStorage()
	{
	};
	virtual int GetLastId()
	{
		return _lastId;
	};
	
	TestCase GetSeed(int seedId)
	{
		if ( seedId < 0 || seedId >_lastId)
			throw_out_of_range_error("Seed id doesn't match existing range");
		
		std::string fName = GetSeedFileNameById(seedId);
		std::string path("");
		path += ".\\" + _storageDir + "\\" + GetSeedFileNameById(seedId);	

		try
		{
			TestCase testCase =  ReadData(path);
			return testCase;
		}
		catch (const winafl_runtime_error &e)
		{
			throw e;
		}
	};

	int NewSeed(TestCase testCase)
	{
		_lastId++;
		if (testCase.size()>SeedStorage::maxSize)
			throw_runtime_error("seed size exceeds limit");
		
		std::string path("");
		path += ".\\"+_storageDir+"\\"+GetSeedFileNameById(_lastId);	
		
		WriteTestCase(path, testCase);
		return _lastId;
	};
};
```

### Выводы и рефлексия
При написании класс `SeedStorage` мыслился просто как вспомогательный класс для работы с каталогом, в котором хранятся исходные тестовые кейсы для дальнейшего фаззинга. При написании комментария переосмыслил роль этого класса в общей системе, понял, что это по сути некое "постоянное" хранилище с описанным в комментарии набором свойств. Несмотря на то, что в текущей реализации данного хранилища вполне достаточно, в голову пришли мысли о том, как сделать  его более соответствующим своей спецификации. Так, например, пришел к тому,  идентификатор тесткейса должен быть некоторой непрозрачной структурой (по аналогии с дескриптором  HANDLE в WinApi), он должен быть выделен в отдельный тип. 
Также в текущей реализации перед запуском фаззера, я размещал подготовленные и соответствующим образом названные тестовые кейсы в каталог "storageDir". В результате, при использовании фаззера я полагался на реализацию данного класса. В итоге написал отдельный код, который считывает исходные тесткейсы из отдельно заданного каталога.

### Итерация 2
### Код
```C++
/*
	Данный класс позволяет доставить тесткейс до 
	тестируемого приложения, для его дальнейшего 
	запуска. Реализует в себе все процедуры, 
	необходимые для обеспечения корректности 
	указазанного поведения, в зависимости от 
	среды, модели работы самого тестируемого 
	приложения и иных сопутствующих факторов. 	
*/

class Publisher
{
private:
	std::string _outFileName;
	double _totalExecTime;
public:

	Publisher(std::string outFileName): _outFileName(outFileName)
	{
		_totalExecTime = 0;
	};
	~Publisher()
	{
	};
	
	void WriteTestcase(TestCase & data)
	{
		std::time_t start = std::time(nullptr);
		
		std::ofstream outFile;
		outFile.open(_outFileName, std::ofstream::out|std::ofstream::trunc|std::ofstream::binary);
		if (outFile.fail())				
			throw_runtime_error("could not open out file"+_outFileName);

		outFile.write(data.data(), data.size());
	
		if (outFile.fail())
			throw_runtime_error("could not write testcase to file");			
		
		outFile.close();
		_totalExecTime += std::difftime(std::time(nullptr), start);
	};
	
	double GetTotalExecTime()
	{
		return _totalExecTime;
	}
};
```

### Выводы и рефлексия
Как и в предыдущем пункте, данный был просто вспомогательным классом, который реализовывал в себе запись тестового кейса в заранее заданный файл, который подавался на вход приложению. На самом деле это по сути отдельный полноценный компонент, который, в зависимости от модели функционирования тестируемого приложения может реализовывать в себе разную логику работы с тесткейсами. В данном случае мы просто записываем его в заранее заданный файл, однако есть и иные вариации, когда мы записываем тесткейс например в сокет, пройдя процедуру установления соединения по тому или иному протоколу, либо записывая тестовые данные напрямую в выделенный участок памяти процесса, в рамках которого функционирует тестируемая программа. 
В обоих случаях данный интерфейс является вполне подходящим. Размышляя над этим, увидел также лишний метод `GetTotalExecTime`, который к функциональности класса не относится, который я впоследствии убрал. 

### Итерация 3
### Код
```python
'''
 Данный класс предназначен для символьного исполнения участка кода, 
 на основе заданного  начального состояния (значений символьных переменных). 
 Останавливает выполнение, когда не удается вычислить адрес следующего участка кода. 
'''
class BlockRunner:
    def __init__(self, machine = None, symbolicExecutionEngine = None, execStrategy = None):
        self.loc_db = LocationDB()
        self.machine = Machine("x86_64") if machine is None else machine
        self.bs = bin_stream_ida()

        self.lifter = self.machine.lifter(loc_db=self.loc_db)
        self.mdis = self.machine.dis_engine(self.bs, loc_db=self.loc_db)
        self.sb = SymbolicExecutionEngine(self.lifter) if symbolicExecutionEngine is None else symbolicExecutionEngine(self.lifter)
        
        self.execStrategy = execStrategy if execStrategy is not None else DefaultExecutionStrategy()
            
    def runStateUntilDstUnknown(self, state, start = None):
        if start is None:
            self.execStrategy.onBlockExecuted(state)
            start = getDestinationFromState(state)
            
        if not isinstance(start, int) and not  isinstance(start, ExprInt) and not isinstance(start, ExprLoc):
            return None

        self.sb.set_state(state)
        asmcfg = self.mdis.dis_multiblock(start)
        ircfg = self.lifter.new_ircfg_from_asmcfg(asmcfg)
        run_by_execution_strategy(self.sb, self.mdis, self.lifter, ircfg, start, self.execStrategy)
        return self.getCurrentStateRef()
    
    def getModifiedSymbols(self, initState):
        return self.sb.modified(init_state = initState)

    def makeBlankState(self):
        blankState = self.machine.mn.regs.regs_init.copy()
        blankState[Q('IRDst')] = Q('INIT')
        self.ircfg = None
        return blankState

    def setStateAndReturnItsCopy(self, initialState):
        self.sb.set_state(initialState)
        return self.sb.symbols.copy()

    def setState(self, initialState):
        self.sb.set_state(initialState)
    
    def getCurrentStateCopy(self):
        return self.sb.symbols.copy()

    def getCurrentStateRef(self):
        return self.sb.symbols
```

### Выводы и рефлексия
Выше приведен вспомогательный класс, который реализует символьное исполнение заданного участка кода на основе начального состояния, которое представляет собой некий заранее заданный набор  значений символьных переменных. Эту функциональность реализует   метод `runStateUntilDstUnknown`. Несмотря на то, что этот класс относительно  простой по своей функциональности, в ходе написания комментария увидел, что у него довольно много лишних методов для работы с его текущим внутренним состоянием - `getCurrentStateRef`, `getCurrentStateCopy`, `setStateAndReturnItsCopy`, `makeBlankState`. По сути данные методы не имеют прямого отношения непосредственно к данному классу и  их необходимо выделить логически в другой блок, который реализует работу с состоянием.  
