# Art 虚拟机代码加载简析

# Art 虚拟机代码加载简析

## 前言

最近在学加固，做了点笔记，前面分享过一篇[Dalvik 虚拟机代码加载简析](https://bbs.pediy.com/thread-266028.htm)

 

这篇是Art版，基于**android-5.0.0_r2**

 

懒得画图了，art其他版本后续有空再更吧（逃

 

java层和android-4.4.4_r1好像区别不大

 

android5的mCookie 类型由int 变成了long

 

android4 native层和java层（好像？）是通过brige连接

 

android5 开始才是JNI标准

## DexClassLoader

```
xref: libcore/dalvik/src/main/java/dalvik/system/DexClassLoader.java
 
extends BaseDexClassLoader
```

### 构造函数

```
dexPath：dex文件源码目录
optimizedDirectory：优化以后odex生成目录
librarySearchPath：需要的本地so库的搜索路径
parent：父加载器
 
public DexClassLoader(String dexPath, String optimizedDirectory,
        String libraryPath, ClassLoader parent) {
    super(dexPath, new File(optimizedDirectory), libraryPath, parent);
}
```

## BaseDexClassLoader

#### pathList

```
xref: libcore/dalvik/src/main/java/dalvik/system/BaseDexClassLoader.java
 
extends ClassLoader
private final DexPathList pathList;
```

### 构造函数

```
public BaseDexClassLoader(String dexPath, File optimizedDirectory,
        String libraryPath, ClassLoader parent) {
    super(parent);
    this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
}
```

## DexPathList

#### dexElements

```
xref: /libcore/dalvik/src/main/java/dalvik/system/DexPathList.java
 
private final Element[] dexElements;
```

### 构造函数

```
public DexPathList(ClassLoader definingContext, String dexPath,
           String libraryPath, File optimizedDirectory) {
    ...
    this.dexElements = makeDexElements(splitDexPath(dexPath),
        optimizedDirectory,suppressedExceptions);
}
```

### makeDexElements

```
private static Element[] makeDexElements(ArrayList<File> files, File optimizedDirectory,
        ArrayList<IOException> suppressedExceptions) {
    ArrayList<Element> elements = new ArrayList<Element>();
    ...
    for (File file : files) {
        DexFile dex = loadDexFile(file, optimizedDirectory);
    }
    if ((zip != null) || (dex != null)) {
        elements.add(new Element(file, false, zip, dex));
    }
    return elements.toArray(new Element[elements.size()]); //返回赋值给dexElements
}
```

### loadDexFile

```
private static DexFile loadDexFile(File file, File optimizedDirectory)
        throws IOException {
    ...
    return DexFile.loadDex(file.getPath(), optimizedPath, 0);
}
```

## DexFile

#### mCookie

```
xref: /libcore/dalvik/src/main/java/dalvik/system/DexFile.java
 
private long mCookie;
```

### loadDex

```
static public DexFile loadDex(String sourcePathName, String outputPathName,
        int flags) throws IOException {
    return new DexFile(sourcePathName, outputPathName, flags);
}
```

### 构造函数

```
private DexFile(String sourceName, String outputName, int flags) throws IOException {
    ...
    mCookie = openDexFile(sourceName, outputName, flags);
}
```

### openDexFile

```
private static long openDexFile(String sourceName, String outputName,
        int flags) throws IOException {
    return openDexFileNative(new File(sourceName).getCanonicalPath(),
        (outputName == null) ? null : new File(outputName).getCanonicalPath(),
        flags);
}
```

### openDexFileNative

```
native private static long openDexFileNative(String sourceName, String outputName,
        int flags) throws IOException;
```

# java层小结：

```
DexPathList pathList
    Element[] dexElements
        DexFile dex // new Element(file, false, zip, dex)
            long mCookie
 
BaseDexClassLoader中有pathList变量
pathList中有dexElements变量，dexElements中每一个Element由dex初始化
每个DexFile中最关键的就是mCookie，也就是native层的dex_files指针
```

## dalvik_system_DexFile

#### dex_files

支持多multidex

```
xref: /art/runtime/native/dalvik_system_DexFile.cc
 
std::unique_ptr<std::vector<const DexFile*>> dex_files(new std::vector<const DexFile*>());
```

### DexFile_openDexFileNative

```
static jlong DexFile_openDexFileNative(JNIEnv* env, jclass, jstring javaSourceName, jstring javaOutputName, jint) {
  ...
  ClassLinker* linker = Runtime::Current()->GetClassLinker();
  std::unique_ptr<std::vector<const DexFile*>> dex_files(new std::vector<const DexFile*>());
  bool success = linker->OpenDexFilesFromOat(sourceName.c_str(), outputName.c_str(), &error_msgs,dex_files.get());
  if (success || !dex_files->empty()) {
    // In the case of non-success, we have not found or could not generate the oat file.
    // But we may still have found a dex file that we can use.
    return static_cast<jlong>(reinterpret_cast<uintptr_t>(dex_files.release()));
  }
}
```

## class_linker

#### open_oat_file

```
xref: /art/runtime/class_linker.cc
```

### OpenDexFilesFromOat

```
bool ClassLinker::OpenDexFilesFromOat(const char* dex_location, const char* oat_location,
                                      std::vector<std::string>* error_msgs,
                                      std::vector<const DexFile*>* dex_files) {
  if (!DexFile::GetChecksum(dex_location, dex_location_checksum_pointer, &checksum_error_msg)) {
    ...
  }
  // 检查是否我们有一个已经打开的oatfile(内存中)
  const OatFile::OatDexFile* oat_dex_file = FindOpenedOatDexFile(oat_location, dex_location,dex_location_checksum_pointer);
  std::unique_ptr<const OatFile> open_oat_file(
      oat_dex_file != nullptr ? oat_dex_file->GetOatFile() : nullptr);
 
  // 检查是否我们有一个可以使用的的oatfile(磁盘上)，通过oat_location字符串
  if (open_oat_file.get() == nullptr) {
      open_oat_file.reset(FindOatFileInOatLocationForDexFile(dex_location, dex_location_checksum,oat_location, &error_msg));
      ...
    needs_registering = true;
  }
 
  // 如果我们有一个oat文件，检查所有包含的multidex文件以确定我们的dex位置
  bool success = LoadMultiDexFilesFromOatFile(open_oat_file.get(), dex_location,
                                              dex_location_checksum_pointer,
                                              false, error_msgs, dex_files);
  if (success) {
    const OatFile* oat_file = open_oat_file.release();  // Avoid deleting it.
    if (needs_registering) {
      // We opened the oat file, so we must register it.
      RegisterOatFile(oat_file);
    }
    return oat_file->IsExecutable();
  }
 
  // 如果不是这样(要么没有oat文件，要么不匹配)，重新生成并加载oat，通过dex_location字符串
  if (Runtime::Current()->IsDex2OatEnabled() && has_flock && scoped_flock.HasFile()) {
    // Create the oat file.
    open_oat_file.reset(CreateOatFileForDexLocation(dex_location, scoped_flock.GetFile()->Fd(),oat_location, error_msgs));
  }
 
  // Failed, bail.art模式下的解析模式，以上都是编译成oat的编译模式
  if (open_oat_file.get() == nullptr) {
    std::string error_msg;
    // dex2oat was disabled or crashed. Add the dex file in the list of dex_files to make progress.
    DexFile::Open(dex_location, dex_location, &error_msg, dex_files);
    error_msgs->push_back(error_msg);
    return false;
  }
 
  // Try to load again, but stronger checks.
  success = LoadMultiDexFilesFromOatFile(open_oat_file.get(), dex_location,
                                         dex_location_checksum_pointer,
                                         true, error_msgs, dex_files);
  if (success) {
    RegisterOatFile(open_oat_file.release());
    return true;
  }
}
```

**CreateOatFileForDexLocation**

```
const OatFile* ClassLinker::CreateOatFileForDexLocation(const char* dex_location,int fd, const char* oat_location,std::vector<std::string>* error_msgs) {
    GenerateOatFile(dex_location, fd, oat_location, &error_msg)
    std::unique_ptr<OatFile> oat_file(OatFile::Open(oat_location, oat_location, nullptr,!Runtime::Current()->IsCompiler(),&error_msg));
    return oat_file.release();
}
```

### GenerateOatFile

```
bool ClassLinker::GenerateOatFile(const char* dex_filename,
                                  int oat_fd,
                                  const char* oat_cache_filename,
                                  std::string* error_msg) {
    std::string dex2oat(Runtime::Current()->GetCompilerExecutable());
    // 执行dex2oat
    ...
    return Exec(argv, error_msg);
}
```

## dex_file

#### dex_file

```
xref: /art/runtime/dex_file.cc
```

### DexFile::Open

```
bool DexFile::Open(const char* filename, const char* location, std::string* error_msg,std::vector<const DexFile*>* dex_files) {
    std::unique_ptr<const DexFile> dex_file(DexFile::OpenFile(fd.release(), location, true,error_msg));
    if (dex_file.get() != nullptr) {
      dex_files->push_back(dex_file.release());
}
```

### DexFile::OpenFile

```
const DexFile* DexFile::OpenFile(int fd, const char* location, bool verify,
                                 std::string* error_msg) {
    std::unique_ptr<MemMap> map;
    map.reset(MemMap::MapFile(length, PROT_READ, MAP_PRIVATE, fd, 0, location, error_msg));
    const Header* dex_header = reinterpret_cast<const Header*>(map->Begin());
    const DexFile* dex_file = OpenMemory(location, dex_header->checksum_, map.release(), error_msg);
    return dex_file;
}
```

### DexFile::OpenMemory

```
const DexFile* DexFile::OpenMemory(const std::string& location,
                                   uint32_t location_checksum,
                                   MemMap* mem_map,
                                   std::string* error_msg) {
  return OpenMemory(mem_map->Begin(),
                    mem_map->Size(),
                    location,
                    location_checksum,
                    mem_map,
                    error_msg);
}
 
const DexFile* DexFile::OpenMemory(const byte* base,
                                   size_t size,
                                   const std::string& location,
                                   uint32_t location_checksum,
                                   MemMap* mem_map, std::string* error_msg) {
  CHECK_ALIGNED(base, 4);  // various dex file structures must be word aligned
  std::unique_ptr<DexFile> dex_file(new DexFile(base, size, location, location_checksum, mem_map));
  if (!dex_file->Init(error_msg)) {
    return nullptr;
  } else {
    return dex_file.release();
  }
}
```

**DexFile::DexFile**

```
DexFile::DexFile(const byte* base, size_t size,
                 const std::string& location,
                 uint32_t location_checksum,
                 MemMap* mem_map)
    : begin_(base),
      size_(size),
      location_(location),
      location_checksum_(location_checksum),
      mem_map_(mem_map),
      header_(reinterpret_cast<const Header*>(base)),
      string_ids_(reinterpret_cast<const StringId*>(base + header_->string_ids_off_)),
      type_ids_(reinterpret_cast<const TypeId*>(base + header_->type_ids_off_)),
      field_ids_(reinterpret_cast<const FieldId*>(base + header_->field_ids_off_)),
      method_ids_(reinterpret_cast<const MethodId*>(base + header_->method_ids_off_)),
      proto_ids_(reinterpret_cast<const ProtoId*>(base + header_->proto_ids_off_)),
      class_defs_(reinterpret_cast<const ClassDef*>(base + header_->class_defs_off_)),
      find_class_def_misses_(0),
      class_def_index_(nullptr),
      build_class_def_index_mutex_("DexFile index creation mutex") {
  CHECK(begin_ != NULL) << GetLocation();
  CHECK_GT(size_, 0U) << GetLocation();
}
```

# so层小结：

```
dex_files
    open_oat_file
    dex_file
 
DexFile_openDexFileNative中的dex_files即为关键mCookie
有两种赋值形式：
    1.编译模式
        优先打开内存中的oat
        没有的话打开磁盘上的oat
        没有的话打开dex重新编译并加载oat
 
        通过open_oat_file给dex_files赋值
    2.解析模式
        打开dex生成并加载DexFile文件
 
        通过dex_file给dex_files赋值
```

# 总结：

## 关于脱壳

其实这里可以看到关键脱壳点OpenMemory函数，Android6，Android7也是这个函数，通过源码代码执行的分析，可以帮助我们去更好地理解脱壳。

## 关于加固

这里附上基于Android5的不落地加载代码：

```
void *ArtLoader::loadFromMemory(const u1 *pDex, u4 len) {
    auto libArt=dlopen("libart.so",RTLD_NOW);
    if(libArt == nullptr){
        return nullptr ;
    }
    auto openMemory=(void* (*) (const u1* base,
            size_t size,
            const std::string& location,
            uint32_t location_checksum,
            void* mem_map,
            std::string* error_msg))
                    dlsym(libArt,"_ZN3art7DexFile10OpenMemoryEPKhjRKNSt3__112basic_stringIcNS3_11char_traitsIcEENS3_9allocatorIcEEEEjPNS_6MemMapEPS9_");
    if(openMemory!= nullptr){
        auto header=(DexHeader*)pDex;
        std::string location="<memory>";
        std::string error_msg;
        auto dex_file =openMemory(pDex,len,location,header->checksum, nullptr,&error_msg);
        std::unique_ptr<std::vector<const u1*>> dex_files(new std::vector<const u1*>());
        dex_files->push_back(reinterpret_cast<u1*>(dex_file));
        return dex_files.release();
    }
    return nullptr;
}
```

**注：**仅学习笔记，如有理解错误的地方还望包涵~



>略老
>https://github.com/WrBug/DeveloperHelper/blob/develop/xposedmodule/src/main/cpp/util/deviceutils.cpp
>可以看看这个，安卓4.x到9脱壳点都有