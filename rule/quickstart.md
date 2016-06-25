/*
Title: 快速入门
Sort: 0
Tmpl : page-quick
*/


## 1.理解规则表达式

Wilddog提供了一种类`JavaScript`语法的，基于表达式的规则语言，用于定义数据的读写权限和校验规则。与终端用户认证机制相结合，就可以控制哪些用户有权限访问哪些数据，确保用户数据安全。

规则表达式有三种类型：`.read`，`.write`，`.validate`：

规则表达式     | 描述
-------- | ---
.read | 声明在何种情况下允许用户读取数据
.write    | 声明在何种情况下允许用户更新数据
.validate     | 声明数据的格式和约束，例如是否拥有子节点，数据类型等。

每个应用在新建时都有默认的规则表达式，它允许任何人读写任何节点的数据：

```js
{
  "rules": {
    ".read": true,
    ".write": true
  }
}
```
<br>
## 2.内置对象

规则表达式中有许多内置的对象可用：

内置对象     | 描述
-------- | ---
now| 云端的时间戳，以毫秒为单位。
root|`RuleDataSnapshot`类型的对象，代表操作之前，数据根节点`/`的数据的引用。
newData| `RuleDataSnapshot`类型的对象，代表假设数据操作成功，之后此节点下的数据，也就是此节点下的旧数据和本次操作写入的新数据合并之后的数据。
data| `RuleDataSnapshot`类型的对象，代表此节点被操作前的原始数据。
$variables| 节点的名称变量，代表动态路径的通配符。 
auth| `Auth`类型对象，代表已经登录用户对象，参考[终端用户认证](/rule/guide#3-ji-yu-zhong-duan-yong-hu-ren-zheng-di-an-quan0)。

在规则表达式中可以使用这些内置对象。例如，下面的规则表达式要求写入`/foo`节点的数据必须是字符串类型，且长度小于100：

```javascript
{
  "rules": {
    "foo": {
      // /foo节点可被任何人读取。
      ".read": true,
      
      // /foo节点可被任何人写入。
      ".write": true,
      
      // 写入/foo 的数据必须是字符串类型且长度小于100。
      ".validate": "newData.isString() && newData.val().length() < 100"
    }
  }
}
```
<br>
## 3.动态路径

规则表达式的结构应该和数据的结构一致。例如，你存储了一个message的列表，数据是这样的：

```json
{
  "messages": {
    "message0": {
      "content": "Hello",
      "timestamp": 1405704370369
    },
    "message1": {
      "content": "Goodbye",
      "timestamp": 1405704395231
    },
    ...
  }
}
```

那么规则表达式的结构应该和数据一致。可以使用`$`表示动态的子节点路径。此外，可以使用`.validate`约束数据的格式。例如，下面的规则表达式中，`$message`代表`/messages/`节点下的每一个子节点数据：

```json
{
  "rules": {
    "messages": {
      "$message": {
        // 只有最近10分钟的message数据可以被读取
        ".read": "data.child('timestamp').val() > (now - 600000)",
        
        // 每个message数据都必须包含一个字符串的content字段和一个数值时间戳
        ".validate": "newData.hasChildren(['content', 'timestamp']) && newData.child('content').isString() && newData.child('timestamp').isNumber()"
      }
    }
  }
}
```

