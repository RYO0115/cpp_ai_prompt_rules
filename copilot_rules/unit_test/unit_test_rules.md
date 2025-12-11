# 単体テスト作成のルール

## 役割

- あなたはC++と自動運転システム, Google Testのエキスパートです。
- 丁寧な口調で以下のルールに沿って仕様書の作成をしてください
- 言語は日本語で作成してください
- もしわからないことがあれば作業を止めて質問してください


## 目的

- 指定したC++関数/クラスに対する単体テストを作成してください
    - クラス内のすべての関数に対し単体テストを作成してください
    - テストは以下のパターンを可能な限り網羅する広範なテストシナリオを作成してください
      - 正常系
      - 異常系
      - 境界値(エッジケース)
      - エラーハンドリング
    - 各テストケースにはテストの意図、期待する動作を説明するコメントを記述してください
    - すでに単体テストが部分的に書かれている場合は不足しているテストを追加してください。


## 出力フォーマット

- 生成するコードはプロジェクトのコーディング標準に従ってください
- テストファイル名は[元のファイル名]_test.cppとし、命名規則に従ってください
- テストケース名はテスト内容がわかるユニークな名前を設定してください

## 出力先

- include, srcなどと同じ場所にtestディレクトリが無ければ作成してください
- testディレクトリに元のファイル名と同じディレクトリが無ければディレクトリを作ってください。
- 作成したテストコードは test/元ファイル名/ に保存してください

## CMakeLists.txt

### CMakeLists.txtが無い場合

以下を参考に作成してください。

'''
# Cmake version
cmake_minimum_required(VERSION 3.0)

# Project Name and Language
#project(byte_control_test CXX)

set(CMAKE_CXX_FLAGS "-g -O3 -Wall -pthread -std=c++14 -Wextra -Wpedantic -Wshadow -Wnull-dereference -Wconversion")


#-----------------Below here for Unit Test ----------------------#

# To Use GoogleTest
find_package(GTest REQUIRED)
add_executable(byte_control_test byte_control_test.cpp)
target_link_libraries( byte_control_test
	#GoogleTest Libarary
	GTest::GTest
	GTest::Main
	#Below other needed library
	byte_control
)


if(CMAKE_BUILD_TYPE STREQUAL "Coverage")

set(test_name byte_control_test)
target_compile_options( ${test_name} PRIVATE --coverage)
target_link_libraries( ${test_name} --coverage)
add_test(NAME ${test_name} COMMAND ${test_name})


endif()

'''

### すでにCMakeLists.txtがある場合

今回追加するテストファイルがCMakeLists.txtに追加されているかを確認し、無ければ追加してください。

## private関数・private変数のテスト方法

### 基本方針
- **本番コード（ヘッダーファイル・実装ファイル）は一切変更しない**
- **スタンダードライブラリの挙動を変えずに実装する**
- **循環依存やコンパイル時間の増大を避ける**
- **テスト専用のアクセス機能を分離したファイルで実装する**

### ファイル命名規則
- テスト対象クラスが `class_name.hpp` の場合：
  - `test/class_name_private_access.hpp` （アクセス関数宣言）
  - `test/class_name_private_access.cpp` （アクセス関数実装）
  - `test/class_name_test.cpp` （メインテストファイル）

### private_access.hpp の作成ルール

```cpp
#ifndef [CLASS_NAME_UPPER]_PRIVATE_ACCESS_HPP
#define [CLASS_NAME_UPPER]_PRIVATE_ACCESS_HPP

#include "[original_header_file].hpp"

// テスト専用のアクセス関数宣言
namespace PrivateAccess {
    // private変数のgetter関数群
    [return_type] get[VariableName](const [ClassName]& obj);
    
    // private変数のsetter関数群  
    void set[VariableName]([ClassName]& obj, [param_type] value);
    
    // privateメソッドの呼び出し関数群
    [return_type] call[MethodName]([ClassName]& obj[, parameters]);
}

#endif
```

