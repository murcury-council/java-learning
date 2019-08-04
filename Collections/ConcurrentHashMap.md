# ConcurrentHashMap

####addCount()

CAS操作成功则直接修改baseCount

CounterCell[] 记录CAS操作失败的线程的修改

所以在计算size()时是通过baseCount加所有的CounterCell的值

但是在累加的过程中也存在着并发的插入和删除，所有`size()`以及`mappingCount`都是一个预估值



f