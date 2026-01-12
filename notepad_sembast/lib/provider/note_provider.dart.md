# Note Provider 实现详解

## 概述

`note_provider.dart` 文件实现了一个基于 Sembast 数据库的笔记数据提供者，提供了完整的 CRUD 操作和响应式数据流支持。该文件是笔记应用的核心数据层，负责管理笔记的存储、检索、更新和删除操作。

## 核心组件

### 数据转换函数

```dart 8:12:notepad_sembast/lib/provider/note_provider.dart
DbNote snapshotToNote(RecordSnapshot snapshot) {
  return DbNote()
    ..fromMap(snapshot.value as Map)
    ..id = snapshot.key as int;
}
```

`snapshotToNote` 函数负责将数据库快照转换为笔记对象。它从快照中提取数据映射和主键 ID，创建并返回完整的 `DbNote` 实例。

### 笔记列表缓存类

```dart 14:35:notepad_sembast/lib/provider/note_provider.dart
class DbNotes extends ListBase<DbNote> {
  final List<RecordSnapshot<int, Map<String, Object?>>> list;
  late List<DbNote?> _cacheNotes;

  DbNotes(this.list) {
    _cacheNotes = List.generate(list.length, (index) => null);
  }

  @override
  DbNote operator [](int index) {
    return _cacheNotes[index] ??= snapshotToNote(list[index]);
  }

  @override
  int get length => list.length;

  @override
  void operator []=(int index, DbNote? value) => throw 'read-only';

  @override
  set length(int newLength) => throw 'read-only';
}
```

`DbNotes` 类继承自 `ListBase<DbNote>`，提供了一个只读的笔记列表实现。它使用懒加载缓存机制，只有在首次访问时才会将数据库快照转换为笔记对象，提高性能并减少不必要的对象创建。

## 数据库提供者类

### 初始化和配置

```dart 37:47:notepad_sembast/lib/provider/note_provider.dart
class DbNoteProvider {
  static const String dbName = 'notes.db';
  static const int kVersion1 = 1;
  static final String notesStoreName = 'notes';
  final lock = Lock(reentrant: true);
  final DatabaseFactory dbFactory;
  Database? db;

  final notesStore = intMapStoreFactory.store(notesStoreName);

  DbNoteProvider(this.dbFactory);
}
```

`DbNoteProvider` 类是核心的数据库操作类：

- 使用常量定义数据库名称、版本和存储名称
- 实现可重入锁确保数据库操作的线程安全
- 通过依赖注入接收数据库工厂实例
- 使用整数映射存储来管理笔记记录

### 数据库打开和初始化

```dart 49:56:notepad_sembast/lib/provider/note_provider.dart
Future<Database> openPath(String path) async {
  db = await dbFactory.openDatabase(
    path,
    version: kVersion1,
    onVersionChanged: _onVersionChanged,
  );
  return db!;
}
```

`openPath` 方法打开指定路径的数据库，并在版本变化时执行初始化逻辑。

```dart 58:64:notepad_sembast/lib/provider/note_provider.dart
Future<Database> get ready async =>
    db ??= await lock.synchronized<Database>(() async {
      if (db == null) {
        await open();
      }
      return db!;
    });
```

`ready` getter 提供延迟初始化的数据库访问，确保数据库只被打开一次。

### 版本变更处理

```dart 77:94:notepad_sembast/lib/provider/note_provider.dart
void _onVersionChanged(Database db, int oldVersion, int newVersion) async {
  if (oldVersion < kVersion1) {
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
  }
}
```

版本变更处理器在数据库首次创建时添加示例笔记。特别的，欢迎笔记会根据是否运行在 Web 环境显示不同的内容。

## CRUD 操作

### 读取笔记

```dart 66:75:notepad_sembast/lib/provider/note_provider.dart
Future<DbNote?> getNote(int id) async {
  var map = await notesStore.record(id).get(db!);
  // devPrint('getNote: ${map}');
  if (map != null) {
    return DbNote()
      ..fromMap(map)
      ..id = id;
  }
  return null;
}
```

`getNote` 方法通过 ID 获取单个笔记，返回完整的笔记对象或 null。

