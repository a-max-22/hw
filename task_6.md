### Итерация 1.
### Код
Первую итерацию решил сделать на примере структуры данных "кольцевой буфер". 
Код к нему следующий: 
```python
class CircularBuffer:
    def __init__(self, size):
        self.buffer = [None] * size
        self.head = 0
        self.tail = 0
        self.size = size

    def _is_empty(self, cell_index):
        return self.buffer[cell_index] is None 

    def _free_cell(self, cell_index):
        self.buffer[cell_index] = None 

    def get(self):
        if self._is_empty(self.head):
            return None
        value = self.buffer[self.head]
        self._free_cell(self.head)
        self.head = (self.head + 1) % self.size
        return value

    def put(self, value):
        next_tail = (self.tail + 1) % self.size
        if not self._is_empty(self.tail):
            return False
        self.buffer[self.tail] = value
        self.tail = next_tail
        return True
```

Тесты представлены ниже: 
```python
import unittest
from  CircularBuffer import CircularBuffer

class TestCircularBuffer(unittest.TestCase):
    def setUp(self):
        self.buffer = CircularBuffer(3)

    def test_single_elem(self):
        self.assertTrue(self.buffer.put(1))
        self.assertEqual(self.buffer.get(), 1)

    def test_multiple_elems(self):
        self.assertTrue(self.buffer.put(1))
        self.assertTrue(self.buffer.put(2))
        self.assertTrue(self.buffer.put(3))

        self.assertEqual(self.buffer.get(), 1)
        self.assertEqual(self.buffer.get(), 2)
        self.assertEqual(self.buffer.get(), 3)

    def test_get_empty(self):
        self.assertIsNone(self.buffer.get())

    def test_put_full(self):
        self.assertTrue(self.buffer.put(1))
        self.assertTrue(self.buffer.put(2))
        self.assertTrue(self.buffer.put(3))
        self.assertFalse(self.buffer.put(4))

    def test_full_buf(self):
        self.assertTrue(self.buffer.put(1))
        self.assertTrue(self.buffer.put(2))
        self.assertTrue(self.buffer.put(3))

        self.assertFalse(self.buffer.put(4)) 
        self.assertEqual(self.buffer.get(), 1)
        self.assertEqual(self.buffer.get(), 2)
        self.assertTrue(self.buffer.put(4)) 
        self.assertEqual(self.buffer.get(), 3)
        self.assertEqual(self.buffer.get(), 4)

if __name__ == '__main__':
    unittest.main()
```

### Рассуждения
В данном случае я писал код большими кусками - вначале реализовал вырожденный случай, когда буфер имеет размер в 1 элемент и написал на него тесты. Далее сделал уже тесты для полного случая и дописал соответствующий функционал.
По сути получилось 2 коммита. Но по сути можно разбить их на более мелкие части с коммитом , как минимум по такой схеме: 
1) тест на получение из пустого буфера
2) тест на вставку элемента
3) тест на вставку/извлечение элемента
4) тест на вставку в полный буфер
5) тест на заполнение буфера/извлечение одного элемента/повторную вставку
Таким образом, получилось как минимум 5 микрошагов, которыми можно двигаться, реализуя данную логику. 

### Итерация 2.
В рамках данной итерации я писал код, который проходит по графу базовых блоков. Под базовым блоком понимается  последовательность инструкций, которая не содержит внутри себя условий, ветвлений и переходов.  Базовый блок  характеризуется своим начальным адресом  и конечным адресом. Базовые блоки формируют между собой связи, которые обозначают, что между ними возможна передача управления. Таким образом, они образуют ориентированный граф. 

Задача состоит в том, что по набору инструкций необходимо сформировать такой вышеописанный граф передачи управления. 

