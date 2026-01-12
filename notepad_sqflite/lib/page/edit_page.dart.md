# EditNotePage - 笔记编辑页面

## 概述

`EditNotePage` 是一个 Flutter 页面组件，用于创建新笔记或编辑现有笔记。该页面提供了完整的表单验证、数据保存、删除功能，并包含脏数据检测机制来防止意外丢失用户输入。

## 主要功能特性

- **双模式支持**：支持新建笔记和编辑现有笔记两种模式
- **表单验证**：确保标题和内容字段都不为空
- **脏数据检测**：在用户尝试退出时检测是否有未保存的更改
- **删除功能**：提供删除笔记的确认对话框
- **自动保存**：支持保存笔记并自动更新时间戳

## 代码结构分析

### 类定义

```dart 9:18:notepad_sembast/lib/page/edit_page.dart
class EditNotePage extends StatefulWidget {
  /// null when adding a note
  final DbNote? initialNote;

  const EditNotePage({super.key, required this.initialNote});

  @override
  // ignore: library_private_types_in_public_api
  _EditNotePageState createState() => _EditNotePageState();
}
```

`EditNotePage` 继承自 `StatefulWidget`，接收一个可选的 `DbNote` 参数：

- 当 `initialNote` 为 `null` 时，表示新建笔记模式
- 当 `initialNote` 不为 `null` 时，表示编辑现有笔记模式

### 状态类结构

```dart 20:24:notepad_sembast/lib/page/edit_page.dart
class _EditNotePageState extends State<EditNotePage> {
  final _formKey = GlobalKey<FormState>();
  TextEditingController? _titleTextController;
  TextEditingController? _contentTextController;

  int? get _noteId => widget.initialNote?.id;
```

状态类维护以下核心成员：

- `_formKey`：用于表单验证的全局键
- `_titleTextController` 和 `_contentTextController`：分别管理标题和内容的文本输入
- `_noteId`：便捷 getter，用于获取当前笔记的 ID（编辑模式下使用）

## 初始化逻辑

```dart 28:36:notepad_sembast/lib/page/edit_page.dart
@override
void initState() {
  super.initState();
  _titleTextController = TextEditingController(
    text: widget.initialNote?.title.v,
  );
  _contentTextController = TextEditingController(
    text: widget.initialNote?.content.v,
  );
}
```

在 `initState` 中初始化文本控制器：

- 如果是编辑模式，会用现有笔记的标题和内容预填充文本字段
- 如果是新建模式，文本字段将为空

## 保存功能

```dart 38:56:notepad_sembast/lib/page/edit_page.dart
Future save() async {
  if (_formKey.currentState!.validate()) {
    _formKey.currentState!.save();
    await noteProvider.saveNote(
      DbNote()
        ..id = _noteId
        ..title.v = _titleTextController!.text.trimmedNonEmpty()
        ..content.v = _contentTextController!.text.trimmedNonEmpty()
        ..date.v = DateTime.now().millisecondsSinceEpoch,
    );
    // ignore: use_build_context_synchronously
    Navigator.pop(context);
    // Pop twice when editing
    if (_noteId != null) {
      // ignore: use_build_context_synchronously
      Navigator.pop(context);
    }
  }
}
```

保存逻辑包含以下步骤：

1. **表单验证**：确保所有必填字段都已填写
2. **数据保存**：通过 `noteProvider` 保存笔记，自动设置当前时间戳
3. **导航处理**：
   - 新建模式：返回到列表页面
   - 编辑模式：连续弹出两次，返回到列表页面（从编辑页面回到详情页面，再回到列表页面）

## 页面布局

### 应用栏配置

```dart 111:167:notepad_sembast/lib/page/edit_page.dart
appBar: AppBar(
  title: Text('Edit Note'),
  actions: <Widget>[
    if (_noteId != null)
      IconButton(
        icon: Icon(Icons.delete),
        onPressed: () async {
          // ignore: use_build_context_synchronously
          if (await showDialog<bool>(
                context: context,
                barrierDismissible: false, // user must tap button!
                builder: (BuildContext context) {
                  return AlertDialog(
                    title: Text('Delete note?'),
                    content: SingleChildScrollView(
                      child: ListBody(
                        children: <Widget>[
                          Text('Tap \'YES\' to confirm note deletion.'),
                        ],
                      ),
                    ),
                    actions: <Widget>[
                      TextButton(
                        onPressed: () {
                          Navigator.of(context).pop(true);
                        },
                        child: Text('YES'),
                      ),
                      TextButton(
                        onPressed: () {
                          Navigator.of(context).pop(false);
                        },
                        child: Text('NO'),
                      ),
                    ],
                  );
                },
              ) ??
              false) {
            await noteProvider.deleteNote(widget.initialNote!.id);
            // Pop twice to go back to the list
            // ignore: use_build_context_synchronously
            Navigator.of(context).pop();
            // ignore: use_build_context_synchronously
            Navigator.of(context).pop();
          }
        },
      ),
    // action button
    IconButton(
      icon: Icon(Icons.save_alt),
      onPressed: () {
        save();
      },
    ),
  ],
),
```

应用栏包含：

- **标题**："Edit Note"
- **删除按钮**：仅在编辑模式下显示，点击后显示确认对话框
- **保存按钮**：用于保存笔记更改

