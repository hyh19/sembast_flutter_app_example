# model.dart - 笔记数据模型

## 概述

`model.dart` 文件定义了笔记应用的数据库记录模型结构。该文件包含一个 `DbNote` 类，用于表示笔记数据的结构和字段定义。

## 导入依赖

```dart 1:1:notepad_sembast/lib/model/model.dart
import 'package:tekartik_notepad_sembast_app/db/db.dart';
```

文件导入了自定义的数据库包，该包提供了基础的数据库记录类型和字段定义功能。

## DbNote 类定义

### 类声明

```dart 3:3:notepad_sembast/lib/model/model.dart
class DbNote extends DbRecord {
```

`DbNote` 类继承自 `DbRecord`，这是基于 `cv` 包的模型基类。`DbRecord` 提供了基本的数据库记录功能，包括：

- `id` 字段：记录的唯一标识符
- 序列化/反序列化方法
- 与 Sembast 数据库的集成

### 字段定义

```dart 4:6:notepad_sembast/lib/model/model.dart
  final title = CvField<String>('title');
  final content = CvField<String>('content');
  final date = CvField<int>('date');
```

类定义了三个核心字段：

- **`title`**: 笔记标题，字符串类型
- **`content`**: 笔记内容，字符串类型
- **`date`**: 创建或修改日期，整数类型（通常为时间戳）

每个字段都使用 `CvField<T>` 泛型类定义，其中：

- 类型参数 `T` 指定字段的数据类型
- 构造函数参数为字段名称，用于数据库存储和序列化

### 字段列表重写

```dart 8:9:notepad_sembast/lib/model/model.dart
  @override
  List<CvField> get fields => [title, content, date];
```

重写了基类的 `fields` getter，返回包含所有字段的列表。这个列表用于：

- 数据库模式的定义
- 数据验证
- 序列化/反序列化的字段映射

## 在应用中的使用

### 数据操作

在 `note_provider.dart` 中，`DbNote` 对象通过以下方式使用：

```dart 8:12:notepad_sembast/lib/provider/note_provider.dart
DbNote snapshotToNote(RecordSnapshot snapshot) {
  return DbNote()
    ..fromMap(snapshot.value as Map)
    ..id = snapshot.key as int;
}
```

- 从数据库快照创建 `DbNote` 实例
- 使用 `fromMap()` 方法从 Map 反序列化数据
- 设置记录的 ID

### 数据存储

保存笔记时使用 `toMap()` 方法序列化：

```dart 102:108:notepad_sembast/lib/provider/note_provider.dart
  Future saveNote(DbNote updatedNote) async {
    if (updatedNote.id != null) {
      await notesStore.record(updatedNote.id!).put(db!, updatedNote.toMap());
    } else {
      updatedNote.id = await notesStore.add(db!, updatedNote.toMap());
    }
  }
```

### 数据查询

查询操作使用字段名称进行排序：

```dart 137:141:notepad_sembast/lib/provider/note_provider.dart
  Stream<List<DbNote>> onNotes() {
    return notesStore
        .query(finder: Finder(sortOrders: [SortOrder('date', false)]))
        .onSnapshots(db!)
        .transform(notesTransformer);
  }
```

## 数据结构特点

1. **类型安全**：使用泛型字段确保类型安全
2. **序列化支持**：自动支持 Map 格式的序列化/反序列化
3. **数据库集成**：与 Sembast 数据库无缝集成
4. **响应式更新**：支持流式数据监听和更新
5. **字段验证**：通过字段定义提供基础验证

## 字段使用示例

在数据库版本变更时创建示例数据：

```dart 80:92:notepad_sembast/lib/provider/note_provider.dart
      await notesStore.addAll(db, [
        (DbNote()
              ..title.v = 'Simple title'
              ..content.v = 'Simple content'
              ..date.v = 1)
            .toMap(),
        (DbNote()
              ..title.v = 'Welcome to NotePad'
              ..content.v =
                  'Enter your notes\n\nThis is a content. Just tap anywhere to edit the note.\n'
                  '${kIsWeb ? '\nYou can open multiple tabs or windows and see that the content is the same in all tabs' : ''}'
              ..date.v = 2)
            .toMap(),
      ]);
```

通过 `.v` 属性访问和设置字段值，这是 `CvField` 提供的值访问器。

## 架构意义

`DbNote` 类是应用数据层的核心组成部分：

- **数据建模**：定义了笔记实体的结构
- **持久化接口**：提供了与数据库的桥梁
- **业务逻辑支持**：为上层业务逻辑提供数据访问接口
- **跨平台兼容**：支持 Web 和移动端的统一数据模型

这种设计使得数据模型既保持了类型安全，又提供了灵活的序列化能力，是 Flutter + Sembast 应用中数据建模的最佳实践。