Ниже привел упрощенную версию кода, который это реализует: 
```python 
class Instruction:
    def __init__(self, op, args, length ):
        self.operation = op
        self.args = args
        self.length = length

    @staticmethod
    def is_control_flow(instruction):
        return instruction.operation == 'jump'
    
    @staticmethod
    def get_args(instruction):
        return instruction.args

class Block:
    def __init__(self):
        pass

class BasicBlock:
    def __init__(self, start = None, end = None):
        self.start = start
        self.end = end
        self.links = []

class EmptyBlock(Block):
    pass 

def handle_regular_instruction(block, addr):
    if block is None:
        block = BasicBlock(addr, addr)
    block.end = addr
    return block

def handle_instructions(block, addr, instr):
    if not instr.is_control_flow(instr):
        return handle_regular_instruction(block, addr)
    
    destination = instr.args[0]
    dest_block = BasicBlock(destination, destination)
    block.links.append(dest_block)
    return dest_block

def build_blocks(instructions_trace):
    if len(instructions_trace) == 0:        
        return EmptyBlock()
    
    blocks = None
    curr_block = blocks
    
    for (addr, instr) in instructions_trace:         
        curr_block = handle_instructions(curr_block, addr, instr)
        blocks = curr_block if blocks is None else blocks

    return blocks
```

Тесты к нему представлены ниже: 
```python
import unittest
from  BasicBlockBuilder import *

class TestBasicBlockBuilder(unittest.TestCase):
    def setUp(self):
        self.instr_len = 4
        self.start_addr = 0
        self.current_addr = self.start_addr - self.instr_len

        self.trace = []
        self.args = [1, 2]
    
    def appendJump(self, destination):
        self.current_addr += self.instr_len
        addr = self.current_addr 
        self.trace.append( (addr,  Instruction('jmp', [destination], self.instr_len)) )

    def appendInstr(self, args):
        self.current_addr += self.instr_len
        addr = self.current_addr 
        self.trace.append( (addr,  Instruction('instr', args, self.instr_len)) )
    
    def test_empty_trace(self):
        basic_blocks = build_blocks(self.trace)
        self.assertIsInstance(basic_blocks, EmptyBlock)

    def test_single_instruction_trace(self):
        self.appendInstr(self.args)

        basic_blocks = build_blocks(self.trace)
        self.assertIsInstance(basic_blocks, BasicBlock)
        self.assertEqual(basic_blocks.start, self.start_addr)
        self.assertEqual(basic_blocks.end, self.start_addr)


    def test_multiple_instructions_in_single_bb(self):
        self.appendInstr(self.args)        
        self.appendInstr(self.args)        

        basic_blocks = build_blocks(self.trace)
        self.assertIsInstance(basic_blocks, BasicBlock)
        self.assertEqual(basic_blocks.start, self.start_addr)
        self.assertEqual(basic_blocks.end, self.current_addr)

    def test_jump(self):
        self.appendInstr(self.args)        
        self.appendInstr(self.args)
        
        new_bb_start = self.current_addr + self.instr_len * 2
        new_bb_end = new_bb_start
        
        self.appendJump(new_bb_start)        
        self.appendInstr(self.args)        

        basic_blocks = build_blocks(self.trace)
        self.assertIsInstance(basic_blocks, BasicBlock)
        self.assertEqual(len(basic_blocks.links), 1)
        self.assertEqual(len(basic_blocks.start), self.start_addr)
        self.assertEqual(len(basic_blocks.end), self.start_addr + self.instr_len)
        
        next_block = basic_blocks.links[0]
        self.assertIsInstance(next_block, BasicBlock)
        self.assertEqual(next_block.start,  new_bb_start)
        self.assertEqual(next_block.end,  new_bb_end)


if __name__ == '__main__':
    unittest.main()
```


### Рассуждения
Здесь я писал код согласно рекомендациям и в итоге, несмотря на то, что здесь представлено только 4 теста, итоговых коммитов получилось 13 - в средем по 3 на тест. Это связано с тем, что я коммитил код после написания почти  каждого ассерта (в случае,  если он был неуспешный, а затем стал успешный). В итоге получается достаточно быстро продвигаться, да и психологически легче писать - каждый раз, когда все тесты срабатывают испытываешь чувство удовлетворения и двигаясь по такому циклу входишь в  своеобразный поток.  Да и проводить некоторый рефакторинг, убирая излишнюю вложенность в условиях как-то проще. Несмотря на некоторую кажущуюся избытычность таких микрошагов реализовать код получилось достаточно быстро, так как меньше времени уходит на то, чтобы понять, что же ты сделал не так - изменения минимальные и место, где они сделаны максимально изолированно.  В реальном рабочем процессе данный подход не применял, взял на вооружение. 
