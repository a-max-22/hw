## Снижение цикломатической сложности

### Пример 1. 

Изначальный код (cyclomatic complexity = 12).

```python

def dis_callback(mdis, cur_block, offsets_to_dis):
    if cur_block is None:
        return 
        
    try:
        last_instr = cur_block.lines[-1]
    except Exception as e:
        log(__name__, e)
        return 
    
    if not last_instr.is_subcall():
        return
    
    loc_db = mdis.loc_db
    last_instr = cur_block.lines[-1]
    dst_key = last_instr.getdstflow(loc_db)[0]
    
    if not isinstance(dst_key, ExprLoc):
        return
    
    call_dst = loc_db.get_location_offset(dst_key.loc_key)
    if Context.text_seg_start <= call_dst <= Context.text_seg_end:
        next_call_dst = get_call_dst(mdis, call_dst)
        
        if next_call_dst is None or (text_seg_start <= next_call_dst <= text_seg_end):
            log(__name__, hex(call_dst), hex(next_call_dst) if next_call_dst is not None else None, "within seg")
            Context.global_referenced_funcs.append(call_dst)            
            if call_dst <= Context.processed_address:
                log("dis_callback: adding call_dst %x for disasm:" % call_dst)
                cur_block.add_cst(dst_key.loc_key, AsmConstraint.c_to)
                offsets_to_dis.add(call_dst)
            return
        else:
            log(hex(call_dst), hex(next_call_dst), "not within seg")

    actual_call_dst, actual_ret_addr = get_actual_call_dst(last_instr.offset)
    if (actual_call_dst, actual_ret_addr) == (None, None):
        log(__name__, "cannot get destination for instruction %x" % last_instr.offset)
        return
    
    if actual_call_dst is not None:
        actual_call_dst_loc = loc_db.get_or_create_offset_location(actual_call_dst)
        last_instr.args = [ExprLoc(actual_call_dst_loc, 64)]

    if actual_ret_addr is None:
        return  

    current_ret_addr = loc_db.get_location_offset(cur_block.get_next())
    remove_block_next_constraint(cur_block)
    actual_ret_addr_loc = loc_db.get_or_create_offset_location(actual_ret_addr)
    cur_block.add_cst(actual_ret_addr_loc, AsmConstraint.c_next)

    offsets_to_dis.remove(current_ret_addr)
    offsets_to_dis.add(actual_ret_addr)
    log(__name__, "[note]: replaced %x by %x" %(current_ret_addr, actual_ret_addr))    

```

dis_callback может работать по следующим сценариям: 
1) нам не нужно искать адрес возврата -  default-ный сценарий. Например, такое происходит, когда функция находится в другом сегменте 
2) нам нужно искать адрес возврата  - тогда здесь возникает много сценариев, в том числе подмена адреса, с которого 

В ходе рефакторинга применял следующие приемы: 
1) объединение условий в одно 
2) вынос проверок в отдельные функции - `within_text_segment`, `need_to_get_actual_dst`, `enqueue_dst_for_further_processing_if_needed`.

Переработанный код (cyclomatic complexity = 6).: 
```python

def within_text_segment(addr):
	return (not addr is None) and (Context.text_seg_start <= addr <= Context.text_seg_end)
	
def need_to_get_actual_dst(call_dst, next_call_dst):
	return not within_text_segment(call_dst) or not within_text_segment(call_dst) 

def need_to_enqueue_dst_for_further_processing(call_dst):
	return (call_dst <= Context.processed_address)

def enqueue_dst_for_further_processing_if_needed(cur_block, offsets_to_dis, call_dst):
    if not need_to_enqueue_dst_for_further_processing(call_dst):
        return 
    cur_block.add_cst(dst_key.loc_key, AsmConstraint.c_to)
    offsets_to_dis.add(call_dst)
    log(__name__, "adding call_dst %x for processing:" % call_dst)


def dis_callback(mdis, cur_block, offsets_to_dis):
    try:
        last_instr = cur_block.lines[-1]
        loc_db = mdis.loc_db
        last_instr = cur_block.lines[-1]
        dst_key = last_instr.getdstflow(loc_db)[0]
    except Exception as e:
        log(__name__, e)
        return 
    
    if not last_instr.is_subcall() or not isinstance(dst_key, ExprLoc):
        return
        
    call_dst = loc_db.get_location_offset(dst_key.loc_key)
    next_call_dst = get_call_dst(mdis, call_dst)
    need_to_get_actual_destination = need_to_get_actual_dst(call_dst, next_call_dst)
    
    if not need_to_get_actual_destination:
        enqueue_dst_for_further_processing_if_needed(cur_block, offsets_to_dis, call_dst)
        Context.referenced_funcs.append(call_dst)
        return
    
    actual_call_dst, actual_ret_addr = get_actual_call_dst(last_instr.offset)    
    if actual_ret_addr is None:
        return  
    
    actual_call_dst_loc = loc_db.get_or_create_offset_location(actual_call_dst)
    last_instr.args = [ExprLoc(actual_call_dst_loc, 64)]

    current_ret_addr = loc_db.get_location_offset(cur_block.get_next())
    remove_block_next_constraint(cur_block)
    actual_ret_addr_loc = loc_db.get_or_create_offset_location(actual_ret_addr)
    cur_block.add_cst(actual_ret_addr_loc, AsmConstraint.c_next)

    offsets_to_dis.remove(current_ret_addr)
    offsets_to_dis.add(actual_ret_addr)
    log(__name__, "[note]: replaced %x by %x" %(current_ret_addr, actual_ret_addr))    

```


