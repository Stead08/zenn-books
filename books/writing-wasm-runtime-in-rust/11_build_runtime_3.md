---
title: "Runtimeの実装 ~ 外部関数実行まで ~"
---

本章では前章までの実装を拡張して、外部関数を実行する機能を実装していく。
最終的には次のようにクロージャーを`Wasm Runtime`に登録することで外部関数を実行できるようになる。

```rust
runtime.add_import("env", "add", |_, args| {
    let arg = args[0];
    Ok(Some(arg + arg))
})?;
let args = vec![Value::I32(arg)];
let result = runtime.call("call_add", args)?;
assert_eq!(result, Some(Value::I32(want)));
```

## `Import Section`のデコード実装
[Wasmバイナリの構造](https://zenn.dev/skanehira/books/writing-wasm-runtime-in-rust/viewer/04_wasm_binary_structure#import-section)の章で以下のように説明をしたとおり、`Wasm Runtime`にはインポート機能がある。

> `Import Section`がモジュール外にあるメモリや関数などをインポートするための情報が定義されている領域である。
> モジュール外とは、他モジュールやRuntimeが用意したメモリや関数のことを指す。

本節では最終的に以下のWATをデコードできるようになる。
といっても`Export Section`の処理とほぼ同じなので、すんなり理解できると思う。

```wat:src/fixtures/import.wat
(module
  (func $add (import "env" "add") (param i32) (result i32))
  (func (export "call_add") (param i32) (result i32)
    (local.get 0)
    (call $add)
  )
)
```

バイナリ構造は次とおり。

```
; section "Import" (2)
0000010: 02                ; section code
0000011: 0b                ; section size
0000012: 01                ; num imports
; import header 0
0000013: 03                ; string length
0000014: 656e 76           ; import module name (env)
0000017: 03                ; string length
0000018: 6164 64           ; import field name (add)
000001b: 00                ; import kind
000001c: 00                ; import signature index
```

まず`Export`と同様に`Import`を追加する。

```diff:src/binary/types.rs
diff --git a/src/binary/types.rs b/src/binary/types.rs
index 191c34b..64912f8 100644
--- a/src/binary/types.rs
+++ b/src/binary/types.rs
@@ -36,3 +36,15 @@ pub struct Export {
     pub name: String,
     pub desc: ExportDesc,
 }
+
+#[derive(Debug, Clone, PartialEq, Eq)]
+pub enum ImportDesc {
+    Func(u32),
+}
+
+#[derive(Debug, PartialEq, Eq)]
+pub struct Import {
+    pub module: String,
+    pub field: String,
+    pub desc: ImportDesc,
+}
```

```diff:src/binary/module.rs
diff --git a/src/binary/module.rs b/src/binary/module.rs
index 5bf739d..dbcb30e 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -2,7 +2,7 @@ use super::{
     instruction::Instruction,
     opcode::Opcode,
     section::{Function, SectionCode},
-    types::{Export, ExportDesc, FuncType, FunctionLocal, ValueType},
+    types::{Export, ExportDesc, FuncType, FunctionLocal, Import, ValueType},
 };
 use nom::{
     bytes::complete::{tag, take},
@@ -22,6 +22,7 @@ pub struct Module {
     pub function_section: Option<Vec<u32>>,
     pub code_section: Option<Vec<Function>>,
     pub export_section: Option<Vec<Export>>,
+    pub import_section: Option<Vec<Import>>,
 }
 
 impl Default for Module {
@@ -33,6 +34,7 @@ impl Default for Module {
             function_section: None,
             code_section: None,
             export_section: None,
+            import_section: None,
         }
     }
 }
```

次に、文字列をデコードする部分を`decode_name(...)`に切り出す。
バイナリを見たらわかると思うが、モジュール名と関数名は文字列のため、複数回文字列をデコードする必要があるので使い回せるようにする必要がある。

```diff:src/binary/module.rs
diff --git a/src/binary/module.rs b/src/binary/module.rs
index dbcb30e..7430fbe 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -214,9 +214,7 @@ fn decode_export_section(input: &[u8]) -> IResult<&[u8], Vec<Export>> {
     let mut exports = vec![];
 
     for _ in 0..count {
-        let (rest, name_len) = leb128_u32(input)?;
-        let (rest, name_bytes) = take(name_len)(rest)?;
-        let name = String::from_utf8(name_bytes.to_vec()).expect("invalid utf-8 string");
+        let (rest, name) = decode_name(input)?;
         let (rest, export_kind) = le_u8(rest)?;
         let (rest, idx) = leb128_u32(rest)?;
         let desc = match export_kind {
@@ -230,6 +228,15 @@ fn decode_export_section(input: &[u8]) -> IResult<&[u8], Vec<Export>> {
     Ok((input, exports))
 }
 
+fn decode_name(input: &[u8]) -> IResult<&[u8], String> {
+    let (input, size) = leb128_u32(input)?;
+    let (input, name) = take(size)(input)?;
+    Ok((
+        input,
+        String::from_utf8(name.to_vec()).expect("invalid utf-8 string"),
+    ))
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::{
```

続けて、デコード処理を実装する。
やっていることは`Import Section`のデコードとほぼ同じである。

```diff:src/binary/module.rs
diff --git a/src/binary/module.rs b/src/binary/module.rs
index 7430fbe..e3253de 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -2,7 +2,7 @@ use super::{
     instruction::Instruction,
     opcode::Opcode,
     section::{Function, SectionCode},
-    types::{Export, ExportDesc, FuncType, FunctionLocal, Import, ValueType},
+    types::{Export, ExportDesc, FuncType, FunctionLocal, Import, ImportDesc, ValueType},
 };
 use nom::{
     bytes::complete::{tag, take},
@@ -83,6 +83,10 @@ impl Module {
                             let (_, exports) = decode_export_section(section_contents)?;
                             module.export_section = Some(exports);
                         }
+                        SectionCode::Import => {
+                            let (_, imports) = decode_import_section(section_contents)?;
+                            module.import_section = Some(imports);
+                        }
                         _ => todo!(),
                     };
 
@@ -228,6 +232,34 @@ fn decode_export_section(input: &[u8]) -> IResult<&[u8], Vec<Export>> {
     Ok((input, exports))
 }
 
+fn decode_import_section(input: &[u8]) -> IResult<&[u8], Vec<Import>> {
+    let (mut input, count) = leb128_u32(input)?;
+    let mut imports = vec![];
+
+    for _ in 0..count {
+        let (rest, module) = decode_name(input)?;
+        let (rest, field) = decode_name(rest)?;
+        let (rest, import_kind) = le_u8(rest)?;
+        let (rest, desc) = match import_kind {
+            0x00 => {
+                let (rest, idx) = leb128_u32(rest)?;
+                (rest, ImportDesc::Func(idx))
+            }
+            _ => unimplemented!("unsupported import kind: {:X}", import_kind),
+        };
+
+        imports.push(Import {
+            module,
+            field,
+            desc,
+        });
+
+        input = rest;
+    }
+
+    Ok((&[], imports))
+}
+
 fn decode_name(input: &[u8]) -> IResult<&[u8], String> {
     let (input, size) = leb128_u32(input)?;
     let (input, name) = take(size)(input)?;
```

続けてテストコードを追加して、デコードの実装が問題ないことを確認する。

```diff:src/binary/module.rs
diff --git a/src/binary/module.rs b/src/binary/module.rs
index e3253de..2a8dbfa 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -275,7 +275,7 @@ mod tests {
         instruction::Instruction,
         module::Module,
         section::Function,
-        types::{Export, ExportDesc, FuncType, FunctionLocal, ValueType},
+        types::{Export, ExportDesc, FuncType, FunctionLocal, Import, ImportDesc, ValueType},
     };
     use anyhow::Result;
 
@@ -427,4 +427,39 @@ mod tests {
         );
         Ok(())
     }
+
+    #[test]
+    fn decode_import() -> Result<()> {
+        let wasm = wat::parse_file("src/fixtures/import.wat")?;
+        let module = Module::new(&wasm)?;
+        assert_eq!(
+            module,
+            Module {
+                type_section: Some(vec![FuncType {
+                    params: vec![ValueType::I32],
+                    results: vec![ValueType::I32],
+                }]),
+                import_section: Some(vec![Import {
+                    module: "env".into(),
+                    field: "add".into(),
+                    desc: ImportDesc::Func(0),
+                }]),
+                export_section: Some(vec![Export {
+                    name: "call_add".into(),
+                    desc: ExportDesc::Func(1),
+                }]),
+                function_section: Some(vec![0]),
+                code_section: Some(vec![Function {
+                    locals: vec![],
+                    code: vec![
+                        Instruction::LocalGet(0),
+                        Instruction::Call(0),
+                        Instruction::End
+                    ],
+                }]),
+                ..Default::default()
+            }
+        );
+        Ok(())
+    }
 }
```

```sh
running 10 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_func_call ... ok
test binary::module::tests::decode_func_add ... ok
test execution::runtime::tests::not_found_export_function ... ok
test binary::module::tests::decode_import ... ok
test execution::runtime::tests::func_call ... ok
test execution::runtime::tests::execute_i32_add ... ok
```

## 外部関数実行の実装

内部関数と同様、外部関数を実行する際に必要になる情報（モジュール名、関数名、関数シグネチャ）を持った`ExternalFuncInst`を用意する。

```diff:src/execution/store.rs
diff --git a/src/execution/store.rs b/src/execution/store.rs
index 3f6ecb2..922f2b1 100644
--- a/src/execution/store.rs
+++ b/src/execution/store.rs
@@ -19,8 +19,17 @@ pub struct InternalFuncInst {
     pub code: Func,
 }
 
+#[derive(Debug, Clone)]
+pub struct ExternalFuncInst {
+    pub module: String,
+    pub func: String,
+    pub func_type: FuncType,
+}
+
+#[derive(Clone)]
 pub enum FuncInst {
     Internal(InternalFuncInst),
+    External(ExternalFuncInst),
 }
 
 pub struct ExportInst {
```

内部関数と違って、外部関数はインポートしたものなので、実体Wasmバイナリ外にある。
そのため、Wasmバイナリからみればなにかの関数を呼び出すだけということになる。

これをRustに例えると次のようなことをやっていることになる。

```rust
extern "C"  {
    fn double(x: i32) -> i32;
}

fn main() {
    unsafe { double(10) };
}
```

デコード処理は`Code Section`とほぼ同じことをやっていて、モジュール名と関数名、シグネチャ情報を取得している。

```diff:src/execution/store.rs
diff --git a/src/execution/store.rs b/src/execution/store.rs
index 922f2b1..5666a39 100644
--- a/src/execution/store.rs
+++ b/src/execution/store.rs
@@ -3,7 +3,7 @@ use std::collections::HashMap;
 use crate::binary::{
     instruction::Instruction,
     module::Module,
-    types::{ExportDesc, FuncType, ValueType},
+    types::{ExportDesc, FuncType, ImportDesc, ValueType},
 };
 use anyhow::{bail, Result};
 
@@ -57,6 +57,33 @@ impl Store {
 
         let mut funcs = vec![];
 
+        if let Some(ref import_section) = module.import_section {
+            for import in import_section {
+                let module_name = import.module.clone();
+                let field = import.field.clone();
+                let func_type = match import.desc {
+                    ImportDesc::Func(type_idx) => {
+                        let Some(ref func_types) = module.type_section else {
+                            bail!("not found type_section")
+                        };
+
+                        let Some(func_type) = func_types.get(type_idx as usize) else {
+                            bail!("not found func type in type_section")
+                        };
+
+                        func_type.clone()
+                    }
+                };
+
+                let func = FuncInst::External(ExternalFuncInst {
+                    module: module_name,
+                    func: field,
+                    func_type,
+                });
+                funcs.push(func);
+            }
+        }
+
         if let Some(ref code_section) = module.code_section {
             for (func_body, type_idx) in code_section.iter().zip(func_type_idxs.into_iter()) {
                 let Some(ref func_types) = module.type_section else {
```

続けて、外部関数をRuntimeに登録してWasmから実行できるようにしていく。
まず、モジュール名と関数名で実行する関数を逆引きできるデータ構造の型エイリアスを定義する。

```rust:src/execution/import.rs
use anyhow::Result;
use std::collections::HashMap;

use super::{store::Store, value::Value};

pub type ImportFunc = Box<dyn FnMut(&mut Store, Vec<Value>) -> Result<Option<Value>>>;
pub type Import = HashMap<String, HashMap<String, ImportFunc>>;
```

```diff:src/execution.rs
diff --git a/src/execution.rs b/src/execution.rs
index acbafa4..5d6aec6 100644
--- a/src/execution.rs
+++ b/src/execution.rs
@@ -1,3 +1,4 @@
+pub mod import;
 pub mod runtime;
 pub mod store;
 pub mod value;
```

`ImportFunc`は登録する関数を表現していて、クロージャーとなっている。
次章でメモリを操作する処理を実装するため、`&mut Store`を受け取るインターフェイスにしている。
`Import`は実際に逆引きする際に使うデータ構造の型となっている。

続けて、インポート情報を`Wasm Runtime`で持てるように`import`フィールドを追加する。

```diff:src/execution/runtime.rs
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 5e2772f..cf7bcb9 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -1,5 +1,6 @@
 use super::{
-    store::{FuncInst, InternalFuncInst, Store},
+    import::Import,
+    store::{ExternalFuncInst, FuncInst, InternalFuncInst, Store},
     value::Value,
 };
 use crate::binary::{
@@ -23,6 +24,7 @@ pub struct Runtime {
     pub store: Store,
     pub stack: Vec<Value>,
     pub call_stack: Vec<Frame>,
+    pub import: Import,
 }
 
 impl Runtime {
```

これでインポート情報を元に関数を逆引きできるようになったので、外部関数を実行する処理を次のように実装する。
`invoke_external(...)`は外部関数を実行する関数で、やっていることはモジュール名と関数名で逆引きして得たクロージャーを呼び出しているだけ。

```diff:src/execution/runtime.rs
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index cf7bcb9..bd1a1e5 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -56,6 +56,7 @@ impl Runtime {
         }
         match func_inst {
             FuncInst::Internal(func) => self.invoke_internal(func.clone()),
+            FuncInst::External(func) => self.invoke_external(func.clone()),
         }
     }
 
@@ -102,6 +103,20 @@ impl Runtime {
         Ok(None)
     }
 
+    fn invoke_external(&mut self, func: ExternalFuncInst) -> Result<Option<Value>> {
+        let args = self
+            .stack
+            .split_off(self.stack.len() - func.func_type.params.len());
+        let module = self
+            .import
+            .get_mut(&func.module)
+            .ok_or(anyhow!("not found module"))?;
+        let import_func = module
+            .get_mut(&func.func)
+            .ok_or(anyhow!("not found function"))?;
+        import_func(&mut self.store, args)
+    }
+
     fn execute(&mut self) -> Result<()> {
         loop {
             let Some(frame) = self.call_stack.last_mut() else {
@@ -139,8 +154,14 @@ impl Runtime {
                     let Some(func) = self.store.funcs.get(*idx as usize) else {
                         bail!("not found func");
                     };
-                    match func {
-                        FuncInst::Internal(func) => self.push_frame(&func.clone()),
+                    let func_inst = func.clone();
+                    match func_inst {
+                        FuncInst::Internal(func) => self.push_frame(&func),
+                        FuncInst::External(func) => {
+                            if let Some(value) = self.invoke_external(func)? {
+                                self.stack.push(value);
+                            }
+                        }
                     }
                 }
             }
```

これで外部関数を実行できるようになったので、次に外部関数を登録する関数を`Runtime`に追加する。

```diff:src/execution/runtime.rs
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index bd1a1e5..41e50c6 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -37,6 +37,17 @@ impl Runtime {
         })
     }
 
+    pub fn add_import(
+        &mut self,
+        module_name: impl Into<String>,
+        func_name: impl Into<String>,
+        func: impl FnMut(&mut Store, Vec<Value>) -> Result<Option<Value>> + 'static,
+    ) -> Result<()> {
+        let import = self.import.entry(module_name.into()).or_default();
+        import.insert(func_name.into(), Box::new(func));
+        Ok(())
+    }
+
     pub fn call(&mut self, name: impl Into<String>, args: Vec<Value>) -> Result<Option<Value>> {
         let idx = match self
             .store
```

最後にテストを追加して、問題なく動作することを確認する。

```diff:src/execution/runtime.rs
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 41e50c6..437cef3 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -241,4 +241,22 @@ mod tests {
         }
         Ok(())
     }
+
+    #[test]
+    fn call_imported_func() -> Result<()> {
+        let wasm = wat::parse_file("src/fixtures/import.wat")?;
+        let mut runtime = Runtime::instantiate(wasm)?;
+        runtime.add_import("env", "add", |_, args| {
+            let arg = args[0];
+            Ok(Some(arg + arg))
+        })?;
+        let tests = vec![(2, 4), (10, 20), (1, 2)];
+
+        for (arg, want) in tests {
+            let args = vec![Value::I32(arg)];
+            let result = runtime.call("call_add", args)?;
+            assert_eq!(result, Some(Value::I32(want)));
+        }
+        Ok(())
+    }
 }
```

```sh
running 11 tests
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_func_add ... ok
test binary::module::tests::decode_func_call ... ok
test binary::module::tests::decode_import ... ok
test binary::module::tests::decode_simplest_func ... ok
test execution::runtime::tests::execute_i32_add ... ok
test execution::runtime::tests::call_imported_func ... ok
test execution::runtime::tests::func_call ... ok
test execution::runtime::tests::not_found_export_function ... ok
```

外部関数が見つからない場合はちゃんとエラーになることも合わせ確認する。

```diff:src/execution/runtime.rs
diff --git a/src/execution/runtime.rs b/src/execution/runtime.rs
index 437cef3..3c492bc 100644
--- a/src/execution/runtime.rs
+++ b/src/execution/runtime.rs
@@ -259,4 +259,14 @@ mod tests {
         }
         Ok(())
     }
+
+    #[test]
+    fn not_found_imported_func() -> Result<()> {
+        let wasm = wat::parse_file("src/fixtures/import.wat")?;
+        let mut runtime = Runtime::instantiate(wasm)?;
+        runtime.add_import("env", "fooooo", |_, _| Ok(None))?;
+        let result = runtime.call("call_add", vec![Value::I32(1)]);
+        assert!(result.is_err());
+        Ok(())
+    }
 }
```

```sh
running 1 test
test execution::runtime::tests::not_found_imported_func ... ok
```

## まとめ
本章では外部関数を実行できるところまで実装した。
徐々に`Wasm Runtime`が形になってきた。

実は外部関数を実行できる時点で`"Hello, World"`を出力できるようになるが、
本書はちゃんと`WASI`を実装して出力する。
