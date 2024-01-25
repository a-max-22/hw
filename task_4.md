### Итерация № 1
#### Мутации
Работая с фаззером из предыдущего задания, рассмотрим то, как генерируются тесткейсы для проверки. Поскольку данный фаззер писался как калька с afl, то в нем были реализованы характерные для afl мутации: 
1) bitflip - инвертирование одного бита на какой-то заданной позиции в тесткейсе. 
2) byteflip/wordflip/dwordflip/... - инвертирование какого-либо байта/слова/двойного слова на заданной позиции в тесткейсе. 
3) арифметические операции - сложение какого-либо байта/слова/двойного слова с заранее заданной константой. 
4) замена байта на "интересное" значение, вычисленное на основе того, как тестируемая программа реагирует на изменение значения "интересного" байта. Если при его изменении меняется покрытие, то значение считается "интересным" и добавляется в словарь.   
5) сплайсинг - замена одного участка подряд идущих байт в исходном файле на другой участок подряд идущих байт другого файла, как правило выбранных случайным образом. 

#### Алгоритм применения мутаций 
Мутации могут применяться последовательно, одна за другой, либо выбираться в случайном порядке. Как правило вначале идет некоторая "регулярная" часть,  в ходе которой : 
1) последовательно инвертируется каждый бит заданного тесткейса; 
2) последовательно инвертируется каждый байт заданного тесткейса 
3) и.т.д.
Затем то же самое  повторяется с арифметическими операциями  к каждому байту/слову/двойному слову  последовательно прибавляется значение из заранее заданного списка констант. Иными словами, каждый вид мутации последовательно применяется к каждому биту/байту/слову/двойному слову и.т.д. до тех пор, пока возможность их применения не будет исчерпана. 

После "регулярной" части может выполняться некоторая "хаотическая" часть, когда разные мутации из обозначенного списка применяются к случайным участкам тесткейса, по отдельности, либо совместно, последовательно применяя несколько мутаций к одному и тому же тесткейсу. 

#### Изначальный код
Ниже представлена изначальная версия кода, реализующего последовательные мутации по инвертированию  байта/слова/двойного слова в тесткейсе:
```C++
typedef std::vector<char> TestCase;
...

class Mutator
{
	public:
		Mutator()
		{
		};
		virtual ~Mutator()
		{
		};
		virtual TestCase &Mutate() = 0;	
};

class SequentialMutator:public Mutator
{
	public:
		SequentialMutator()
		{
		};
		virtual ~SequentialMutator()
		{};
		virtual TestCase &Mutate() = 0; //mutation params
		virtual bool IsMutationFinished() = 0;		
};

class SequentialByteGrainedMutator:public SequentialMutator
{
	protected:
		TestCase _data;	
		int _range;
		int _mutationPos;

		// count of successive bytes changed by sungle mutation;
		int _mutationSize;

	protected:		
		// modification area begin
		// modification ared end
		virtual char* ModifyData(char* modBegin, char* modEnd)
		{
			return modBegin;
		}
		virtual void MakeMutation(int mutationPos)
		{				
			try
			{								
				// get original value ptr
				ModifyData(&_data.at(mutationPos), &_data.at(mutationPos+_mutationSize-1));
			}
			catch (std::out_of_range)
			{				
				std::cout<<"don't mutate" << std::endl;			
			}
		}
		virtual char* RestoreData(char* modBegin, char* modEnd)
		{
			return modBegin;
		}
		
		virtual void RestorePreviouslyMutatedData(int restorePos)
		{			
			try
			{								
				// get original value ptr
				RestoreData(&_data.at(restorePos), &_data.at(restorePos+_mutationSize-1));
			}
			catch (std::out_of_range)
			{
				std::cout<<"don't restore" << std::endl;
			}
		}

		virtual void inline AdvancePosition()
		{
			_mutationPos++;
		}
	
		virtual void inline SkipMutation()
		{
			_mutationPos++;
		}
		
	public:
		SequentialByteGrainedMutator(TestCase data)
		{			
			_data = data;
			_range = data.size();			
			_mutationSize = 1;
			_mutationPos = -1;
		};
		virtual ~SequentialByteGrainedMutator()
		{
			
		};

		virtual TestCase &Mutate() 
		{	
			if (IsMutationFinished())
				return _data;
			
			RestorePreviouslyMutatedData(_mutationPos);			
			AdvancePosition();
			MakeMutation(_mutationPos);	
			return _data;
		};
		virtual bool IsMutationFinished()
		{
			return _mutationPos + _mutationSize - 1 >= _range;
		};		
};

class ByteFlip:public SequentialByteGrainedMutator
{
	protected:
		unsigned int _origValue;
	protected:
		// perform dword flip
		char* ModifyData(char* modBegin, char* modEnd) override
		{
			_origValue = *modBegin;			
			*modBegin ^= 0xFF;
			return modBegin;
		}		
		char* RestoreData(char* modBegin, char* modEnd) override
		{
			*modBegin = _origValue;
			return modBegin;
		}		
	public:
		ByteFlip(TestCase data): SequentialByteGrainedMutator(data)
		{
			_origValue = 0xFF;
			_mutationSize = 1;
			_mutationPos = -1;
		}
		~ByteFlip()
		{	
		};
};

class WordFlip:public SequentialByteGrainedMutator
{
	protected:
		unsigned short _origValue;
	protected:
		// perform dword flip
		char* ModifyData(char* modBegin, char* modEnd) override
		{
			_origValue = *((unsigned int *)modBegin);			
			*((unsigned short *)modBegin) ^= 0xFFFFFFFF;
			return modBegin;
		}		
		char* RestoreData(char* modBegin, char* modEnd) override
		{
			*((unsigned short *)modBegin) = _origValue;
			return modBegin;
		}		
	public:
		WordFlip(TestCase data): SequentialByteGrainedMutator(data)
		{
			_origValue = 0xFF;
			_mutationSize = 2;
			_mutationPos = -1;
		}
		~WordFlip()
		{	
		};
};

class DwordFlip:public SequentialByteGrainedMutator
{
	protected:
		unsigned int _origValue;
	protected:
		char* ModifyData(char* modBegin, char* modEnd) override
		{
			_origValue = *((unsigned int *)modBegin);			
			*((unsigned int *)modBegin) ^= 0xFFFFFFFF;
			return modBegin;
		}		
		char* RestoreData(char* modBegin, char* modEnd) override
		{
			*((unsigned int *)modBegin) = _origValue;
			return modBegin;
		}		
	public:
		DwordFlip(TestCase data): SequentialByteGrainedMutator(data)
		{
			_origValue = 0xFF;
			_mutationSize = 4;
			_mutationPos = -1;
		}
		~DwordFlip()
		{	
		};
};
```

