# Youpk: Another ART-based active call huller
## Open sourced
Happy Children's Day to all
Warehouse address: https://github.com/youlor/unpacker, welcome to the big guy star.

## Principle

Youpk is a shelling machine for Dex overall reinforcement + various Dex extraction

The basic process is as follows:

1. Dump DEX from memory
2. Construct the complete call chain, actively call all methods and dump CodeItem
3. Merge DEX, CodeItem

### Dump DEX from memory

DEX files are represented in the art virtual machine using DexFile objects, and these objects are referenced in ClassLinker, so they can be obtained by traversing DexFile objects from ClassLinker and dumping.

```c++
//unpacker.cc
std::list<const DexFile*> Unpacker::getDexFiles() {
  std::list<const DexFile*> dex_files;
  Thread* const self = Thread::Current();
  ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
  ReaderMutexLock mu(self, *class_linker->DexLock());
  const std::list<ClassLinker::DexCacheData>& dex_caches = class_linker->GetDexCachesData();
  for (auto it = dex_caches.begin(); it != dex_caches.end(); ++it) {
    ClassLinker::DexCacheData data = *it;
    const DexFile* dex_file = data.dex_file;
    dex_files.push_back(dex_file);
  }
  return dex_files;
}
```

In addition, to avoid any form of optimization affecting the dumped dex file, set CompilerFilter to verify only in dex2oat

```c++
//dex2oat.cc
compiler_options_->SetCompilerFilter(CompilerFilter::kVerifyAtRuntime);
```



### Construct the complete call chain, actively calling all methods

1. Create a shelling thread

   ```java
   //unpacker.java
   public static void unpack() {
       if (Unpacker.unpackerThread != null) {
           return;
       }
   
       //Turn on thread calls
       Unpacker.unpackerThread = new Thread() {
           @Override public void run() {
               while (true) {
                   try {
                       Thread.sleep(UNPACK_INTERVAL);
                   }
                   catch (InterruptedException e) {
                       e.printStackTrace();
                   }
                   if (shouldUnpack()) {
                       Unpacker.unpackNative();
                   }   
               }
           }
       };
       Unpacker.unpackerThread.start();
   }
   ```

2. Iterate through all of DexFile's ClassDefs in a shelling thread

   ```c++
   //unpacker.cc
   for (; class_idx < dex_file->NumClassDefs(); class_idx++) {
   ```

3. Parse and initialize the Class

   ```c++
   //unpacker.cc
   mirror::Class* klass = class_linker->ResolveType(*dex_file, dex_file->GetClassDef(class_idx).class_idx_, h_dex_cache, h_class_loader);
   StackHandleScope<1> hs2(self);
   Handle<mirror::Class> h_class(hs2.NewHandle(klass));
   bool suc = class_linker->EnsureInitialized(self, h_class, true, true);
   ```

4. Actively call all methods of the class, and modify ArtMethod::Invoke to force the switch interpreter

   ```c++
   //art_method.cc
   if (UNLIKELY(!runtime->IsStarted() || Dbg::IsForcedInterpreterNeededForCalling(self, this) 
   || (Unpacker::isFakeInvoke(self, this) && !this->IsNative()))) {
   if (IsStatic()) {
   art::interpreter::EnterInterpreterFromInvoke(
   self, this, nullptr, args, result, /*stay_in_interpreter*/ true);
   } else {
   mirror::Object* receiver =
   reinterpret_cast<StackReference<mirror::Object>*>(&args[0])->AsMirrorPtr();
   art::interpreter::EnterInterpreterFromInvoke(
   self, this, receiver, args + 1, result, /*stay_in_interpreter*/ true);
   }
   }
   
   //interpreter.cc
   static constexpr InterpreterImplKind kInterpreterImplKind = kSwitchImplKind;
   ```

5. Instrumentation in the interpreter and setting callbacks before each instruction is executed

   ```c++
   //interpreter_switch_impl.cc
   // Code to run before each dex instruction.
     #define PREAMBLE()                                                                 \
     do {                                                                               \
       inst_count++;                                                                    \
       bool dumped = Unpacker::beforeInstructionExecute(self, shadow_frame.GetMethod(), \
                                                        dex_pc, inst_count);            \
       if (dumped) {                                                                    \
         return JValue();                                                               \
       }                                                                                \
       if (UNLIKELY(instrumentation->HasDexPcListeners())) {                            \
         instrumentation->DexPcMovedEvent(self, shadow_frame.GetThisObject(code_item->ins_size_),  shadow_frame.GetMethod(), dex_pc);            						   										   \
       }                                                                                \
     } while (false)
   ```

6. Do targeted CodeItem dump in the callback, here is just a simple example of direct dump, in fact, for some manufacturers' extraction, you can really execute a few instructions to wait for CodeItem decryption before dumping
   ```c++
   //unpacker.cc
   bool Unpacker::beforeInstructionExecute(Thread *self, ArtMethod *method, uint32_t dex_pc, int inst_count) {
     if (Unpacker::isFakeInvoke(self, method)) {
     	Unpacker::dumpMethod(method);
       return true;
     }
     return false;
   }
   ```



### Merge DEX, CodeItem

Fill the dumped CodeItem in the corresponding position of the DEX. Mainly based on Google DX tool modifications.

### Reference link

FUPK3: https://bbs.pediy.com/thread-246117.htm

FART: https://bbs.pediy.com/thread-252630.htm

## Flash

1. Only support Pixel 1 generation
2. Reboot to bootloader: 'adb reboot bootloader'
3. Unzip Youpk_sailfish.zip and double-click 'flash-all.bat' 

Baidu Cloud: https://pan.baidu.com/s/1nUC5PpYGEBvkuvV-82pluA 
Extraction code: SC82 



## Usage

1. **This tool is only for learning communication, please do not use it for illegal purposes, otherwise you will pay the consequences! **
   
2. Configure the name of the app package to be shelled, which is the process name to be precise

    ```bash
    adb shell "echo cn.youlor.mydemo >> /data/local/tmp/unpacker.config"
    ```

3. Launch the apk and wait for the shelling
    Shelling is automatically re-shelled every 10 seconds (dex that has been completely dumped will be ignored), and the shelling is complete when the log prints unpack end

4. Pull out the dump file, the dump file path is '/data/data/package name/unpacker' 

    ```bash
    adb pull /data/data/cn.youlor.mydemo/unpacker
    ```

5. Call the repair tool dexfixer.jar, two parameters, the first is the dump file directory (must be a valid path), the second is the reorganized DEX directory (will be created if it does not exist)
    ```bash
    java -jar dexfixer.jar /path/to/unpacker /path/to/output
    ```




## Frequently Asked Questions

1. Dump quit or freeze halfway, restart the process, and wait for the shelling again
2. Currently, only shell-protected dex is supported, and dex/jar dynamically loaded by the App is not supported