### Пример 2.

Код до рефакторинга (cyclomatic complexity = 5): 

```python
def rename_exports(module_start, module_size):
    view = IdaView(module_start, module_size)
    pe = PE.PE(view, inmem=True)
    exports = pe.getExports()
    
    for offset, index, name in exports:
        export_addr = module_start + offset
        flags = idaapi.get_flags(export_addr)
        
        if idaapi.is_code(flags):
            existing_name = idaapi.get_name(export_addr)
            if existing_name == name:
                log("function is already renamed", existing_name)
            continue

        ida_funcs.add_func(export_addr)
        rename_status = idaapi.set_name(export_addr, name)
        if rename_status == 1:
            log("successfully renamed export item", hex(export_addr), name)
        else:
            log("error renaming export item", hex(export_addr), name)
```


Код после  рефакторинга (cyclomatic complexity = 2).

```python
def rename_func_if_need_to(export_addr, name):
    flags = idaapi.get_flags(export_addr)
    existing_name = idaapi.get_name(export_addr)
    
    if idaapi.is_code(flags) and (existing_name == name):
        log("function is already renamed", existing_name)
        return 

    ida_funcs.add_func(export_addr)
    rename_status = idaapi.set_name(export_addr, name)
    
    log("successfully renamed exported item" if rename_status == 1 \
	    else "error renaming exported item", hex(export_addr), name)
      

def rename_exports(module_start, module_size):
    view = IdaView(module_start, module_size)
    pe = PE.PE(view, inmem=True)
    exports = pe.getExports()
    
    for offset, index, name in exports:
        export_addr = module_start + offset
        rename_func_if_need_to(export_addr, name)
```

В ходе рефакторинга также по аналогии применял следующие приемы: 
1) объединение условий в одно 
2) вынос проверок в отдельные функции
3) замена конструкции else выражением `<expr1> if <condition> else <expr2>`. 

### Пример 3. 

Код до рефакторинга (cyclomatic complexity = 7):
```python

def run_symexec(symExecEngine, mdis, lifter, ircfg, initialAddress, lbl_stop=None, step=False, maxIterations = 10000):
    counter = 0
    nextDst = initialAddress
    prevSymbols = symExecEngine.symbols.copy()
    prevIrBlock = None
    prevDst = None
    
    for counter in range(threshold):
        if prevDst == nextDst:
            print("Destination doesn't change, breaking")
            break
            
        counter += 1
        print(counter, "nextDst, prevDst:" , nextDst, prevDst)
        irblock = ircfg.get_block(nextDst)
        if irblock is not None:
            prevIrBlock = irblock
            prevSymbols = symExecEngine.symbols.copy()
            prevDst = nextDst
            nextDst = symExecEngine.eval_updt_irblock(irblock, step=step)
            continue
            
        nextBlock = getNextBlockByPc(mdis, nextDst, symExecEngine)
        if nextBlock is not None:
            lifter.add_asmblock_to_ircfg(nextBlock, ircfg)
            continue
            
        prevDst = nextDst
        nextDst = symExecEngine.eval_expr(nextDst)
        if isinstance(nextDst, ExprCond):
            nextDst = tryToResolveConditionalExpression(nextDst)
        
        if not isinstance(nextDst, ExprInt):
            print("Can't determine next dst for", nextDst)
            break            

```

Код после рефакторинга (cyclomatic complexity = 3):

```python 
def add_block_to_cfg(nextBlock):
    if nextBlock is not None:
        lifter.add_asmblock_to_ircfg(nextBlock, ircfg)

def get_next_destination_update_state(symExecEngine, mdis, nextDst):
    irblock = ircfg.get_block(nextDst)
    if irblock is None:
        return None
    
    nextDst = symExecEngine.eval_updt_irblock(irblock, step=step)
    nextBlock = getNextBlockByPc(mdis, nextDst, symExecEngine)
    
    add_block_to_cfg(nextBlock)
    nextDst = symExecEngine.eval_expr(nextDst)
    if isinstance(nextDst, ExprCond):
        nextDst = tryToResolveConditionalExpression(nextDst)
    
    return nextDst


def run_symexec(symExecEngine, mdis, ircfg, initialAddress, maxIterations = 10000):
    counter = 0
    nextDst = initialAddress
    prevDst = None
    
    for counter in range(maxIterations):
        if (prevDst == nextDst) or (nextDst is None) or (not isinstance(nextDst, ExprInt)):
            log("Destination doesn't change, breaking execution")
            break
        log(counter, "nextDst, prevDst:" , nextDst, prevDst)
        counter += 1

        prevDst = nextDst
        nextDst = get_next_destination_update_state(symExecEngine, mdis, nextDst)    
```

В ходе рефакторинга также по аналогии применял следующие приемы: 
1) объединение условий в одно 
2) вынос проверок в отдельные функции
3) замена всех условий в цикле на одно.