### 保存笔记

```dart 102:108:notepad_sembast/lib/provider/note_provider.dart
Future saveNote(DbNote updatedNote) async {
  if (updatedNote.id != null) {
    await notesStore.record(updatedNote.id!).put(db!, updatedNote.toMap());
  } else {
    updatedNote.id = await notesStore.add(db!, updatedNote.toMap());
  }
}
```

`saveNote` 方法实现"upsert"逻辑：如果笔记已有 ID 则更新，否则创建新记录并设置生成的 ID。

### 删除笔记

```dart 110:114:notepad_sembast/lib/provider/note_provider.dart
Future deleteNote(int? id) async {
  if (id != null) {
    await notesStore.record(id).delete(db!);
  }
}
```

`deleteNote` 方法删除指定 ID 的笔记记录。

## 响应式数据流

### 数据转换器

```dart 116:124:notepad_sembast/lib/provider/note_provider.dart
var notesTransformer =
    StreamTransformer<
      List<RecordSnapshot<int, Map<String, Object?>>>,
      List<DbNote>
    >.fromHandlers(
      handleData: (snapshotList, sink) {
        sink.add(DbNotes(snapshotList));
      },
    );
```

`notesTransformer` 将数据库快照列表转换为笔记对象列表，使用缓存的 `DbNotes` 类。

```dart 126:134:notepad_sembast/lib/provider/note_provider.dart
var noteTransformer =
    StreamTransformer<
      RecordSnapshot<int, Map<String, Object?>>?,
      DbNote?
    >.fromHandlers(
      handleData: (snapshot, sink) {
        sink.add(snapshot == null ? null : snapshotToNote(snapshot));
      },
    );
```

`noteTransformer` 处理单个笔记的快照转换，支持 null 值。

### 监听器方法

```dart 137:142:notepad_sembast/lib/provider/note_provider.dart
Stream<List<DbNote>> onNotes() {
  return notesStore
      .query(finder: Finder(sortOrders: [SortOrder('date', false)]))
      .onSnapshots(db!)
      .transform(notesTransformer);
}
```

`onNotes` 方法返回按日期降序排列的所有笔记的响应式流。每当笔记数据发生变化时，监听器会自动推送更新后的笔记列表。

```dart 145:147:notepad_sembast/lib/provider/note_provider.dart
Stream<DbNote?> onNote(int id) {
  return notesStore.record(id).onSnapshot(db!).transform(noteTransformer);
}
```

`onNote` 方法监听特定笔记的变化，返回该笔记的响应式流。

## 工具方法

### 清空所有笔记

```dart 149:151:notepad_sembast/lib/provider/note_provider.dart
Future clearAllNotes() async {
  await notesStore.delete(db!);
}
```

`clearAllNotes` 方法删除存储中的所有笔记记录。

### 数据库关闭

```dart 153:155:notepad_sembast/lib/provider/note_provider.dart
Future close() async {
  await db!.close();
}
```

`close` 方法关闭数据库连接。

### 数据库删除

```dart 157:159:notepad_sembast/lib/provider/note_provider.dart
Future deleteDb() async {
  await dbFactory.deleteDatabase(await fixPath(dbName));
}
```

`deleteDb` 方法完全删除数据库文件，用于重置或清理。

## 设计模式和架构特点

1. **响应式设计**：使用 Stream API 提供实时数据更新，支持 Flutter 的声明式 UI 模式

2. **缓存优化**：`DbNotes` 类实现懒加载缓存，减少对象创建开销

3. **线程安全**：使用可重入锁确保数据库操作的并发安全

4. **依赖注入**：通过构造函数注入数据库工厂，提高可测试性和灵活性

5. **数据转换**：清晰分离数据库层和业务逻辑层，通过转换器处理数据映射

6. **平台适配**：通过 `fixPath` 方法支持不同平台的路径处理（在基类中实现具体逻辑）

这个提供者类为笔记应用提供了完整的数据管理解决方案，结合 Sembast 的强大功能和响应式编程模式，为 Flutter UI 层提供了可靠的数据支持。
