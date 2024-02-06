# Common commands in compilation process
generate unoptimized IR *.bc
```
clang {cflags} -emit-llvm -c -O3 -Xclang -disable-llvm-optzns -o {IR}
```
from *.bc to human readable *.ll file
```
llvm-dis {*.bc} -o {*.ll}
```
use opt to optimize IR
```
opt {seq} {IR} -o {IR_opt}
```
from optimized IR to obj *.o (use -filetype=asm to generate asm *.s)
```
llc -O3 -filetype=obj {IR_opt} -o {obj}
```
from obj to exe
```
clang {obj} -o {exe}
```

# Common commands in generating passes
LLVM 17 generate O3 pipeline
```shell
llvm-as < /dev/null | opt -O3 -disable-output --print-pipeline-passes
```