В представленной реализации на основе изначального тесткейса создавался экземпляр класса `Mutator`, который при вызове метода `Mutate` производил модификацию тесткейса. Последовательное инвертирование байта/слова/двойного слова осуществлялось в классах  `ByteFlip`, `WordFlip`,`DwordFlip` соответственно. С помощью метода `AdvancePosition` и `IsMutationFinished` код, использующий данные классы, мог контролировать, когда переходить к следующей мутации и получать сведения о том, завершен ли текущий цикл мутаций. Указанные классы хранили соответствующее внутреннее состояние. 

Код, использующий данные классы, выглядел так (показано на примере класса `WordFlip`):
```C++
void WalkingWord(TestCase &testCase, ByteMap &effectorMap)
{
	WordFlip wordFlip(testCase,&effectorMap);			
	LOG(plog::info) << "Starting";			
	while(!wordFlip.IsMutationFinished())
	{								
		RunSingleTestCase(wordFlip.Mutate());		
	}
	LOG(plog::info) << "Finished mutating testcase of length =" << testCase.size();			
}
``` 

##### Описание дизайна + попытка посмотреть на вопрос формально
Если посмотреть, с формальной стороны, то мутация - это некоторое отображение множества тесткейсов в себя `f: T->T`.  Помимо самого тесткейса, функция мутации может зависеть и от других параметров.  Например, bitflip  будет иметь вид `bitflip(testcase, bit_position) = new_testcase`; замена значения определенного байта на другое у нас будет иметь вид: `replace_byte(testcase, byte_position, new_value)`. Но если сводить мутацию к отображению заданного вида, то например ,  для мутации каждого бита на определенной позиции будет формально генерироваться отдельная функция.
Таким образом, можно сказать, что у нас имеется некоторая конечная последовательность функций, которую мы одна за другой применяем к тесткейсу. Но сами функции мутации нам не нужны, поэтому мы будем оперировать "ленивыми" последовательностями уже "мутированных" тесткейсов. 

#### Итоговый код
На основании приведенного выше описания дизайна код, реализующий  мутацию `ByteFlip` был переписан следующим образом: 
```C++

template <typename T>
class Sequence
{
public:
    virtual void next() = 0;
    virtual bool isFinished() = 0;
    virtual T get() = 0;
};

template <typename T>
class IncrementalSequence :public Sequence<T>
{
private:
    T value;
    T end;
public:
    IncrementalSequence(T begin, T end) : value(begin), end(end) {}
    virtual void next() override
    {
        ++value;
    }
    virtual bool isFinished() override
    {
        return value >= end;
    }
    virtual T get() override
    {
        return value;
    }
};

template <typename ArgType>
using MutationFunc = std::function<TestCase& (TestCase&, ArgType)>;

template <typename ArgType>
void ApplyMutationCycle(TestCase baseTestCase, Sequence<ArgType>& sequence, MutationFunc<ArgType> mutationFunc)
{
    for (; !sequence.isFinished(); sequence.next())
    {
        TestCase mutatedTestCase = baseTestCase;
        auto arg = sequence.get();
        mutatedTestCase = mutationFunc(mutatedTestCase, arg);
        ProcessTestcase(mutatedTestCase);
    }
};
...

TestCase& ByteFlipMutation(TestCase& testcase, size_t position)
{
    testcase[position] ^= 0xFF;
    return testcase;
};

void ApplyByteFlipsSequence(TestCase testcase)
{
    IncrementalSequence<size_t>  incrementalSequence(0, testcase.size());
    std::function<TestCase& (TestCase&, size_t)> byteFlipMutation = &ByteFlipMutation;
    ApplyMutationCycle<size_t>(testcase, incrementalSequence, &ByteFlipMutation);

    return;
}
```
Если обобщить эту операцию до инвертирования идущих подряд участков тесткейса  произвольной длины, то получим  код: 
```C++
template <typename T>
TestCase& ByteGrainedFlip(TestCase& testcase, size_t position)
{
    T* modifiedPart = (T*)(&testcase.at(position));
    (*modifiedPart) ^=  (~0);
    return testcase;
};

template <typename T>
void ApplyByteGrainedFlipSequence(TestCase testcase)
{
    IncrementalSequence<size_t>  incrementalSequence(0, testcase.size() - sizeof(T) + 1);
    ApplyMutationCycle<size_t>(testcase, incrementalSequence, &ByteGrainedFlip<T>);
}
```

Применение указанных циклов мутации выглядит следующим образом: 
```C++
...
    ApplyByteGrainedFlipSequence<char>(currentTestCase);
    ApplyByteGrainedFlipSequence<short>(currentTestCase);
    ApplyByteGrainedFlipSequence<int>(currentTestCase);
...
```

