## Введение
Данную работу я решил проделать на примере самописного coverage-guided - фаззера, который пишу для себя, чтобы разобраться в теме и применять в своих экспериментах. Данный фаззер пишется на С++ и изначально был некоторой калькой с afl, обернутой в классы.  На примере данного проекта я постарался применить и усвоить подходы к мышлению о программе. Первым этапом  я просто скопировал часть  функциональности afl, ведя разработку с применением TDD. Затем я немного отошел от кодинга и попытался описать данную систему словесно, выделить её основные элементы и представить, как они взаимодействуют между собой. Подумать о программе на 3 уровне. Результат представлен ниже.  

## Описание программы
Данное описание было сделано после первого цикла написания фаззера, когда я просто копировал afl, оборачивая его в классы. 

Фаззер в моем исполнении представляет собой некий вариант coverage-guided - фаззинга, когда среди всех возможных тесткейсов отбираются для использование те, на основе которых генерируется какое-то новое покрытие. 
Цикл работы фаззера на самом верхнем уровне можно описать следующим образом: 
1) Запуск тестируемого приложения с каким-то вариантом входных данных.  Получение результата запуска приложения. 
2) Получение результата запуска тестируемого приложения. Включает в себя сведения о том, как отработало приложения на заданных входных данных.  Также дополнительно может включать в себя сведения о некоторых характеристиках, полученных в результате запуска приложения с заданными тестовыми данными. Одной из таких характеристик может быть некоторое  "покрытие", которое дает заданный тестовый кейс. 
3) Генерацию новых входных данных (тесткейсов). Повторение цикла заново. 

Таким образом мы можем выделить несколько сущностей: 
1) Тестируемое приложение.
2) Подаваемые на вход тестовые данные - они же тесткейсы.  
3) Результат запуска тестируемого приложения. 
	1) Характеристики запуска приложения. Зависят от входных данных, тестируемого приложения, номера итерации и возможных других случайных факторов, обусловленных внутренним состоянием тестируемого приложения, которое может меняться от запуска к запуску.  "Покрытие" - вид указанной характеристики. 

В самом простом случае код, который генерирует тесткейсы не знает о том, что за тестовое приложение у нас работает, с каким форматом данных оно работает. Но в более сложных вариантах такие возникновение таких зависимостей возможно, как например в следующих случаях: 
1) отбор тестовых кейсов для дальнейшей мутации на основе результатов выполнения тестируемого приложения;    
2) генерация тестовых кейсов на основе выходных данных  информации некоторого тестового приложения. 
3) учет иных сведений при генерации/отборе тестовых кейсов: состояние того или иного участка памяти тестируемого приложения; результат работы той или иной функции; результат работы различных дополнительных средств типа сантайзера, и.т.д. 

Также просто запуск тестового приложения сам по себе является  объемной  задачей, особенно если  мы, например,  хотим использовать какой-то вариант динамической инструментации, которая требует специфических средств наподобие DynamoRio или Pin.  И эту логику целесообразно выделить в отдельный компонент, который будет отвечает за этот процесс и при необходимости поддерживать связанное с этим внутреннее состояние. Этот компонент будет знать такие вещи, как:
1) какое приложение с какими параметрами запускается
2) какая инструментация используется 
3) как взаимодействовать с тестируемым приложением
4) в каком окружении запускать тестируемое приложение 
5) каким образом передавать входные данные тестируемому приложению.
6) и.т.д. 
Пока что назовем этот компонент: `test_app_runner`.  

В виде псевдокода данный алгоритм можно представить в таком виде:    
```C++
while (not all_testcases_processed)
{
	run_result = run(tested_application, test_case,
				     tested_application_unknown_internal_state);
	test_case = make_test_case(run_result);
}
```
`tested_application_unknown_internal_state` в коде мы не будем учитывать, просто надо иметь в виду, что оно может оказывать влияние на результат запуска тестового приложения. 

