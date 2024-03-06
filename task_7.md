Для примера возьмем следующий код: 
```python
...

class Module:
    def __init__(self, base, size):
        pass

    def getSize(self):
        return self.size

    def getBase(self):
        return self.base


class ElfModule(Module):
    def __init__(self, base, size):
        self.base = base
        self.size = size
        view = IdaView(module_start, module_size)
        self.elf = ELF.ELF(view, inmem=True)
    
    def getSymbols():
        return self.elf.getSymbols()
    
    
class PeModule(Module):
    def __init__(self, base, size):
        self.base = base
        self.size = size
        view = IdaView(module_start, module_size)
        self.pe = PE.PE(view, inmem=True)
    
    def getImports(self):
        return self.pe.getImports()
    
    def getExports(self):
        return self.pe.getExports()


class ModuleAction:
    def __init__(self):
        pass
    def DoAction(self):
        pass

class PrintPeImports(ModuleAction):
    def __init__(self):
        pass
        
    def DoAction(self, module):
        imports = module.getImports()
        for offset, index, name in imports:
            print("import", hex(module.getBase()), hex(offset), name)


class PrintPeExports(ModuleAction):
    def __init__(self):
        pass
        
    def DoAction(self, module:PeModule):
        exports = module.getExports()
        for offset, index, name in exports:
            print("export", hex(module.getBase()), hex(offset), name)

class PrintElfSymbols(ModuleAction):
    def __init__(self):
        pass
        
    def DoAction(self, module:ElfModule):
        symbols = module.getSymbols()
        for symbol in symbols:
            print("symbols: ", symbol.getName(), symbol)


class RenamePeExports(ModuleAction):
    def __init__(self):
        pass
        
    def DoAction(self, module:PeModule):
        imports = module.getExports()
        for offset, index, name in exports:
            export_addr = module_start + offset
            self.RenameSingleExport(export_addr, name)
    
    def RenameSingleExport(self, export_addr, name):
        ...

```

Здесь мы видим две иерархии классов - одна представляет собой производную от класса `Module` и соответствует некоторому модулю, загруженному в память процесса. Вторая иерархия классов - производных от `ModuleAction` представляет собой некоторые действия, которые можно выполнять над модулями определенного типа - например вывести сведения об импортах/экспортах, вывести имена символов, переименовать их и.т.д. 

В классах, производных от `ModuleAction` мы как раз наблюдем пример 'неистинного наследования', когда переопределённый метод DoAction никогда не вызывает метода родительского класса. Поэтому данная иерархия является отличным кандидатом на преобразование  с использованием паттерна `Visitor`. 

Преобразованный код выглядит следующим образом (приведены только изменения, часть общего с прошлым примером кода пропущена): 
```python

class Module:
	...	
	# добавлен новый метод
    def accept(self, action):
        pass
	...

class ElfModule(Module):
	...    
    def accept(self, action):
        action.DoElfModuleAction(self)
	...
    
class PeModule(Module):
	...
    def accept(self, action):
        action.DoPeModuleAction(self)
	...


class ModulesActionVisitor:
    def __init__(self):
        pass
        
    def DoPeModuleAction(self, module:PeModule):
        pass

    def DoElfModuleAction(self, module:ElfModule):
        pass    


class PrintExportsVisitor(ModulesActionVisitor):
    def DoPeModuleAction(self, module:PeModule):
        exports = module.getExports()
        for offset, index, name in exports:
            print("export", hex(module.getBase()), hex(offset), name)


class PrintImportsVisitor(ModulesActionVisitor):
    def DoPeModuleAction(self, module:PeModule):
        exports = module.getImports()
        for offset, index, name in exports:
            print("import", hex(module.getBase()), hex(offset), name)
            

class PrintSymbolsVisitor(ModulesActionVisitor):
    def DoElfModuleAction(self, module:PeModule):
        symbols = module.getSymbols()
        for symbol in symbols:
            print("symbols: ", symbol.getName(), symbol)
    

class RenameExportsVisitor(ModulesActionVisitor):
    def DoPeModuleAction(self, module:PeModule):
        imports = module.getExports()
        for offset, index, name in exports:
            export_addr = module_start + offset
            self.RenameSingleExport(export_addr, name)

    def RenameSingleExport(self, export_addr, name):
		...

...
printImports = PrintImportsVisitor()
for module in modules:
	module.accept(printImports)
```

В данном случае мы дополнили интерфейс класса Module новым методом `accept`, который в качестве параметра принимает экземпляр класса `ModulesActionVisitor` - то действие, которое необходимо осуществить над модуляем. Все соответствующие действия, такие как печать импортов/экспортов/символов, их переименование и.т.д. упакованы в производные от данного класса. 

Что это нам дало в итоге? Самое главное заключается в том, что  этот подход  позволил единообразно  применять те или иные действия по отношению к совокупности модулей. Например, теперь перед тем, как применить действие "RenameExports" нам не надо проверять то, что аргумент имеет тип `PeModule`, это происходит автоматически. С другой стороны при добавлении нового типа модулей надо менять интерфейс класса `ModulesActionVisitor`, добавляя в него новый метод, для работы с новым типом, однако место, в котором эти изменения происходят ограничено, мы можем добавить пустой метод и остальные подклассы буду сохранять работоспособность. В целом видится так, что это приемлемая цена за возможность единообразного применения тех или иных действий для множества модулей. 