#### Выводы и рефлексия
Первое, что бросается в глаза после того, как код был переписан - это то, что он стал значительно компактнее. В исходном варианте он занимал 260 строк, после переделки - менее  70.  Также на мой взгляд, получилось сделать этот код декларативным, хотя бы по поведению, несмотря на то, что например класс `IncrementalSequence` хранит внутри себя состояние. В целом мы избавились от большого количества разнообразных классов, которые хранят в себе внутреннее состояние. Небольшие компоненты и функции стало проще тестировать. 

При этом, данная итерация не далась мне легко. В сумме она заняла порядка 15 часов работы. Было очень непривычно думать в таком стиле, пытаться понять, как на практике сделать свой код декларативным, хотя теорию ты знаешь. Настоящая "ломка" привычного мышления.

Свою роль также сыграло то, что я был слабо знаком с шаблонами и метапрограммированием на С++, особенно в последних стандартах, так что попутно изучил и эту тему. И безусловно еще  стоит предпринять усилия для того, чтобы сделать код еще выразительнее, практика покаывает, что это окупается. 


### Итерация № 2 
#### Вводная часть
После завершения первой итерации стало любопытно, насколько хорошо получится реализовать другие мутации поверх уже описанной логической основы.  В итоге в течение второй итерации я занялся реализацией других ранее упомнянутых алгоритмов мутации: 
1) арифметические операции - последовательное  сложение какого-либо байта/слова/двойного слова с заранее заданным набором констант. 
2) замена байта на "интересное" значение, вычисленное на основе того, как тестируемая программа реагирует на изменение значения "интересного" байта. 


#### Описание логики
 Результат мутации полностью определяется следующими аргументами: 
 1) исходным тесткейсом;
 2) функцией мутации;
 3) аргументами функции мутации;
 Т.е. операцию мутации можно записать так: `mutated_testcase = Mutate(testcase, mutationFunction, mutationFunctionArguments)`. 
 В каждом цикле мутации мы фиксируем исходный тесткейс и функцию мутации. Таким образом, последовательность измененных тесткейсов, которую мы получаем, зависит от последовательности аргументов -  `mutationFunctionArguments`. Именно их логику изменения и задает генератор последовательности аргументов. 
 По сути мы таким образом задаем отображение последовательности аргументов на последовательность измененных тесткейсов. 

Рассмотрим теперь детальнее те мутации, которые мы хотим реализовать. Саму операцию мутации можно представить как функцию с тремя аргументами: `F(testcase, position, constant)`. Например, в случае с арифметической операцией это будет выглядеть так: `F(testcase, position, constant){ testcase[position] += constant}`, а в случае с подстановкой "интереcной константы" это будет `F(testcase, position, constant) = { testcase[position] = constant }`. Теперь возникает вопрос, а как формировать последовательность аргументов для этих функций? 
Если в качестве констант взять, например набор из значений `{33, 44}`, то последовательность аргументов к функции мутации будет выглядеть так: 
`(position = 0, constant =  33); (position = 0, constant =  44); (position = 1, constant =  33); (position = 1, constant =  44) ...`
Т.е.  по сути мы получаем упорядоченное декартово произведение множества позиций и множества констант.    
Такая же логика применима например к операции bitflip, только там в качестве операции будет `xor`, а в качестве констант выступает единица, выставленная  на соответствующие позиции в байте. 