### 表单界面

```dart 168:205:notepad_sembast/lib/page/edit_page.dart
body: Padding(
  padding: const EdgeInsets.all(16.0),
  child: ListView(
    children: <Widget>[
      Form(
        key: _formKey,
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          mainAxisAlignment: MainAxisAlignment.start,
          children: <Widget>[
            TextFormField(
              decoration: InputDecoration(
                labelText: 'Title',
                border: OutlineInputBorder(),
              ),
              controller: _titleTextController,
              validator: (val) =>
                  val!.isNotEmpty ? null : 'Title must not be empty',
            ),
            SizedBox(height: 16),
            TextFormField(
              decoration: InputDecoration(
                labelText: 'Content',
                border: OutlineInputBorder(),
              ),
              controller: _contentTextController,
              validator: (val) => val!.isNotEmpty
                  ? null
                  : 'Description must not be empty',
              keyboardType: TextInputType.multiline,
              maxLines: 5,
            ),
          ],
        ),
      ),
    ],
  ),
),
```

表单包含两个主要字段：

1. **标题字段**：单行文本输入，必填验证
2. **内容字段**：多行文本输入（最多5行），必填验证

## 退出保护机制

### PopScope 配置

```dart 60:109:notepad_sembast/lib/page/edit_page.dart
return PopScope(
  canPop: isDirty(),
  onPopInvokedWithResult: (invoked, result) async {
    if (invoked) {
      return;
    }
    var dirty = isDirty();
    if (dirty) {
      var doDiscard =
          (await showDialog<bool>(
            context: context,
            barrierDismissible: false, // user must tap button!
            builder: (BuildContext context) {
              return AlertDialog(
                title: Text('Discard change?'),
                content: SingleChildScrollView(
                  child: ListBody(
                    children: <Widget>[
                      Text('Content has changed.'),
                      SizedBox(height: 12),
                      Text('Tap \'CONTINUE\' to discard your changes.'),
                    ],
                  ),
                ),
                actions: <Widget>[
                  TextButton(
                    onPressed: () {
                      Navigator.pop(context, true);
                    },
                    child: Text('CONTINUE'),
                  ),
                  TextButton(
                    onPressed: () {
                      Navigator.pop(context, false);
                    },
                    child: Text('CANCEL'),
                  ),
                ],
              );
            },
          )) ??
          false;
      if (!doDiscard) {
        return;
      }
    }
    if (context.mounted) {
      Navigator.of(context).pop();
    }
  },
```

使用 `PopScope` 实现退出保护：

1. **canPop 控制**：当有未保存更改时，阻止直接返回
2. **确认对话框**：询问用户是否要放弃更改
3. **条件导航**：只有用户确认放弃或没有更改时才允许退出

## 脏数据检测

```dart 210:225:notepad_sembast/lib/page/edit_page.dart
bool isTextDirty(String? newText, String? originalText) {
  return newText?.trimmedNonEmpty() != originalText?.trimmedNonEmpty();
}

bool isDirty() {
  var dirty = false;
  if (isTextDirty(_titleTextController!.text, widget.initialNote?.title.v)) {
    dirty = true;
  } else if (isTextDirty(
    _contentTextController!.text,
    widget.initialNote?.content.v,
  )) {
    dirty = true;
  }
  return dirty;
}
```

脏数据检测逻辑：

- **文本比较**：使用 `trimmedNonEmpty()` 方法去除空白字符后进行比较
- **整体检测**：检查标题或内容任一字段是否有更改
- **短路逻辑**：一旦发现任一字段有更改，就立即返回 true

## 依赖项和导入

```dart 1:8:notepad_sembast/lib/page/edit_page.dart
// ignore_for_file: prefer_const_constructors, prefer_const_literals_to_create_immutables

import 'package:flutter/material.dart';
import 'package:tekartik_common_utils/common_utils_import.dart';
import 'package:tekartik_common_utils/string_utils.dart';
import 'package:tekartik_notepad_sembast_app/app.dart';
import 'package:tekartik_notepad_sembast_app/model/model.dart';
```

主要依赖：

- **Flutter Material**：UI 组件和基础功能
- **tekartik_common_utils**：提供 `trimmedNonEmpty()` 等实用方法
- **应用相关**：笔记提供者和服务，数据模型

## 使用场景和流程

### 新建笔记流程

1. 用户点击新建按钮，进入编辑页面
2. 页面显示空的标题和内容字段
3. 用户输入内容后点击保存
4. 验证表单，通过后保存并返回列表页面

### 编辑笔记流程

1. 用户选择现有笔记进入编辑页面
2. 页面预填充现有标题和内容
3. 用户修改内容后点击保存
4. 验证表单，通过后更新并返回列表页面

### 删除笔记流程

1. 在编辑模式下点击删除按钮
2. 显示确认对话框
3. 用户确认后删除笔记并返回列表页面

## 注意事项

- **状态管理**：页面使用传统的 StatefulWidget 模式，没有集成状态管理库
- **错误处理**：保存和删除操作没有显式的错误处理逻辑
- **用户体验**：使用英文界面文本，可能需要国际化支持
- **性能考虑**：文本控制器在组件销毁时需要手动释放（代码中未显示）

这个页面实现了一个功能完整的笔记编辑界面，提供了良好的用户体验和数据完整性保护。