Также отсюда следует ограничение: формат `run_result` должен быть общим для "условной" функции  `run` и `make_test_case`. 

После того, как я сделал это описание, появилось несколько мыслей о том, как правильнее построить фаззер. Приведу несколько примеров. 

## Пример № 1
Изначально я писал фаззер как некую "кальку" c afl не особо задумываясь над тем, как правильнее было бы его построить исходя например из целей расширяемости функционала фаззера. 

Например, результат выполнения тестового кейса - `run_result` в предыдущем коде я представил следующим образом (сами unit-тесты не привожу для компактности изложения): 
```C++
class RunResultImpl: public RunResult
{
private:
	bool _isCrashed;
	bool _isHanged;
	Coverage _coverage;
	
public:
	RunResultImpl(Coverage &coverage, bool isHanged, bool isCrashed):_coverage(coverage), _isHanged(isHanged), _isCrashed(isCrashed)
	{
	};
	virtual ~RunResultImpl()
	{
	};
	virtual Coverage GetCoverage() override
	{	
		return _coverage;
	};		
	virtual bool IsCrashOccured() override 
	{
		return _isCrashed;
	};
	virtual bool IsHangOccured() override 
	{	
		return _isHanged;
	};
};
```

В результате я перешел к более лаконичной форме: 
```C++
class RunResult
{
public:
	bool isCrashed;
	bool isHanged;
	Coverage coverage;
}
```
Ушел от наследования от единого интерфейса, фактически перешел просто к структуре. Ввиду того, что `RunResult` может быть довольно изменчивой, содержать разные поля, в зависимости от того, какие данные требуются для генерации следующего тесткейса. Если оформлять её как иерархию классов это может запутать её и возможно поспособствует появлению каких-то неочевидных зависимостей, в связи с чем здесь я сделал шаг с сторону простоты. 

## Пример № 2.
Также до того, как я сделал вышеприведенное условное описание системы, само понятие "покрытие" не было явно выделено. Подразумевалось, что оно имеется, но не было явного понимания его основных свойств. 

Ранее для работы с покрытием я применял отдельный класс, который называл `CoverageObserverHelper`. Он включал в себя основные действия с покрытием: 
1) проверить, что покрытие содержит в себе какие-то новые элементы по сравнению со старым
2) объединить покрытия 
3) сравнить покрытия (одинаковые или нет). 
При этом подразумевалось, что покрытие всегда имеет вид массива размером 4096 байт (одна страница в памяти), и есть некоторое  отображение адресов базовых блоков тестируемой программы в данный массив и это как раз дает некую характеристику того, как хорошо покрывается тестируемый код.  