#### Изначальный код
Ниже представлен код указанных мутаций, который был написан изначально: 
```C++

class MultipleMutationsOnSingleDataChunk:public SequentialByteGrainedMutator
{			
	public:
	protected:	
		int _scale;
	protected:	
		virtual char* ModifyData(char* modBegin, char* modEnd) override
		{
			return modBegin;
		}
		virtual char* RestoreData(char* modBegin, char* modEnd) override
		{
			return modBegin;
		}
		virtual char* SaveData(char* modBegin, char* modEnd)
		{
			return modBegin;
		}
		virtual void SaveMutatedValue(int pos)
		{
			try
			{
				SaveData(&_data.at(pos), &_data.at(pos+_mutationSize-1));				
			}
			catch (std::out_of_range)
			{
			}	
		}
	public:
		MultipleMutationsOnSingleDataChunk(TestCase data)
		{			
			_mutationSize = 1;
			_mutationPos = 0;			
			_scale = 1;
			_range = (data.size()-_mutationSize+1)*_scale;
		}
		~MultipleMutationsOnSingleDataChunk()
		{	
		};
		virtual TestCase &Mutate() override
		{			
			if (IsMutationFinished())
				return _data;			
			
			int ind = _mutationPos/_scale;
							
			if (_mutationPos%_scale == 0) 
			{
				try
				{						
					RestorePreviouslyMutatedData(ind - 1);	
				}					
				catch (std::out_of_range)
				{}					
				SaveMutatedValue(ind);
			}				
			RestorePreviouslyMutatedData(ind);								
			MakeMutation(ind);				
				
			_mutationPos++;			
			return _data;
		};
		virtual bool IsMutationFinished() override
		{
			return _mutationPos >= _range;
		};		
};

class ByteArithmetic:public MultipleMutationsOnSingleDataChunk
{			
	public:
		static const char _maxOffset = 35;
		
	protected:
		unsigned char _origValue;
		
	protected:
		virtual char* ModifyData(char* modBegin, char* modEnd) override
		{		
			int offset = (_mutationPos%_scale) - _maxOffset;					
			char  mutatedVal = *modBegin;						
			mutatedVal += offset;
			*modBegin = mutatedVal;
			return modBegin;
		}
		virtual char* RestoreData(char* modBegin, char* modEnd) override
		{
			*modBegin = _origValue;
			return modBegin;
		}
		virtual char* SaveData(char* modBegin, char* modEnd)
		{
			_origValue = *modBegin;
			return modBegin;
		}

	public:
		ByteArithmetic(TestCase data)
		{
			try
			{
				_origValue = data.at(0);
			}
			catch (std::out_of_range)
			{
				_origValue = 0;
			}
			_mutationSize = 1;
			_mutationPos = 0;			
			_scale = 2*_maxOffset;			
			_range = data.size()*_scale;
		}
		~ByteArithmetic()
		{	
		};
};


class WordArithmetic:public MultipleMutationsOnSingleDataChunk
{			
	#define SWAP16(x) (((x) >> 8) | ((x) << 8))	
	public:
		static const unsigned char _maxOffset = 35;
		
	protected:
		unsigned short _origValue;	
	protected:	
		virtual char* ModifyData(char* modBegin, char* modEnd) override
		{			
			int offset = (_mutationPos%(_scale/2)) - _maxOffset;		
			unsigned short mutatedVal = *((unsigned short *)modBegin);			
			bool needToSwap =(_mutationPos%(_scale/2) < _mutationPos%(_scale));
			
			if (needToSwap) mutatedVal = SWAP16(mutatedVal);			
			mutatedVal+=offset;
			if (needToSwap) mutatedVal = SWAP16(mutatedVal);
			*((unsigned short *)modBegin) = mutatedVal;
			return modBegin;
		}
		virtual char* RestoreData(char* modBegin, char* modEnd) override
		{
			*((unsigned short *)modBegin) = _origValue;
			return modBegin;
		}
		virtual char* SaveData(char* modBegin, char* modEnd)
		{
			_origValue = *((unsigned short*)modBegin);
			return modBegin;
		}
	public:
		WordArithmetic(TestCase data)
		{
			_origValue = 0xFF;
			_mutationSize = 2;
			_mutationPos = 0;			
			_scale = 4*_maxOffset;
			_range = (data.size()-_mutationSize+1)*_scale;
		}
		~WordArithmetic()
		{	
		};
};

class DwordArithmetic:public MultipleMutationsOnSingleDataChunk
{			
	#define SWAP32(x) (((x) >> 24) | (((x) & 0x00FF0000) >> 8) | (((x) & 0x0000FF00) << 8) | ((x) << 24))
	public:
		static const unsigned char _maxOffset = 35;
		
	protected:
		unsigned __int32 _origValue;	
		//unsigned char _scale;
	protected:	
		virtual char* ModifyData(char* modBegin, char* modEnd) override
		{		
			int offset = (_mutationPos%(_scale/2)) - _maxOffset;					
			unsigned __int32 mutatedVal = *((unsigned __int32 *)modBegin);			
			bool needToSwap =(_mutationPos%(_scale/2) < _mutationPos%(_scale));
			
			if (needToSwap) mutatedVal = SWAP32(mutatedVal);						
			mutatedVal += offset;
			if (needToSwap) mutatedVal = SWAP32(mutatedVal);
			*((unsigned __int32 *)modBegin) = mutatedVal;
			
			return modBegin;
		}
		virtual char* RestoreData(char* modBegin, char* modEnd) override
		{
			*((unsigned __int32 *)modBegin) = _origValue;
			return modBegin;
		}
		virtual char* SaveData(char* modBegin, char* modEnd)
		{
			_origValue = *((unsigned __int32 *)modBegin);
			return modBegin;
		}
	public:
		DwordArithmetic(TestCase data)
		{
			_origValue = 0xFF;
			_mutationSize = sizeof(__int32);
			_mutationPos = 0;			
			_scale = 4*_maxOffset;
			_range = (data.size()-_mutationSize+1)*_scale;
		}
		~DwordArithmetic()
		{	
		};
};


class InterestingByte:public MultipleMutationsOnSingleDataChunk
{				
	protected:
		char _origValue;
	public:		
		const std::vector<char> _values = {
			  -128,          /* Overflow signed 8-bit when decremented  */ \
			  -1,            /*                                         */ \
			   0,            /*                                         */ \
			   1,            /*                                         */ \
			   16,           /* One-off with common buffer size         */ \
			   32,           /* One-off with common buffer size         */ \
			   64,           /* One-off with common buffer size         */ \
			   100,          /* One-off with common buffer size         */ \
			   127           /* Overflow signed 8-bit when incremented  */ 
		};
	protected:	
		virtual char* ModifyData(char* modBegin, char* modEnd) override
		{
			char mutatedVal = _values[_mutationPos%_scale];			
			*modBegin = mutatedVal;
			return modBegin;
		}
		virtual char* RestoreData(char* modBegin, char* modEnd) override
		{
			*modBegin = _origValue;
			return modBegin;
		}
		virtual char* SaveData(char* modBegin, char* modEnd) override
		{
			_origValue = *modBegin;
			return modBegin;
		}

	public:
		InterestingByte(TestCase data)
		{
			_origValue = 0xFF;
			_mutationSize = 1;
			_mutationPos = 0;			
			_scale = _values.size();
			_range = (data.size()-_mutationSize+1)*_scale;
		}
		~InterestingByte()
		{	
		};
};

class InterestingWord:public MultipleMutationsOnSingleDataChunk
{				
	protected:
		short _origValue;
	public:		
		const std::vector<short> _values = {
			  -32768,        /* Overflow signed 16-bit when decremented */
			  -129,          /* Overflow signed 8-bit                   */
			   128,          /* Overflow signed 8-bit                   */
			   255,          /* Overflow unsig 8-bit when incremented   */
			   256,          /* Overflow unsig 8-bit                    */
			   512,          /* One-off with common buffer size         */
			   1000,         /* One-off with common buffer size         */
			   1024,         /* One-off with common buffer size         */
			   4096,         /* One-off with common buffer size         */
			   32767         /* Overflow signed 16-bit when incremented */ 		
			};
	protected:	
		virtual char* ModifyData(char* modBegin, char* modEnd) override
		{
			short mutatedVal = _values[_mutationPos%_scale];			
			*((short *)modBegin) = mutatedVal;
			return modBegin;
		}
		virtual char* RestoreData(char* modBegin, char* modEnd) override
		{
			*((short *)modBegin) = _origValue;
			return modBegin;
		}
		virtual char* SaveData(char* modBegin, char* modEnd) override
		{
			_origValue = *((short *)modBegin);
			return modBegin;
		}

	public:
		InterestingWord(TestCase data)
		{
			_origValue = 0xFFFF;
			_mutationSize = 2;
			_mutationPos = 0;			
			_scale = _values.size();
			_range = (data.size()-_mutationSize+1)*_scale;
		}
		~InterestingWord()
		{	
		};
};

class InterestingDword:public MultipleMutationsOnSingleDataChunk
{				
	protected:
		__int32 _origValue;
	public:		
		const std::vector<__int32> _values = {
		  -2147483648LL, /* Overflow signed 32-bit when decremented */ 
		  -100663046,    /* Large negative number (endian-agnostic) */ 
		  -32769,        /* Overflow signed 16-bit                  */ 
		   32768,        /* Overflow signed 16-bit                  */ 
		   65535,        /* Overflow unsig 16-bit when incremented  */ 
		   65536,        /* Overflow unsig 16 bit                   */ 
		   100663045,    /* Large positive number (endian-agnostic) */ 
		   2147483647    /* Overflow signed 32-bit when incremented */
		};
	protected:	
		virtual char* ModifyData(char* modBegin, char* modEnd) override
		{
			__int32 mutatedVal = _values[_mutationPos%_scale];			
			*((__int32 *)modBegin) = mutatedVal;
			return modBegin;
		}
		virtual char* RestoreData(char* modBegin, char* modEnd) override
		{
			*((__int32 *)modBegin) = _origValue;
			return modBegin;
		}
		virtual char* SaveData(char* modBegin, char* modEnd) override
		{
			_origValue = *((__int32 *)modBegin);
			return modBegin;
		}

	public:
		InterestingDword(TestCase data)
		{
			_origValue = 0xFFFFFFFF;
			_mutationSize = sizeof(__int32);
			_mutationPos = 0;			
			_scale = _values.size();
			_range = (data.size()-_mutationSize+1)*_scale;
		}
		~InterestingDword()
		{	
		};
};

```
В исходном коде мы реализовали 2 типа мутации для 3 разных размеров - сложение и подстановка с типами byte/word/dword. В результате мы получили 7 классов, включая класс `MultipleMutationsOnSingleDataChunk` , который является базовым для них и реализует основную логику последовательной модификации одного непрерывного участка тесткейса и перехода к следующему участку, когда мутации исчерпаны. 
#### Итоговый код
Описанные операции были реализованы заново в рамках обозначенной логики. Был введен отдельный тип для аргумента функции мутации и реализованы операции его изменения (operator++). Также введен отдельный тип `Mutation` , который объединяет в себе тип аргумента мутации, ограничения на диапазон его изменения. Для того, чтобы "откатывать" внесенные на каждой итерации изменения была введена функция "восстановления" тесткейса, которая зависит от аргументов и типа мутации. Это нужно, чтобы избежать копирования исходного тесткейса на каждой итерации. 

