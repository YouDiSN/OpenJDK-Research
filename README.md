# 2017.10.7 更新

## Mac下编译OpenJDK9

编译JDK9步骤：

1. ```shell
   brew install hg
   ```

2. ```shell
   brew install freetype
   ```

3. ```shell
   hg clone http://hg.openjdk.java.net/jdk9/jdk9
   ```

4. ```shell
   sh ./get_source.sh
   ```

5. ```shell
   bash configure --with-debug-level=slowdebug \
   --with-freetype=/usr/local/opt/freetype \
   --disable-warnings-as-errors
   ```

6. ```shell
   cd jdk9
   ```

7. ```shell
   make
   ```

8. 如果在make的过程中代码报错，主要有以下3点错误，每一行diff代表一个

   ```shell
   diff -r b756e7a2ec33 src/share/vm/memory/virtualspace.cpp
   --- a/src/share/vm/memory/virtualspace.cpp      Thu Aug 03 18:56:57 2017 +0000
   +++ b/src/share/vm/memory/virtualspace.cpp      Sat Oct 07 21:27:59 2017 +0800
   @@ -581,7 +581,8 @@
      assert(markOopDesc::encode_pointer_as_mark(&_base[size])->decode_pointer() == &_base[size],
             "area must be distinguishable from marks for mark-sweep");
   -  if (base() > 0) {
   +  // if (base() > 0) {
   +  if (base() != NULL) {
        MemTracker::record_virtual_memory_type((address)base(), mtJavaHeap);
      }
    }
    

   diff -r b756e7a2ec33 src/share/vm/opto/lcm.cpp
   --- a/src/share/vm/opto/lcm.cpp Thu Aug 03 18:56:57 2017 +0000
   +++ b/src/share/vm/opto/lcm.cpp Sat Oct 07 21:27:59 2017 +0800
   @@ -39,7 +39,8 @@
    // Check whether val is not-null-decoded compressed oop,
    // i.e. will grab into the base of the heap if it represents NULL.
    static bool accesses_heap_base_zone(Node *val) {
   -  if (Universe::narrow_oop_base() > 0) { // Implies UseCompressedOops.
   +  // if (Universe::narrow_oop_base() > 0) { // Implies UseCompressedOops.
   +  if (Universe::narrow_oop_base() != NULL) { // Implies UseCompressedOops.
        if (val && val->is_Mach()) {
          if (val->as_Mach()->ideal_Opcode() == Op_DecodeN) {
            // This assumes all Decodes with TypePtr::NotNull are matched to nodes that


   diff -r b756e7a2ec33 src/share/vm/opto/loopPredicate.cpp
   --- a/src/share/vm/opto/loopPredicate.cpp       Thu Aug 03 18:56:57 2017 +0000
   +++ b/src/share/vm/opto/loopPredicate.cpp       Sat Oct 07 21:27:59 2017 +0800
   @@ -270,7 +270,8 @@
      Node* ctrl  = iff->in(0);
      // Match original condition since predicate's projections could be swapped.
   -  assert(predicate_proj->in(0)->in(1)->in(1)->Opcode()==Op_Opaque1, "must be");
   +  // assert(predicate_proj->in(0)->in(1)->in(1)->Opcode()==Op_Opaque1, "must be");
   +  // assert(rng->Opcode() == Op_LoadRange || _igvn.type(rng)->is_int()->_hi >= 0, "must be");
      Node* opq = new Opaque1Node(igvn->C, predicate_proj->in(0)->in(1)->in(1)->in(1));
      igvn->C->add_predicate_opaq(opq);
   @@ -900,7 +901,7 @@
          Node*          idx    = cmp->in(1);
          assert(!invar.is_invariant(idx), "index is variant");
          Node* rng = cmp->in(2);
   -      assert(rng->Opcode() == Op_LoadRange || _igvn.type(rng)->is_int() >= 0, "must be");
   +      // assert(rng->Opcode() == Op_LoadRange || _igvn.type(rng)->is_int() >= 0, "must be");
        assert(invar.is_invariant(rng), "range must be invariant");
        int scale    = 1;
        Node* offset = zero;
   ```

9. 打开clion，只要hotspot文件夹，右上角的configuration使用进入build出来的目录下的jdk9/build/macosx-x86_64-normal-server-slowdebug/jdk/bin/java，取消在debug之前的build，在ini.cpp第3884行打断点，忽略clion的各种红色报错，就可以顺利debug虚拟机

编译好的jdk就在项目目录中，可以在build文件夹下找到





## 项目用途

OpenJDK(JVM、Javac)源代码学习研究(包括代码注释、文档、用于代码分析的测试用例) 


## HotSpot、Javac版本

这个项目的源代码是从[openjdk8-b132](http://hg.openjdk.java.net/jdk8/jdk8/tags)
里抽取hotspot、javac目录得来的。


## HotSpot构建与调试

[在Windows平台构建与调试HotSpot](https://github.com/codefollower/OpenJDK-Research/blob/master/hotspot/my-docs/%E5%9C%A8Windows%E5%B9%B3%E5%8F%B0%E6%9E%84%E5%BB%BA%E4%B8%8E%E8%B0%83%E8%AF%95HotSpot.md)


## Javac构建与调试

[构建与调试](https://github.com/codefollower/OpenJDK-Research/blob/master/javac/my-docs/%E6%9E%84%E5%BB%BA%E4%B8%8E%E8%B0%83%E8%AF%95.md)