Ниже частично приведен пример класса `CoverageObserver`, написанный с применением TDD (некоторые функции опущены): 
```C++

	class CoverageObserverHelper
	{
		private:						
			char* _coverageVirginMap;
			char* _initialPath;
			int checkSum;
		public:			
			static const unsigned int mapSizePow2 = 16;
			static const unsigned int mapSize = 1 << mapSizePow2;
			static const unsigned int hashConst = 0xa5b35705;
		private:
			/* Check if the current execution path brings anything new to the table.
			Update virgin bits to reflect the finds. Returns 1 if the only change is
			the hit-count for a particular tuple; 2 if there are new tuples seen. 
			Updates the map, so subsequent calls will always return 0.
			
			This function is called after every exec() on a fairly large buffer, so
			it needs to be fast. We do this in 32-bit and 64-bit flavors. */
			inline static char UpdateIfNewBits(char* virgin_map, const char* trace_bits) 
			{
				  __int32* current = (__int32 *)trace_bits;
				  __int32* virgin  = (__int32 *)virgin_map;
				  __int32  i = (mapSize >> 2);
				unsigned char ret = 0;

				while (i--)
				{
					/* Optimize for (*current & *virgin) == 0 - i.e., no bits in current bitmap
					   that have not been already cleared from the virgin map - since this will
					   almost always be the case. */
					   
					if (*current && (*current & *virgin))
					{
						if (ret < 2)
						{
							char* cur = (char*)current;
							char* vir = (char*)virgin;
						/* Looks like we have not found any new bytes yet; see if any non-zero bytes in current[] are pristine in virgin[]. */ 

						if ((cur[0] && vir[0] == 0xff) || (cur[1] && vir[1] == 0xff) ||
							(cur[2] && vir[2] == 0xff) || (cur[3] && vir[3] == 0xff)) 
							ret = 2;
						else 
							ret = 1;						
						}
						*virgin &= ~*current;
					}
					current++;
					virgin++;
				}				  
				return ret;
			}
			
			/* Check if the current execution path brings anything new to the table.
			Returns 1 if the only change is the hit-count for a particular tuple; 2 if there are new tuples seen. Doesn't update the map.*/
			inline static char HasNewBits(char* virgin_map, const char* trace_bits) 
			{
				  __int32* current = (__int32 *)trace_bits;
				  __int32* virgin  = (__int32 *)virgin_map;

				  __int32  i = (mapSize >> 2);
				unsigned char ret = 0;

				while (i--)
				{
					/* Optimize for (*current & *virgin) == 0 - i.e., no bits in current bitmap
					   that have not been already cleared from the virgin map - since this will
					   almost always be the case. */
					   
					if (*current && (*current & *virgin))
					{
						if (ret < 2)
						{
							char* cur = (char*)current;
							char* vir = (char*)virgin;
						/* Looks like we have not found any new bytes yet; see if any non-zero bytes in current[] are pristine in virgin[]. */ 
						if ((cur[0] && vir[0] == 0xff) || (cur[1] && vir[1] == 0xff) ||
							(cur[2] && vir[2] == 0xff) || (cur[3] && vir[3] == 0xff)) 
							ret = 2;
						else 
							ret = 1;
						}						
					}
					current++;
					virgin++;
				}				  
				return ret;
			}
		public:	
			CoverageObserverHelper()
			{				
				_coverageVirginMap = new char[mapSize];
				_initialPath = new char[mapSize];
				checkSum = 0;
				memset(_coverageVirginMap, 0xff, mapSize);
				memset(_initialPath, 0, mapSize);
			}
			CoverageObserverHelper(const CoverageObserverHelper& observer)
			{
				_coverageVirginMap =  new char[mapSize];
				_initialPath = new char[mapSize];
				checkSum = 0;
				std::memcpy(_coverageVirginMap, observer._coverageVirginMap, mapSize);
				std::memcpy(_initialPath, observer._initialPath, mapSize);
			}
			CoverageObserverHelper(TestRunner &testRunner)
			{
				_coverageVirginMap =  new char[mapSize];
				_initialPath = new char[mapSize];
				checkSum = 0;
				memset(_coverageVirginMap, 0xff, mapSize);
				std::memcpy(_initialPath, testRunner.GetTraceMap(), mapSize);
			}
			
			CoverageObserverHelper operator=(const CoverageObserverHelper& observer)
			{
				std::memcpy(_coverageVirginMap, observer._coverageVirginMap, mapSize);
				std::memcpy(_initialPath, observer._initialPath, mapSize);
				return *this;
			};
			
			~CoverageObserverHelper()
			{
				delete []_coverageVirginMap;
				delete []_initialPath;
			}
			
			/* If there is new coverage CoverageObserverHelper changes it's state, so that  two subsequent checks of the same cobverage map will yield to different results:
				first: observer.IsNewCoverage(map) - true if map has new coverage
				second:  observer.IsNewCoverage(map) - false, the state of observer changed */ 
			bool AdoptPathIfNewCoverage(const char * coverageMap)
			{
				char res = UpdateIfNewBits(_coverageVirginMap, coverageMap);
				return (res==1 || res==2);
			}
			bool AdoptPathIfNewCoverage(TestRunner *testRunner)
			{
				return AdoptPathIfNewCoverage(testRunner->GetTraceMap());
			}
			bool HasNewCoverage(const char * coverageMap)
			{
				return HasNewBits(_coverageVirginMap, coverageMap);
			}
			bool HasNewCoverage(TestRunner *testRunner)
			{
				return HasNewBits(_coverageVirginMap, testRunner->GetTraceMap());
			}			
			bool IsSamePath(const char* coverageMap)
			{
				return hash32(_initialPath, mapSize, hashConst) == hash32(coverageMap, mapSize, hashConst);
			}
			bool IsSamePath(TestRunner *testRunner)
			{
				return IsSamePath(testRunner->GetTraceMap());
			}
	};
```