```C++
template<typename T>
struct ContinuousChunkMutationArgument
{
    size_t position;
    size_t constIndex;
    std::vector<T> constants;

public:
    ContinuousChunkMutationArgument(std::vector<T> constants, size_t position = 0, size_t constIndex = 0) :
        constants(constants), position(position), constIndex(constIndex)
    {
    };
    bool operator >= (ContinuousChunkMutationArgument &other)
    {
        if (position == other.position)
            return constIndex >= other.constIndex;
        
        return (position > other.position);
    }
    ContinuousChunkMutationArgument& operator++ ()
    {
        constIndex = (constIndex + 1) % constants.size();
        if (!constIndex)
            position++;
        return *this;
    }
};

template <typename T>
struct ContinuousChunkMutationRestoreInfo
{
    size_t position;
    T originalValue;
};


template <typename T>
using  ContinuousChunkRestorableMutationResult = std::pair<TestCase&, ContinuousChunkMutationRestoreInfo<T>> ;

template <typename T, typename ArgType>
using MutateContinuousChunk = std::function< T (T, ArgType)>;

template <typename T, typename ArgType>
ContinuousChunkRestorableMutationResult<T> MutationContinuousChunk(TestCase& testcase, ArgType arg, MutateContinuousChunk<T, ArgType> mutateChunk)
{
    T* modifiedPart = (T*)(&testcase.at(arg.position));

    ContinuousChunkMutationRestoreInfo<T> restoreData;
    restoreData.position = arg.position;
    restoreData.originalValue = *modifiedPart;

    *modifiedPart = mutateChunk(*modifiedPart, arg);

    return ContinuousChunkRestorableMutationResult<T>(testcase, restoreData);
}

template <typename T>
TestCase& MutationContinuousChunkRestore(TestCase& testcase, ContinuousChunkMutationRestoreInfo<T> restoreData)
{
    T* modifiedPart = (T*)(&testcase.at(restoreData.position));
    *modifiedPart = restoreData.originalValue;
    return testcase;
}


template <typename T>
struct ContinuousChunkMutation
{
    typedef  ContinuousChunkMutationArgument<T> ArgType;
    typedef  T BasicType;
    typedef  std::function<T (T,T)> Operation;

public: 
    const ContinuousChunkMutationArgument<T> lowerBound;
    const ContinuousChunkMutationArgument<T> upperBound;

public:
    ContinuousChunkMutation(TestCase & basicTestCase, std::vector<T> constants) : lowerBound(constants, 0),
        upperBound(constants, basicTestCase.size() - sizeof(T), constants.size())
    {
    }

    static T Mutate(T modifiedPart, ContinuousChunkMutationArgument<T> arg, Operation operation) 
    {
        return operation(modifiedPart, arg.constants[arg.constIndex]);
    }
};

template<typename T>
T SetVal(T a, T b)
{
    return b;
}

template<typename T>
T Add(T a, T b)
{
    return a + b;
}
```

