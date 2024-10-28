###### 各组件版本：
python：3.13版本

lua：5.4.7

protoc :libprotoc 28.3

###### 流程：
+ github上[https://github.com/sean-lin/protoc-gen-lua](https://github.com/sean-lin/protoc-gen-lua)下载了一份代码，按照readme的教程进行操作：我当前是lua5.4版本，对文件进行了如下修改。
+ 先更改了一下Makefile文件，将lua5.1改为lua：

```c
将LAGS=`pkg-config --cflags lua5.1` -std=gnu99
LDFLAGS=`pkg-config --libs lua5.1`
改为：
CFLAGS=`pkg-config --cflags lua` -std=gnu99
LDFLAGS=`pkg-config --libs lua`
```

+ 修改pb.c文件，

```c
#ifdef _ALLBSD_SOURCE
#include <machine/endian.h>
#else
#include <endian.h>
#endif
替换为
#include <endian.h>
```

这个时候直接去运行make会报错

![](https://cdn.nlark.com/yuque/0/2024/png/40375869/1730077640904-f54df4a9-70c7-47a3-87b4-2909405691d3.png)

+ 继续修改pb.c文件

全局搜一下，将static const struct luaL_reg修改为static const struct luaL_Reg

```c
// int luaopen_pb (lua_State *L)
// {
//     luaL_newmetatable(L, IOSTRING_META);
//     lua_pushvalue(L, -1);
//     lua_setfield(L, -2, "__index");
//     luaL_register(L, NULL, _c_iostring_m);

//     luaL_register(L, "pb", _pb);
//     return 1;
// }
// 替换为
int luaopen_pb (lua_State *L)
{
    luaL_newmetatable(L, IOSTRING_META);
    lua_pushvalue(L, -1);
    lua_setfield(L, -2, "__index");
    // Register the metatable functions
    luaL_setfuncs(L, _c_iostring_m, 0);

    // Create a new table for the "pb" library
    lua_newtable(L);

    // Register the library functions
    luaL_setfuncs(L, _pb, 0);

    // Set the global "pb" table
    lua_setglobal(L, "pb");

} 
```

+ 这个时候执行make，依然会报错pkg-config 找不到lua5.4.pc文件，先执行如下指令找到这个文件，解决办法：

执行find /usr -name "lua*.pc" 2>/dev/null

找到文件存于<font style="color:#000000;">/usr/local/Cellar/lua/5.4.7/lib/pkgconfig</font>

<font style="color:#000000;">执行指令：</font>

<font style="color:#000000;">export PKG_CONFIG_PATH=/usr/local/Cellar/lua/5.4.7/lib/pkgconfig:$PKG_CONFIG_PATH</font>

<font style="color:#000000;">注意这是个临时设置，仅当前会话有效。如需设计永久见下：</font>

```plain
# 永久设置（添加到 ~/.bashrc 或 ~/.zshrc）
echo 'export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH' >> ~/.bashrc
source ~/.bashrc
```

过程中可以使用以下指令看是否设置成功：

```plain
# 验证 pkg-config 能否正确找到并使用 Lua
pkg-config --cflags lua5.4
pkg-config --libs lua5.4
```

现在make就可以成功了，将生成的<font style="color:rgb(77, 77, 77);">pb.so拷贝到/usr/local/bin 目录下就大功告成了。</font>

+ <font style="color:rgb(77, 77, 77);">接下来需要做如下几件事情，确保电脑上已经安装了protobuf，可以使用brew Install protobuf 安装，因为需要执行protoc指令，并且python环境下安装了protobuf，可使用pip3 install protobuf安装，我安装的是python3.13。</font>
+ <font style="color:rgb(77, 77, 77);">然后去你的python环境下/site-packages/google/protobuf/compiler/plugin_pb2.py，找到plugin_pb2.py这个文件。并把这个文件移动至/usr/local/bin 目录。接下来就需要修改protoc-gen-lua。</font>

<font style="color:rgb(77, 77, 77);">将代码改成：</font>

注意第一行代码，#!/usr/local/openresty/lualib/luaenv/bin/python，需要你把这里改成你自己的python环境。然后把<font style="color:rgb(77, 77, 77);">protoc-gen-lua移动至/usr/local/bin 目录。</font>

+ <font style="color:rgb(77, 77, 77);">将代码改成：</font>

```c
#!/usr/local/openresty/lualib/luaenv/bin/python
# protoc-gen-erl
# Google's Protocol Buffers project, ported to lua.
import sys
import os.path as path
from io import StringIO
import os
import plugin_pb2
import google.protobuf.descriptor_pb2 as descriptor_pb2
_packages = {}
_files = {}
_message = {}
FDP = plugin_pb2.google_dot_protobuf_dot_descriptor__pb2.FieldDescriptorProto
if sys.platform == "win32":
    import msvcrt, os
    msvcrt.setmode(sys.stdin.fileno(), os.O_BINARY)
    msvcrt.setmode(sys.stdout.fileno(), os.O_BINARY)
class CppType:
    CPPTYPE_INT32       = 1
    CPPTYPE_INT64       = 2
    CPPTYPE_UINT32      = 3
    CPPTYPE_UINT64      = 4
    CPPTYPE_DOUBLE      = 5
    CPPTYPE_FLOAT       = 6
    CPPTYPE_BOOL        = 7
    CPPTYPE_ENUM        = 8
    CPPTYPE_STRING      = 9
    CPPTYPE_MESSAGE     = 10
CPP_TYPE ={
    FDP.TYPE_DOUBLE         : CppType.CPPTYPE_DOUBLE,
    FDP.TYPE_FLOAT          : CppType.CPPTYPE_FLOAT,
    FDP.TYPE_INT64          : CppType.CPPTYPE_INT64,
    FDP.TYPE_UINT64         : CppType.CPPTYPE_UINT64,
    FDP.TYPE_INT32          : CppType.CPPTYPE_INT32,
    FDP.TYPE_FIXED64        : CppType.CPPTYPE_UINT64,
    FDP.TYPE_FIXED32        : CppType.CPPTYPE_UINT32,
    FDP.TYPE_BOOL           : CppType.CPPTYPE_BOOL,
    FDP.TYPE_STRING         : CppType.CPPTYPE_STRING,
    FDP.TYPE_MESSAGE        : CppType.CPPTYPE_MESSAGE,
    FDP.TYPE_BYTES          : CppType.CPPTYPE_STRING,
    FDP.TYPE_UINT32         : CppType.CPPTYPE_UINT32,
    FDP.TYPE_ENUM           : CppType.CPPTYPE_ENUM,
    FDP.TYPE_SFIXED32       : CppType.CPPTYPE_INT32,
    FDP.TYPE_SFIXED64       : CppType.CPPTYPE_INT64,
    FDP.TYPE_SINT32         : CppType.CPPTYPE_INT32,
    FDP.TYPE_SINT64         : CppType.CPPTYPE_INT64
}
def printerr(*args):
    sys.stderr.write(" ".join(args))
    sys.stderr.write("\n")
    sys.stderr.flush()
class TreeNode(object):
    def __init__(self, name, parent=None, filename=None, package=None):
        super(TreeNode, self).__init__()
        self.child = []
        self.parent = parent
        self.filename = filename
        self.package = package
        if parent:
            self.parent.add_child(self)
        self.name = name
    def add_child(self, child):
        self.child.append(child)
    def find_child(self, child_names):
        if child_names:
            for i in self.child:
                if i.name == child_names[0]:
                    return i.find_child(child_names[1:])
            raise StandardError
        else:
            return self
    def get_child(self, child_name):
        for i in self.child:
            if i.name == child_name:
                return i
        return None
    def get_path(self, end = None):
        pos = self
        out = []
        while pos and pos != end:
            out.append(pos.name)
            pos = pos.parent
        out.reverse()
        return '.'.join(out)
    def get_global_name(self):
        return self.get_path()
    def get_local_name(self):
        pos = self
        while pos.parent:
            pos = pos.parent
            if self.package and pos.name == self.package[-1]:
                break
        return self.get_path(pos)
    def __str__(self):
        return self.to_string(0)
    def __repr__(self):
        return str(self)
    def to_string(self, indent = 0):
        return ' '*indent + '<TreeNode ' + self.name + '(\n' + \
                ','.join([i.to_string(indent + 4) for i in self.child]) + \
                ' '*indent +')>\n'
class Env(object):
    filename = None
    package = None
    extend = None
    descriptor = None
    message = None
    context = None
    register = None
    def __init__(self):
        self.message_tree = TreeNode('')
        self.scope = self.message_tree
    def get_global_name(self):
        return self.scope.get_global_name()
    def get_local_name(self):
        return self.scope.get_local_name()
    def get_ref_name(self, type_name):
        try:
            node = self.lookup_name(type_name)
        except:
            # if the child doesn't be founded, it must be in this file
            return type_name[len('.'.join(self.package)) + 1:]
        if node.filename != self.filename:
            return node.filename + '_pb.' + node.get_local_name()
        return node.get_local_name()
    def lookup_name(self, name):
        names = name.split('.')
        if names[0] == '':
            return self.message_tree.find_child(names[1:])
        else:
            return self.scope.parent.find_child(names)
    def enter_package(self, package):
        if not package:
            return self.message_tree
        names = package.split('.')
        pos = self.message_tree
        for i, name in enumerate(names):
            new_pos = pos.get_child(name)
            if new_pos:
                pos = new_pos
            else:
                return self._build_nodes(pos, names[i:])
        return pos
    def enter_file(self, filename, package):
        self.filename = filename
        self.package = package.split('.')
        self._init_field()
        self.scope = self.enter_package(package)
    def exit_file(self):
        self._init_field()
        self.filename = None
        self.package = []
        self.scope = self.scope.parent
    def enter(self, message_name):
        self.scope = TreeNode(message_name, self.scope, self.filename,
                            self.package)
    def exit(self):
        self.scope = self.scope.parent
    def _init_field(self):
        self.descriptor = []
        self.context = []
        self.message = []
        self.register = []
    def _build_nodes(self, node, names):
        parent = node
        for i in names:
            parent = TreeNode(i, parent, self.filename, self.package)
        return parent
class Writer(object):
    def __init__(self, prefix=None):
        self.io = StringIO()
        self.__indent = ''
        self.__prefix = prefix
    def getvalue(self):
        return self.io.getvalue()
    def __enter__(self):
        self.__indent += '    '
        return self
    def __exit__(self, type, value, trackback):
        self.__indent = self.__indent[:-4]
    def __call__(self, data):
        self.io.write(self.__indent)
        if self.__prefix:
            self.io.write(self.__prefix)
        self.io.write(data)
DEFAULT_VALUE = {
    FDP.TYPE_DOUBLE         : '0.0',
    FDP.TYPE_FLOAT          : '0.0',
    FDP.TYPE_INT64          : '0',
    FDP.TYPE_UINT64         : '0',
    FDP.TYPE_INT32          : '0',
    FDP.TYPE_FIXED64        : '0',
    FDP.TYPE_FIXED32        : '0',
    FDP.TYPE_BOOL           : 'false',
    FDP.TYPE_STRING         : '""',
    FDP.TYPE_MESSAGE        : 'nil',
    FDP.TYPE_BYTES          : '""',
    FDP.TYPE_UINT32         : '0',
    FDP.TYPE_ENUM           : '1',
    FDP.TYPE_SFIXED32       : '0',
    FDP.TYPE_SFIXED64       : '0',
    FDP.TYPE_SINT32         : '0',
    FDP.TYPE_SINT64         : '0',
}
def code_gen_enum_item(index, enum_value, env):
    full_name = env.get_local_name() + '.' + enum_value.name
    obj_name = full_name.upper().replace('.', '_') + '_ENUM'
    env.descriptor.append(
        "local %s = protobuf.EnumValueDescriptor();\n"% obj_name
    )
    context = Writer(obj_name)
    context('.name = "%s"\n' % enum_value.name)
    context('.index = %d\n' % index)
    context('.number = %d\n' % enum_value.number)
    env.context.append(context.getvalue())
    return obj_name
def code_gen_enum(enum_desc, env):
    env.enter(enum_desc.name)
    full_name = env.get_local_name()
    obj_name = full_name.upper().replace('.', '_')
    env.descriptor.append(
        "local %s = protobuf.EnumDescriptor();\n"% obj_name
    )
    context = Writer(obj_name)
    context('.name = "%s"\n' % enum_desc.name)
    context('.full_name = "%s"\n' % env.get_global_name())
    values = []
    for i, enum_value in enumerate(enum_desc.value):
        values.append(code_gen_enum_item(i, enum_value, env))
    context('.values = {%s}\n' % ','.join(values))
    env.context.append(context.getvalue())
    env.exit()
    return obj_name
def code_gen_field(index, field_desc, env):
    full_name = env.get_local_name() + '.' + field_desc.name
    obj_name = full_name.upper().replace('.', '_') + '_FIELD'
    env.descriptor.append(
        "local %s = protobuf.FieldDescriptor();\n"% obj_name
    )
    context = Writer(obj_name)
    context('.name = "%s"\n' % field_desc.name)
    context('.full_name = "%s"\n' % (
        env.get_global_name() + '.' + field_desc.name))
    context('.number = %d\n' % field_desc.number)
    context('.index = %d\n' % index)
    context('.label = %d\n' % field_desc.label)
    if field_desc.HasField("default_value"):
        context('.has_default_value = true\n')
        value = field_desc.default_value
        if field_desc.type == FDP.TYPE_STRING:
            context('.default_value = "%s"\n'%value)
        else:
            context('.default_value = %s\n'%value)
    else:
        context('.has_default_value = false\n')
        if field_desc.label == FDP.LABEL_REPEATED:
            default_value = "{}"
        elif field_desc.HasField('type_name'):
            default_value = "nil"
        else:
            default_value = DEFAULT_VALUE[field_desc.type]
        context('.default_value = %s\n' % default_value)
    if field_desc.HasField('type_name'):
        type_name = env.get_ref_name(field_desc.type_name).upper().replace('.', '_')
        if field_desc.type == FDP.TYPE_MESSAGE:
            context('.message_type = %s\n' % type_name)
        else:
            context('.enum_type = %s\n' % type_name)
    if field_desc.HasField('extendee'):
        type_name = env.get_ref_name(field_desc.extendee)
        env.register.append(
            "%s.RegisterExtension(%s)\n" % (type_name, obj_name)
        )
    context('.type = %d\n' % field_desc.type)
    context('.cpp_type = %d\n\n' % CPP_TYPE[field_desc.type])
    env.context.append(context.getvalue())
    return obj_name
def code_gen_message(message_descriptor, env, containing_type = None):
    env.enter(message_descriptor.name)
    full_name = env.get_local_name()
    obj_name = full_name.upper().replace('.', '_')
    env.descriptor.append(
        "local %s = protobuf.Descriptor();\n"% obj_name
    )
    context = Writer(obj_name)
    context('.name = "%s"\n' % message_descriptor.name)
    context('.full_name = "%s"\n' % env.get_global_name())
    nested_types = []
    for msg_desc in message_descriptor.nested_type:
        msg_name = code_gen_message(msg_desc, env, obj_name)
        nested_types.append(msg_name)
    context('.nested_types = {%s}\n' % ', '.join(nested_types))
    enums = []
    for enum_desc in message_descriptor.enum_type:
        enums.append(code_gen_enum(enum_desc, env))
    context('.enum_types = {%s}\n' % ', '.join(enums))
    fields = []
    for i, field_desc in enumerate(message_descriptor.field):
        fields.append(code_gen_field(i, field_desc, env))
    context('.fields = {%s}\n' % ', '.join(fields))
    if len(message_descriptor.extension_range) > 0:
        context('.is_extendable = true\n')
    else:
        context('.is_extendable = false\n')
    extensions = []
    for i, field_desc in enumerate(message_descriptor.extension):
        extensions.append(code_gen_field(i, field_desc, env))
    context('.extensions = {%s}\n' % ', '.join(extensions))
    if containing_type:
        context('.containing_type = %s\n' % containing_type)
    env.message.append('%s = protobuf.Message(%s)\n' % (full_name,
                                                        obj_name))
    env.context.append(context.getvalue())
    env.exit()
    # print('\n'+str(env.descriptor)+"\n")
    # print('\n'+str(env.context)+"\n")
    # print('\n'+str(env.message)+"\n")
    # print('\n'+str(env.register)+"\n")
    # print('\n'+str(env.filename)+"\n")
    return obj_name
def write_header(writer):
    writer("""-- Generated By protoc-gen-lua Do not Edit
""")
def code_gen_file(proto_file, env, is_gen):
    filename = path.splitext(proto_file.name)[0]
    env.enter_file(filename, proto_file.package)
    includes = []
    for f in proto_file.dependency:
        inc_file = path.splitext(f)[0]
        includes.append(inc_file)
#    for field_desc in proto_file.extension:
#        code_gen_extensions(field_desc, field_desc.name, env)
    for enum_desc in proto_file.enum_type:
        code_gen_enum(enum_desc, env)
        for enum_value in enum_desc.value:
            env.message.append('%s = %d\n' % (enum_value.name,
                                            enum_value.number))
    for msg_desc in proto_file.message_type:
        code_gen_message(msg_desc, env)
    if is_gen:
        lua = Writer()
        write_header(lua)
        lua('local protobuf = require "protobuf"\n')
        for i in includes:
            lua('local %s_pb = require("%s_pb")\n' % (i, i))
        lua("module('%s_pb')\n" % env.filename)
        lua('\n\n')
        for des in env.descriptor:
            lua(des)
        lua('\n')
        for con in env.context:
            lua(con)
        lua('\n')
        env.message.sort()
        for msg in env.message:
            lua(msg)
        lua('\n')
        for reg in env.register:
            lua(reg)
        # print('\n'+str(env.descriptor)+"\n")
        # print('\n'+str(env.context)+"\n")
        # print('\n'+str(env.message)+"\n")
        # print('\n'+str(env.register)+"\n")
        # print('\n'+str(env.filename)+"\n")
        # print('\n'+str(lua.getvalue())+"\n")
        _files[env.filename+ '_pb.lua'] = lua.getvalue()
    env.exit_file()
def main():
    print('-------------')
    plugin_require_bin = sys.stdin.buffer.read()
    # print(plugin_require_bin.decode("utf-8","ignore"))
    code_gen_req = plugin_pb2.CodeGeneratorRequest()
    code_gen_req.ParseFromString(plugin_require_bin)
    env = Env()
    for proto_file in code_gen_req.proto_file:
        code_gen_file(proto_file, env,
                proto_file.name in code_gen_req.file_to_generate)
    code_generated = plugin_pb2.CodeGeneratorResponse()
    for k in  _files:
        file_desc = code_generated.file.add()
        file_desc.name = k
        file_desc.content = _files[k]
    sys.stdout.buffer.write(code_generated.SerializeToString())
    current_directory = os.getcwd()
    for k in _files:
        try:
            # Construct the full path using the current directory
            file_path = os.path.join(current_directory, k)
            
            with open(file_path, "w+") as lua_file:
                lua_file.write(_files[k])
            # print("输出protocol-lua:", k, "正常")
        except Exception as e:
            # print("输出lua报错,请检查")
            print("xxxxxxxxxxxxxxxxxxxxxxxxxxxxx", str(e))
if __name__ == "__main__":
    main()
```

+ 注意第一行代码，#!/usr/local/openresty/lualib/luaenv/bin/python，需要你把这里改成你自己的python环境。然后把<font style="color:rgb(77, 77, 77);">protoc-gen-lua移动至/usr/local/bin 目录。</font>
+ 注意第一行代码，#!/usr/local/openresty/lualib/luaenv/bin/python，需要你把这里改成你自己的python环境。然后把<font style="color:rgb(77, 77, 77);">protoc-gen-lua移动至/usr/local/bin 目录。</font>
+ <font style="color:rgb(77, 77, 77);">现在你就可以执行protoc  --lua_out=. person.proto了。</font>
+ <font style="color:rgb(77, 77, 77);">完整版代码已上传至github：</font>[https://github.com/lionel-daydayup/protoc-gen-lua](https://github.com/lionel-daydayup/protoc-gen-lua)

###### <font style="color:rgb(77, 77, 77);">参考：</font>
[https://blog.csdn.net/morning_bird/article/details/77869038](https://blog.csdn.net/morning_bird/article/details/77869038)

<font style="color:rgb(77, 77, 77);"></font>

<font style="color:rgb(77, 77, 77);"></font>

<font style="color:rgb(77, 77, 77);"></font>

<font style="color:rgb(77, 77, 77);"></font>