Импровизированные юнит-тесты для этого  класса выглядели так: 
```C++
void TestCoverage()
{
	try
	{
		CoverageObserverHelper coverageObserver;
		unsigned int mapSize = coverageObserver.mapSize;
		std::vector<char> coverageMap(mapSize, 0);			
		//1. test no new instrumentation with initial vector		
		if (coverageObserver.AdoptPathIfNewCoverage(coverageMap.data()))
		{
			LOG(plog::error) << "test for no new coverage with initial vector failed\n";
			return;
		}
		//2. test if there is a new instrumentation
		unsigned int i = mapSize/2-1; //simply choose index less than mapsize;
		coverageMap[i]++;
		if (!coverageObserver.AdoptPathIfNewCoverage(coverageMap.data()))
		{
			LOG(plog::error)<< "test new coverage with index" << i <<"failed\n";
			return;
		}
		//3. new coverage with no new edges
		coverageMap[i]++;
		if (!coverageObserver.AdoptPathIfNewCoverage(coverageMap.data()))
		{
			LOG(plog::error)<< "test new coverage with no new tuples (index)" << i <<" failed\n";
			return;
		}
		
		//4. test if there is no new instrumentation discovered
		if (int ret = coverageObserver.AdoptPathIfNewCoverage(coverageMap.data()))
		{
			LOG(plog::error) << "test no new on index " << i << " "<<ret <<" failed\n";
			return;
		}
		
		if (coverageObserver.HasNewCoverage(coverageMap.data()))
		{
			LOG(plog::error) << "HasNewCoverage method test failed: false new coverage detected " << std::endl;
			return;
		}
		coverageMap[mapSize/3-1]++;
		if (!coverageObserver.HasNewCoverage(coverageMap.data()))
		{
			LOG(plog::error) << "HasNewCoverage method test failed: no new coverage detected where" << std::endl;
			return;
		}
		if (!coverageObserver.HasNewCoverage(coverageMap.data()))
		{
			LOG(plog::error) << "HasNewCoverage method test failed: subsequent calls returned different results" << std::endl;
			return;
		}
		
		std::vector<char> coverageMap_2(mapSize, 0);			
		if (!coverageObserver.IsSamePath(coverageMap_2.data()))
		{
			LOG(plog::error) << "IsSamePath method test failed: equal paths treated like unequal" << std::endl;
			return;
		}
		coverageMap_2[1]++;
		if (coverageObserver.IsSamePath(coverageMap_2.data()))
		{
			LOG(plog::error) << "IsSamePath method test failed: unequal paths trated like equal" << std::endl;
			return;
		}
		
		LOG(plog::info) << "all tests passed successfully";
	}
	catch (const std::exception &e)
	{
		LOG(plog::error) << e.what() << '\n';
	}
}
```
 При этом в тестах была жесткая привязка к реализации покрытия как массива байт.

После того, как  я расписал функциональность системы, в голову пришла мысль, что "покрытие"  в данном варианте по сути можно представить как следующую структуру:  $$ X(+,e,C)  $$
Здесь С - множество покрытий, e - "пустое покрытие", операция "+" - слияние покрытий между собой. Также между покрытиями можно ввести строгое отношение частичного порядка "<=", которое показывает, включает ли в себя одно покрытие в другое. По сути у нас здесь получается моноид по аналогии  с алгеброй множеств с операцией объединения и пустым множеством в качестве единицы. Но при этом мы намеренно не называем "покрытие" множеством, так как возможно в будущем захотим ввести некоторые другие свойства, которые не будут соответствовать множествам и будут сбивать нас с толку. 