### private_access.cpp の作成ルール

```cpp
// 標準ライブラリやテスト対象外のライブラリを先に読み込む
#include <iostream>
#include <string>
#include <vector>
#include <fstream>
#include <sstream>
// その他必要な標準ライブラリ...

// プロジェクト固有の依存ライブラリ
#include "formula_tools.hpp"
// その他必要なプロジェクトライブラリ...

// この時点でprivateアクセスを有効化
#define private public
#include "[original_header_file].hpp"
#undef private

#include "[class_name]_private_access.hpp"

namespace PrivateAccess {
    // private変数へのアクセス実装
    [return_type] get[VariableName](const [ClassName]& obj) {
        return obj.[private_variable_name];
    }
    
    void set[VariableName]([ClassName]& obj, [param_type] value) {
        obj.[private_variable_name] = value;
    }
    
    // privateメソッドの呼び出し実装
    [return_type] call[MethodName]([ClassName]& obj[, parameters]) {
        return obj.[private_method_name]([parameters]);
    }
}
```

### テストファイルでの使用ルール

```cpp
#include <gtest/gtest.h>
#include "[original_header_file].hpp"
#include "[class_name]_private_access.hpp"  // テスト専用アクセス

class [ClassName]Test : public ::testing::Test {
protected:
    [ClassName] target_object_;
    
    void SetUp() override {
        // 必要に応じてprivate変数を初期化
        PrivateAccess::set[VariableName](target_object_, initial_value);
    }
};

TEST_F([ClassName]Test, [PrivateMethodName]Test) {
    // privateメソッドを直接テスト
    [return_type] result = PrivateAccess::call[MethodName](target_object_[, args]);
    
    // private変数の状態を検証
    EXPECT_EQ(PrivateAccess::get[VariableName](target_object_), expected_value);
    
    // 戻り値の検証
    EXPECT_EQ(result, expected_result);
}
```

### CMakeLists.txt での設定

```cmake
if(${GTEST_RUN})
    add_executable([class_name]_test 
        test/[class_name]_test.cpp
        test/[class_name]_private_access.cpp  # private access実装を追加
    )
    
    target_link_libraries([class_name]_test
        GTest::GTest
        GTest::Main
        [target_library_name]  # 本体ライブラリ
    )
endif()
```

### 実装時の注意事項

1. **アクセス関数の命名規則**
   - getter: `get[PascalCase変数名]`
   - setter: `set[PascalCase変数名]` 
   - メソッド呼び出し: `call[PascalCaseメソッド名]`

2. **型の完全性**
   - 戻り値の型は元のprivateメンバーと完全に一致させる
   - const correctnessを維持する（const参照で受け取るgetterなど）

3. **エラーハンドリング**
   - privateメソッドが例外を投げる可能性がある場合は、テストでEXPECT_THROWを使用

4. **複雑なprivateメソッドのテスト**
   - 引数が多い場合は、構造体やヘルパー関数で整理する
   - privateメソッドが他のprivateメソッドを呼ぶ場合は、依存関係を考慮してテスト順序を決める

5. **ライブラリのinclude順序**
   - **標準ライブラリやテスト対象外のライブラリを必ず`#define private public`より前にincludeする**
   - これにより、標準ライブラリ内のprivateキーワードが置換されることを防ぐ
   - includeの順序：①標準ライブラリ → ②プロジェクト依存ライブラリ → ③privateアクセス定義 → ④対象ヘッダー → ⑤アクセスヘッダー

### 禁止事項

- **本番のヘッダーファイル・実装ファイルにfriend宣言やテスト用コードを追加しない**
- **#ifdef UNIT_TESTなどの条件コンパイルを本番コードに追加しない**  
- **テストのためにprivateをprotectedに変更しない**
- **テスト用のpublicアクセサメソッドを本番クラスに追加しない**

この方法により、本番コードの品質と保守性を維持しながら、private関数の包括的なテストが可能になります。