Функция, реализующая цикл мутации с использованием указанных типов выглядит следующим образом: 
```C++
template <typename MutationType>
void MakeIncrementalRestorableMutationsOnContinuousChunks(TestCase testcase, 
                                                          MutationType& mutation, typename MutationType::Operation operation)
{
    using ArgType = typename MutationType::ArgType;
    using T = typename MutationType::BasicType;

    IncrementalSequence<ArgType> incrementalSeq(mutation.lowerBound, mutation.upperBound);

    using namespace std::placeholders;
    auto mutateOp = [operation, mutation](T mutatedPart, ArgType arg) { return mutation.Mutate(mutatedPart, arg, operation); };
    auto setConst = std::bind(MutationContinuousChunk<T, ArgType>, _1, _2, mutateOp);

    ApplyRestorableMutationCycle<ArgType, ContinuousChunkMutationRestoreInfo<T>>(testcase, incrementalSeq,
        setConst, &MutationContinuousChunkRestore<T>);

    return;
}
```

Применение данного цикла приведено ниже:
```C++
template <typename T>
void Arythmetic(TestCase testcase, std::vector<T> consts)
{
    ContinuousChunkMutation<T> mutation(testcase, consts);
    MakeIncrementalRestorableMutationsOnContinuousChunks<ContinuousChunkMutation<T>>(testcase, mutation, Add<T>);
    MakeIncrementalRestorableMutationsOnContinuousChunks<ContinuousChunkMutation<T>>(testcase, mutation, SetVal<T>);
}

template <typename T>
void InterestingValues(TestCase testcase, std::vector<T> consts)
{
    ContinuousChunkMutation<T> mutation(testcase, consts);
    MakeIncrementalRestorableMutationsOnContinuousChunks<ContinuousChunkMutation<T>>(testcase, mutation, Add<T>);
    MakeIncrementalRestorableMutationsOnContinuousChunks<ContinuousChunkMutation<T>>(testcase, mutation, SetVal<T>);
}

void ApplyMutations(TestCase testcase)
{
	std::vector<char> arythmeticConsts = { -1, -16, 128 };
    Arythmetic<char>(testcase, consts);
    
	std::vector<char> interestingBytes = { -128,-1, 0, 1, 16, 32, 64, 100, 127   };    
    InterestingValues<char>(testcase, interestingVals);
}
```

#### Выводы и рефлексия. 
Как и в прошлой итерации значительно уменьшился объем кода. В исходной версии код мутаторов занимал 350 строк, в текущей - 150 вместе с функциями, реализующими цикл мутации.
Также мы избавились от множества объектов, с внутренним состоянием, сделав код декларативным, насколько это возможно. Вероятно, еще есть возможности для повышения его ясности и компактности, для этого занялся более глубоким изучением шаблонов на C++.
На второй итерации процесс занимал значительно меньше времени. Всего на реализацию понадобилось порядка 3.5 часов, включая изучение особенностей нового стандарта С++. 

## Итерация 3. 
### Введение. Effector map.
Для каждого тесткейса есть некоторые "интересные" байты, модификация которых влияет на поведение тестируемой программы. Это может быть например сигнатура, которая кодирует формат нижележащего файла, crc32 какого-либо участка данного файла. Также есть "неинтересные" места, модификация которых в тесткейсе никак не влияет на поведение тестируемой программы, т.е. с нашей точки зрения  не меняет "покрытия". Например это могут быть байты выравнивания в том или ином блоке файла. Таким образом, во время проведения bitflip-ов мы можем вычислить некую "карту", которая будет показывать те байты, изменение которых влияет на поведение программы. Соответственно на последующих стадиях мы можем пропустить модификацию байт, которые на поведение программы не влияют. Такая карта в коде afl  называется "effector map". 

### Описание логики
С формальной стороны  "effector map"  можно представить как  предикат `eff: (T,T)->{true;false}`. `eff(t_base, t)` которое показывает, будет ли тесткейс `t`  эффективным, по отношению к тесткейсу `t_base` т.е. будет ли покрытие, полученное в результате выполнения, `t` отличаться от покрытия, полученного в результате выполнения `t_base`.  В целом предикат `eff` может зависит в том числе от характера мутации, её аргументов и тесткейса, который мы меняем. Также важная особенность состоит в том, что сам этот предикат зависит от результатов выполнения мутированных тесткейсов, в тестируемой программе и не может быть задан на этапе компиляции. Соответственно нам нужен какой-то механизм, который будет каким-то образом строить этот предикат, в зависимости от результатов запуска тесткейса. 