После этого, код, реализующий покрытия принял такой вид, который более явно  отражает те свойства, которые мы хотели бы получить от покрытия: 
```C++
	
typedef std::vector<char> CoverageMap;

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
	
	CoverageMap & getCov()
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
	
	Coverage& operator=(const Coverage & r)
	{
		this->_coverage = r._coverage;
		return *this;
	}
			
	Coverage& operator=(const Coverage && r)
	{
		this->_coverage = r._coverage;
		return *this;
	}
	
	inline friend bool operator==(const Coverage & l, const Coverage & r)
	{
		return l._coverage == r._coverage;
	}

	inline friend bool operator!=(const Coverage & l, const Coverage & r)
	{
		return l._coverage != r._coverage;
	}
	
	CoverageMap & GetCov()
	{
		return _coverage;
	}
};
```

Также ниже привожу Unit-тесты, на основе которых был написан данный класс:
```C++
class CoverageTest:public ::testing::Test
{		
	public:
		CoverageTest()
		{	
		}
		~CoverageTest()
		{		
		}	
};

TEST_F(CoverageTest, IsEmptyTest)
{
	Coverage coverage;
	EXPECT_TRUE(coverage.IsEmpty());
}

TEST_F(CoverageTest, EqualityTest)
{	
	Coverage coverage1({1,2,3});
	Coverage coverage2({1,2,3});
	EXPECT_EQ(coverage1, coverage2);
	EXPECT_EQ(coverage2, coverage1);	
	EXPECT_FALSE(coverage1!=coverage2);
};

TEST_F(CoverageTest, NonEqualityTest)
{	
	Coverage coverage1({1,2,3,4});
	Coverage coverage2({1,2,3,1});
	EXPECT_TRUE(coverage1 != coverage2);	
	EXPECT_FALSE(coverage1 == coverage2);	
};

TEST_F(CoverageTest, NonEqualityTestDifferentSizes)
{	
	Coverage coverage1({1,2,3,4});
	Coverage coverage2({1,2,3});
	EXPECT_TRUE(coverage1 != coverage2);	
	EXPECT_FALSE(coverage1 == coverage2);	
};

TEST_F(CoverageTest, ContainsTestSameSizes)
{	
	Coverage coverage1({1,0,1,2,0,2});
	Coverage coverage2({1,0,1,1,0,1});
	EXPECT_TRUE(coverage1.Contains(coverage2));
	EXPECT_FALSE(coverage2.Contains(coverage1));	
};

TEST_F(CoverageTest, ContainsTestDifferentSizes)
{	
	Coverage coverage1({1,2,1,2,0,2});
	Coverage coverage2({1,0,1,2,0});
	EXPECT_FALSE(coverage1.Contains(coverage2));
	EXPECT_FALSE(coverage2.Contains(coverage1));	
};

TEST_F(CoverageTest, NotContainingOneAnotherTest)
{	
	Coverage coverage1({0,1,0,0,0});
	Coverage coverage2({0,0,0,1,0});
	EXPECT_FALSE(coverage1.Contains(coverage2));
	EXPECT_FALSE(coverage2.Contains(coverage1));	
};

TEST_F(CoverageTest, NotContainingOneAnotherBorders)
{	
	Coverage coverage1({1,0,0});
	Coverage coverage2({0,0,1});
	EXPECT_FALSE(coverage1.Contains(coverage2));
	EXPECT_FALSE(coverage2.Contains(coverage1));	
};

TEST_F(CoverageTest, ContainsEmtpyCoverageMap)
{		
	Coverage coverage1({}); 
	Coverage coverage2({0,1}); 
	EXPECT_FALSE(coverage1.Contains(coverage2));
	EXPECT_FALSE(coverage2.Contains(coverage1));
};

TEST_F(CoverageTest, TestAddition)
{	
	Coverage coverage1({1,0,0});
	Coverage coverage2({0,0,1});
	Coverage expected({1,0,1});
	Coverage actual(coverage1 + coverage2);
	EXPECT_EQ(expected, actual);
};

TEST_F(CoverageTest, TestAdditionContainigCoverages)
{	
	Coverage coverage1({1,0,1});
	Coverage coverage2({0,0,1});
	Coverage expected({1,0,1});
	Coverage actual(coverage1 + coverage2);
	EXPECT_EQ(expected, actual);
};

TEST_F(CoverageTest, TestAdditionEmptyCoverages)
{	
	Coverage coverage1({});
	Coverage coverage2({});
	Coverage expected({});
	Coverage actual(coverage1 + coverage2);
	EXPECT_EQ(expected, actual);
};

TEST_F(CoverageTest, TestAdditionDifferentSizes)
{	
	Coverage coverage1({1,0,0,0});
	Coverage coverage2({0,0,1});		
	EXPECT_THROW(coverage1 + coverage2, winafl_runtime_error);
};

TEST_F(CoverageTest, TestAdditionToEmptyCoverage)
{	
	Coverage coverage1({});
	Coverage coverage2({0,0,1});		
	EXPECT_THROW(coverage1 + coverage2, winafl_runtime_error);
};

TEST_F(CoverageTest, TestAssigningAddition)
{	
	Coverage coverage1({1,0,0});
	Coverage coverage2({0,0,1});
	Coverage expected({1,0,1});
	coverage1 += coverage2;
	EXPECT_EQ(expected, coverage1);
};

TEST_F(CoverageTest, TestAssigningAdditionForEmptyCoverages)
{	
	Coverage coverage1({});
	Coverage coverage2({});
	Coverage expected({});
	coverage1 += coverage2;
	EXPECT_EQ(expected, coverage1);
};

TEST_F(CoverageTest, TestAssigningAdditionDifferentSizes)
{	
	Coverage coverage1({1,0,0,0});
	Coverage coverage2({0,0,1});		
	EXPECT_THROW(coverage1 += coverage2, winafl_runtime_error);
};

TEST_F(CoverageTest, TestAssigningAdditionToEmptyCoverage)
{	
	Coverage coverage1({});
	Coverage coverage2({0,0,1});		
	EXPECT_THROW(coverage1 += coverage2, winafl_runtime_error);
};

TEST_F(CoverageTest, TestAssignment)
{	
	Coverage coverage1({});
	Coverage coverage2({1,2,3});
	coverage1 = coverage2;
	EXPECT_EQ(coverage1, coverage2);
};

TEST_F(CoverageTest, TestAssignmentOfSum)
{	
	Coverage coverage1({3,2,4});
	Coverage coverage2({1,3,3});
	Coverage result = coverage1 + coverage2;
	Coverage expected({3,3,4});
	EXPECT_EQ(result, expected);
};

```
Здесь, в тестах мы постарались  отойти от конкретного представления покрытия в виде массива байт, сосредоточившись на тех свойствах покрытия, которое оно должно иметь. До конца это сделать не удалось, все же покрытие инициализируется неким массивом чисел, которое дает представление о его внутреннем содержимом, но полагаю, что код инициализации  можно далее выделить отдельно.

Большим плюсом здесь стала возможность реализовывать другие виды покрытий, например те, которые учитывают два и более предыдущих базовых блока. 
## Выводы
Заставить себя отвлечься от кодирования и немного подумать над дизайном и попытаться хоть как-то описать его - непростая задача, поначалу кажется, что тратишь время просто так. Однако потом понимаешь, что это дает плоды, когда система начинает усложняться. В таком случае ты возвращаешься к обдуманной ранее схеме, быстро вспоминая, как она устроена, что должна делать и какими свойствами должны обладать её компоненты. 
