# Godot Engine Core 核心架构学习笔记

## 学习状态：⏳ 进行中

---

## 一、项目结构概览

### Godot 根目录结构

| 目录 | 说明 |
|------|------|
| `core/` | **核心库** - 基本类型、String、Array、Dictionary、Variant、Math、VM等 |
| `scene/` | **场景系统** - Node、SceneTree、2D/3D场景、Resource管理等 |
| `servers/` | **服务器** - 渲染、物理、音频、导航等子系统 |
| `editor/` | **编辑器** - 编辑器UI、调试器、导入器、导出器 |
| `main/` | **程序入口** - 主循环、初始化、App主类 |
| `drivers/` | **设备驱动** - 输入设备、显示器、音频后端 |
| `platform/` | **平台特定** - Windows、Linux、macOS、Android等 |
| `modules/` | **可插拔模块** - 可选的模块系统（C#、VR、GodotSharp等） |
| `tests/` | **单元测试** |
| `thirdparty/` | **第三方库** - 开源组件 |

---

## 二、Core 核心库结构

### 2.1 core/ 子目录说明

| 目录 | 说明 |
|------|------|
| `variant/` | **Variant类型系统** - Godot的核心统一类型 |
| `object/` | **对象系统** - Object、ClassDB、MethodBind |
| `templates/` | **模板容器** - Vector、List、Map、PoolArray等 |
| `string/` | **字符串处理** - String、StringName |
| `math/` | **数学库** - Vector、Transform、Quaternion等 |
| `os/` | **操作系统抽象** - FileSystem、Thread、Mutex等 |
| `io/` | **输入输出** - ResourceLoader、ResourceSaver等 |

---

## 三、Variant 类型系统

### 3.1 概述
Variant 是 Godot 的**统一类型系统**，可以表示引擎中的任何类型。

### 3.2 类型枚举
位置：`core/variant/variant.h` 第 96-145 行

```cpp
enum Type {
    NIL,           // 空值
    BOOL,          // 布尔
    INT,           // 整数
    FLOAT,         // 浮点数
    STRING,        // 字符串
    VECTOR2,       // 2D向量
    VECTOR2I,      // 2D整数向量
    RECT2,         // 2D矩形
    VECTOR3,       // 3D向量
    TRANSFORM2D,   // 2D变换
    TRANSFORM3D,   // 3D变换
    OBJECT,        // 对象
    CALLABLE,      // 可调用
    SIGNAL,        // 信号
    DICTIONARY,     // 字典
    ARRAY,         // 数组
    PACKED_BYTE_ARRAY,  // 字节数组
    // ... 更多类型
    VARIANT_MAX    // 类型数量
};
```

### 3.3 核心特性
- **类型标记 + 联合体**：使用 `type` 标记当前类型，`_data` 联合体存储值
- **操作符重载**：支持 `+`, `-`, `*`, `/`, `==`, `<` 等所有操作符
- **方法调用**：通过 `call()`, `callp()` 调用方法
- **迭代器支持**：支持 `iter_init()`, `iter_next()`, `iter_get()`
- **类型转换**：自动进行类型转换和类型检查

### 3.4 关键方法

方法 | 说明 |
|------|------|
| `get_type()` | 获取当前类型 |
| `can_convert()` | 检查类型可转换性 |
| `booleanize()` | 转换为布尔值 |
| `hash()` | 计算哈希值 |
| `duplicate()` | 复制值 |

## 四、Object 对象系统

### 4.1 概述
Object 是 Godot **所有对象的基类**，所有 Node、Resource、Scene 都继承自 Object。

### 4.2 核心特性

#### 信号系统

```cpp
// 位置：core/object/object.h
Error emit_signal(StringName p_signal, ...);    // 发送信号
Error connect(StringName p_signal, Callable p_callable, uint32_t p_flags = 0);
void disconnect(StringName p_signal, Callable p_callable);
bool is_connected(StringName p_signal, Callable p_callable);

```

### 属性系统

```cpp
void set(StringName p_property, Variant p_value);
Variant get(StringName p_property);
void get_property_list();
bool property_can_revert(StringName p_property);
```

### 元数据

```cpp
void set_meta(StringName p_name, Variant p_value);
Variant get_meta(StringName p_name);
bool has_meta(StringName p_name);
void remove_meta(StringName p_name);

```

### 脚本实例

```cpp
void set_script(Script *p_script);
ScriptInstance *get_script_instance();
```

### 4.3 对象生命周期

```cpp
NOTIFICATION_POSTINITIALIZE   // 初始化完成后
NOTIFICATION_PREDELETE         // 删除前
NOTIFICATION_EXTENDING_RELOADED // 扩展重新加载
```

### 4.4 ObjectDB对象数据库

管理所有 Object 实例的全局数据库，通过 ObjectID 快速查找对象。

## 五、类型注册流程

### 5.1 register_core_types.cpp

位置：`core/register_core_types.cpp

```cpp
void register_core_types() {
    // 1. 初始化核心子系统
    ObjectDB::setup();        // 设置对象数据库
    StringName::setup();      // 设置字符串名称系统
    register_global_constants();
    CoreStringNames::create();

    // 2. 注册核心类
    GDREGISTER_CLASS(Object);
    GDREGISTER_CLASS(RefCounted);
    GDREGISTER_CLASS(WeakRef);
    GDREGISTER_CLASS(Resource);

    // 3. 注册 Variant 类型
    Variant::register_types();

    // 4. 注册资源格式加载器
    resource_saver_binary.instantiate();
    ResourceSaver::add_resource_format_saver(resource_saver_binary);
    resource_loader_binary.instantiate();
    ResourceLoader::add_resource_format_loader(resource_loader_binary);

    // ... 更多注册
}
```

## 六、核心宏定义

### 6.1 GDCLASS - 定义 Godot 类
#define GDCLASS( ... )  // 用于定义一个 Godot 类

### 6.2 GDREGISTER_CLASS - 注册类
#define GDREGISTER_CLASS( ... )  // 将类注册到 ClassDB

### 6.3 属性注册宏
```cpp
ADD_PROPERTY(...)           // 添加属性
ADD_SIGNAL(...)             // 添加信号
ADD_GROUP(...)              // 添加属性分组
```

## 七、关键设计模式

### 7.1 单例模式
Engine、OS、ResourceLoader 等核心服务使用单例模式：

### 7.2 类注册机制
所有可脚本化的类都需要：
1. 继承 Object 或其子类
2. 使用 GDCLASS 宏
3. 在 register_core_types() 中调用 GDREGISTER_CLASS

### 7.3 虚拟方法绑定
通过 GDVIRTUAL_* 宏实现 C++ 与脚本的虚拟方法调用。

## 八、模块关系图

┌─────────────────────────────────────┐
│           Main Loop                 │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│           Scene Tree                │
│    (Node hierarchy, signals)        │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│     Object (base class)             │
│  - Property system                  │
│  - Signal system                    │
│  - Script instance                  │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│          Variant                    │
│  - Unifies all types               │
│  - Operators, methods, cast        │
└─────────────────────────────────────┘

## 九、学习进度

- [ ] Variant 类型系统
- [ ] Object 对象系统
- [ ] ClassDB 类数据库
- [ ] StringName 字符串系统
- [ ] 类型注册流程