### Изначальный код
Изначально идея effector map была реализована в виде массива байт того же размера, что и тесткейс. Если мутация i-го байта тесткейса была эффективной, т.е. покрытие изменилось по сравнению с исходным, то `effector_map[i] = 1`. В противном случае, оно остается нулевым. 
Для этого в приведенной в итерации 2 иерархии классов был добавлен дополнительный класс `SequentialEffectorMutator`, от которого наследовались классы, непосредственно реалилзующие мутации.  
```C++
class SequentialEffectorMutator:public SequentialMutator
{
	protected:
		TestCase _data;	
		ByteMap _effectorMap;
		int _range;
		int _mutationPos;
		// count of successive bytes changed by sungle mutation;
		int _mutationSize;
		bool _useEffectorMap;
	protected:		
		// modification area begin
		// modification ared end
		virtual char* ModifyData(char* modBegin, char* modEnd)
		{
			return modBegin;
		}
		virtual void MakeMutation(int mutationPos)
		{				
			try
			{								
				// get original value ptr
				ModifyData(&_data.at(mutationPos), &_data.at(mutationPos+_mutationSize-1));
			}
			catch (std::out_of_range)
			{				
				std::cout<<"don't mutate" << std::endl;			
			}
		}
		virtual char* RestoreData(char* modBegin, char* modEnd)
		{
			return modBegin;
		}
		
		virtual void RestorePreviouslyMutatedData(int restorePos)
		{			
			try
			{								
				// get original value ptr
				RestoreData(&_data.at(restorePos), &_data.at(restorePos+_mutationSize-1));
			}
			catch (std::out_of_range)
			{
				std::cout<<"don't restore" << std::endl;
			}
		}
		//advances position of mutated data if it's necessary
		//if we perform multiple mutations on single piece of data
		virtual void inline AdvancePosition()
		{
			_mutationPos++;
		}
		//
		virtual void inline SkipMutation()
		{
			_mutationPos++;
		}
		
	public:
		SequentialEffectorMutator(TestCase data, ByteMap * effectorMap = NULL)
		{			
			if (effectorMap==NULL)
			{
				_effectorMap.resize(data.size());			
				std::fill(_effectorMap.begin(), _effectorMap.end(), 0);
				_useEffectorMap = false;
			}
			else if (effectorMap->size()!=data.size())
			{
				throw_runtime_error("ASSERT: Effector map size doesn't match test case size");
			}
			else				
			{
				_effectorMap = *effectorMap;
				_useEffectorMap = true;
			}
			_data = data;
			_range = data.size();			
			_mutationSize = 1;
			_mutationPos = -1;
		};
		virtual ~SequentialEffectorMutator()
		{
			
		};

		virtual void MutationIsEffective()
		{
			if (_mutationSize > 1 || _useEffectorMap)
				return;
			try
			{
				_effectorMap.at(_mutationPos) = 1;
			}
			catch (std::out_of_range)
			{}
			
		}		
		virtual TestCase &Mutate() 
		{	
			if (IsMutationFinished())
				return _data;
			
			RestorePreviouslyMutatedData(_mutationPos);			
			if (_useEffectorMap)
			{
				try
				{
					do 
					{
						SkipMutation();						
					} while (!_effectorMap.at(_mutationPos));
				}
				catch (std::out_of_range)
				{
					SkipMutation();
				};
			}
			else
				AdvancePosition();
			
			MakeMutation(_mutationPos);	
			return _data;
		};
		virtual bool IsMutationFinished()
		{
			return _mutationPos+_mutationSize-1 >= _range;
		};		

		virtual ByteMap  GetEffectorMap()
		{			
			return _effectorMap;
		};		
};
```

Соответственно теперь код  каждого класса, реализующего мутации типа byteFlip, арифметические операции, замену на константы, был изменен - данные классы были унаследованы не от `SequentialByteGrainedMutator`, а от данного класса - `SequentialEffectorMutator`.
Приведу для примера измененный код классов Byteflip, Wordflip, DwordFlip:
```C++

class ByteFlip:public SequentialEffectorMutator
{
	protected:
		unsigned int _origValue;
	protected:
		// perform dword flip
		char* ModifyData(char* modBegin, char* modEnd) override
		{
			_origValue = *modBegin;			
			*modBegin ^= 0xFF;
			return modBegin;
		}		
		char* RestoreData(char* modBegin, char* modEnd) override
		{
			*modBegin = _origValue;
			return modBegin;
		}		
	public:
		ByteFlip(TestCase data, ByteMap * effectorMap = NULL): SequentialEffectorMutator(data, effectorMap)
		{
			_origValue = 0xFF;
			_mutationSize = 1;
			_mutationPos = -1;
		}
		~ByteFlip()
		{	
		};
};

class WordFlip:public SequentialEffectorMutator
{
	protected:
		unsigned short _origValue;
	protected:
		// perform dword flip
		char* ModifyData(char* modBegin, char* modEnd) override
		{
			_origValue = *((unsigned int *)modBegin);			
			*((unsigned short *)modBegin) ^= 0xFFFFFFFF;
			return modBegin;
		}		
		char* RestoreData(char* modBegin, char* modEnd) override
		{
			*((unsigned short *)modBegin) = _origValue;
			return modBegin;
		}		
	public:
		WordFlip(TestCase data, ByteMap * effectorMap = NULL): SequentialEffectorMutator(data, effectorMap)
		{
			_origValue = 0xFF;
			_mutationSize = 2;
			_mutationPos = -1;
		}
		~WordFlip()
		{	
		};
};

class DwordFlip:public SequentialEffectorMutator
{
	protected:
		unsigned int _origValue;
	protected:
		// perform dword flip
		char* ModifyData(char* modBegin, char* modEnd) override
		{
			_origValue = *((unsigned int *)modBegin);			
			*((unsigned int *)modBegin) ^= 0xFFFFFFFF;
			return modBegin;
		}		
		char* RestoreData(char* modBegin, char* modEnd) override
		{
			*((unsigned int *)modBegin) = _origValue;
			return modBegin;
		}		
	public:
		DwordFlip(TestCase data, ByteMap * effectorMap = NULL): SequentialEffectorMutator(data, effectorMap)
		{
			_origValue = 0xFF;
			_mutationSize = 4;
			_mutationPos = -1;
		}
		~DwordFlip()
		{	
		};
};
```
### Измененный код. 
Исходя из приведенного описания дизайна, я действовал таким образом: создал класс, который  реализует в себе операцию "настраиваемого" предиката. Он по умолчанию всегда возвращает false, однако к нему можно добавлять аргументы, при передаче которых он будет возвращать значение true. Формально это можно представить как создание нового предиката путем применения операции дизъюнкции: 
`new_predicate = old_predicate V always_true(arg_value)`. 
Тогда значение `new_predicate(arg_value)` будет всегда истинным. Пока не до конца разобрался, как эту идею можно лучше всего выразить, поэтому написал код предиката следующим образом: 
```C++
template  <typename ArgType>
struct ExtendablePositionalPredicate
{
    const unsigned long long minSize = 0x1000;
    std::vector<bool> values;
public:
    ExtendablePositionalPredicate(TestCase &testcase) : values(testcase.size(), false){};

    ExtendablePositionalPredicate& AddTrueValue(ArgType arg)
    {
        auto position = arg.position;
        if (position > values.size())
            values.resize(values.size() + std::max(minSize, position - values.size()),  false);
        values[position] = true;
        return *this; 
    };

    bool operator()(ArgType arg)
    {
        auto position = arg.position;
        if (position > values.size())
            return false;
        return values[position];
    }
};
```
Также написал новый генератор последовательностей, который может принимать предикат и на основании его значения "пропускать" те члены последовательности, которым он не удовлетворяет. 
```C++
template <typename T, typename PredicateType>
class IncrementalFilteredSequence :public Sequence<T>
{
private:
    T value;
    T end;
    PredicateType predicate;
public:
    IncrementalFilteredSequence(T begin, T end, PredicateType predicate) : value(begin), end(end), predicate(predicate)
    {
    }
    virtual void next() override
    {
        ++value;
        while (!isFinished() && !predicate(value))
            ++value;
    }
    virtual bool isFinished() override
    {
        return value >= end;
    }
    virtual T get() override
    {
        return value;
    }
};
```

Также дополнил цикл мутации тесткейсов логикой генерации предикатов: 
```C++
template <typename MutationType>
struct PredicateGeneratingMutationCycle
{
    using ArgType = typename MutationType::ArgType;
    using RestoreArgType = typename  MutationType::RestoreArgType;
    using RestorableMutationFunc = typename  MutationType::RestorableMutationFunc;
    using RestoreFunc = typename  MutationType::RestoreFunc;

public:
    static ExtendablePositionalPredicate<ArgType> Apply(TestCase baseTestCase, Sequence<ArgType>& sequence,
                                                RestorableMutationFunc mutate, RestoreFunc restore)
    {
        ExtendablePositionalPredicate<ArgType> isMutationEffective(baseTestCase);
        auto resultBase = ProcessTestcase(baseTestCase);
        
        for (; !sequence.isFinished(); sequence.next())
        {
            auto arg = sequence.get();
            auto [mutatedTestCase, restoreData] = mutate(baseTestCase, arg);
            
            auto result ProcessTestcase(mutatedTestCase);
            if (result.coverage != resultBase.coverage)
	            isMutationEffective.AddTrueValue(arg);

            baseTestCase = restore(mutatedTestCase, restoreData);
        }
        return isMutationEffective;
    };
};
```

В итоге код создания и применения такого предиката имеет следующий вид: 
```C++
template <typename T>
void ApplyMutationCycleGeneratePredicate(TestCase testcase, std::vector<T> consts, ContinuousChunkMutation<T>::Operation operation)
{
    using MutationType =  ContinuousChunkMutation<T>;
    using ArgType = typename MutationType::ArgType;

    MutationType mutation(testcase, consts);
    IncrementalSequence<ArgType> incrementalSeq(mutation.lowerBound, mutation.upperBound);

    using namespace std::placeholders;
    auto mutateOp = [operation, mutation](T mutatedPart, ArgType arg) { return mutation.Mutate(mutatedPart, arg, operation); };
    auto mutate = std::bind(mutation.MakeMutation, _1, _2, mutateOp);

    auto isMutationEffective = PredicateGeneratingMutationCycle<MutationType>::Apply(testcase, incrementalSeq,
            mutate, mutation.Restore);
}

template<typename T>
void TestEffectorMutation(TestCase & testcase, std::vector<T> consts, ContinuousChunkMutation<T>::Operation operation, ExtendablePositionalPredicate<ArgType> predicate)
{
    using MutationType =  ContinuousChunkMutation<T>;
    using ArgType = typename MutationType::ArgType;
    
	MutationType mutation(testcase, consts);
	auto mutateOp = [operation, mutation](T mutatedPart, ArgType arg) { return mutation.Mutate(mutatedPart, arg, operation); };
    auto mutate = std::bind(mutation.MakeMutation, _1, _2, mutateOp);
    
	IncrementalFilteredSequence<ArgType, ExtendablePositionalPredicate<ArgType>> incrementalSeqFiltered(mutation.lowerBound, mutation.upperBound, predicate);
	ApplyRestorableMutationCycle<ArgType, ContinuousChunkMutationRestoreInfo<T>>(testcase, incrementalSeqFiltered, mutate, mutation.Restore);
}

...
auto predicate = ApplyMutationCycleGeneratePredicate<char>(testCase, consts, Add<char>);
TestEffectorMutation<char>(testCase, consts, SetVal<char>, predicate);

```


#### Выводы и рефлексия. 
Несмотря на то, что объем кода, который пришлось добавить был в этот раз довольно значительным, все же не потребовалось как-то вмешиваться и переделывать уже написанные функции, все по сути было создано комбинированием уже существующих. Важным вопросом остается то, как повысить ясность и лаконичность кода, этому надо уделить очень много внимания и хорошо изучить шаблоны С++. 
Внесение изменений потребовало около 4 часов, часть из этого времени - разбирался с многостраничными ошибками компилятора. Однако, в этом есть плюс - если оно скомпилировалось и прошли тесты - ты можешь с высокой долей уверенности сказать, что оно работает так, как и было задумано. 
