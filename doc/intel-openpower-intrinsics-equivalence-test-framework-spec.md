# Intel Intrinsics / OpenPOWER移植 実装等価性テストフレームワーク仕様書

- 文書ID: IOITF-SPEC-001
- バージョン: 1.0.0-draft
- ステータス: 実装前仕様
- 対象読者: SIMD移植担当者、テスト実装者、CI管理者、レビュー担当者

## 1. 目的

本フレームワークは、Intel x86 Intrinsicsで実装された関数（以下「Intel実装」）と、それをOpenPOWER向けのVMX/VSX Intrinsicsへ移植した関数（以下「POWER実装」）へ、同一の入力を与えた際の観測可能な結果が、定義された比較規則のもとで等価であることを自動検証する。

「同一の入力」とは、各ホストのネイティブなベクトルレジスタ表現を共有することではない。アーキテクチャ非依存の形式で直列化した同一テストベクトルを、x86_64ホストとppc64leホストが読み込み、それぞれの実装へ変換して実行することを指す。

Intel実装の結果は移植元基準（differential reference）として扱う。ただし、Intel実装自体の正しさを保証する数学的オラクルではない。

## 2. 適用範囲

### 2.1 初期サポート対象

- OS: Linux
- Intel側アーキテクチャ: x86_64
- POWER側アーキテクチャ: ppc64le
- Intel ISA: SSE2、SSSE3、SSE4.1、AVX、AVX2を優先対象とする
- POWER ISA: POWER8以降のVMX/VSXを基本対象とする
- 言語: CまたはC++で記述されたIntrinsicsラッパー
- コンパイラ:
  - x86_64: GCC、Clang。Intel oneAPI DPC++/C++ Compilerは任意追加対象
  - ppc64le: GCC、Clang、IBM Open XL C/C++のうちプロジェクトで採用するもの
- 戻り値:
  - 整数・浮動小数点スカラー
  - 64/128/256ビットのベクトル値
  - マスク値
  - 明示された出力バッファ

### 2.2 初期対象外

- AVX-512の512ビット幅を単一POWERベクトル命令へ一対一対応させること
- AMX、MPX、SGXなど、VMX/VSXの演算移植として扱えない機能
- CPUID、時刻、乱数、キャッシュ制御、メモリフェンスなど、戻り値以外にプラットフォーム固有の意味を持つIntrinsic
- OS状態やスレッド間タイミングへ依存する処理
- 未定義動作または仕様上結果不定となる入力の一致
- 性能の同等性。性能測定は別仕様とする
- ppc64 big-endian。将来拡張として扱う

## 3. 用語

| 用語 | 定義 |
|---|---|
| Intrinsic | コンパイラがC/C++の関数または組み込み関数として公開する、ISA機能への型付きインターフェース。1個のIntrinsicが必ず1個の機械命令へ変換されるとは限らない。 |
| SIMD | Single Instruction, Multiple Data。1回の演算を複数の論理レーンへ適用する処理方式。本仕様のレーン番号は14節の正規レーン順に従う。 |
| ISA | Instruction Set Architecture。CPUが実行できる命令と、レジスタや例外などの動作を規定するアーキテクチャ仕様。ケース定義の`required_isa`は、そのケースの実行に必要なISA機能を表す。 |
| ABI | Application Binary Interface。呼び出し規約、データ配置、型の受け渡しなどのバイナリ境界。本仕様で「共通ABI」と記す場合は6節のアーキテクチャ非依存境界を指す。 |
| SSE2 / SSSE3 / SSE4.1 | x86のSIMD命令セット拡張。SSE2はStreaming SIMD Extensions 2であり、6.2節の`_mm_add_pd`と`_mm_set1_pd`の移植例が属する。 |
| AVX / AVX2 | Advanced Vector ExtensionsおよびAdvanced Vector Extensions 2。初期対象では256ビット値をPOWER側の2本の128ビットチャンクへ写像する。 |
| VMX（AltiVec） | Vector Multimedia Extension。POWER ISAの128ビットSIMD機能で、コンパイラ文書ではAltiVecとも呼ばれる。 |
| VSX | Vector Scalar Extension。POWER ISAでベクトルおよびスカラー浮動小数点処理を提供・拡張する機能。 |
| x86_64 | 64ビットx86アーキテクチャ。本仕様ではIntel実装を実行するホストの`environment.architecture`値でもある。 |
| ppc64le | little-endianの64ビットPowerPCアーキテクチャ。本仕様ではPOWER実装を実行するホストの`environment.architecture`値でもある。 |
| GCC | GNU Compiler Collection。本仕様の両アーキテクチャで選択可能なコンパイラの1つ。 |
| IBM Open XL C/C++ | IBMが提供するC/C++コンパイラ製品。本仕様で`XL`とだけ省略せず、この製品名を用いる。 |
| JSONL | JSON Lines。1行に1個のJSON値を格納する形式。本仕様では8.1節に従い、空行なし、各行1オブジェクト、LF終端とする。 |
| RFC 8785 / JCS | RFC 8785で規定されるJSON Canonicalization Scheme。入力IDや成果物ハッシュの対象を一意なUTF-8バイト列へ直列化するために使用する。 |
| SHA-256 | SHA-2ファミリーの256ビットハッシュ関数。本仕様では入力・ケース定義・結果成果物の同一性と破損を検出するために使用し、作成者の認証には使用しない。 |
| IEEE 754 | 浮動小数点の表現、丸め、特殊値、例外を規定する標準。本仕様の`f32`と`f64`の生ビット解釈はこれに従う。 |
| ULP | Unit in the Last Place。本仕様の`ulp`比較では、一般的な10進誤差ではなく13.2節で定義した順序キー間の整数距離を指す。 |
| MXCSR | x86のSSE浮動小数点制御・状態レジスタ。丸め、例外状態、FTZ、DAZの取得または設定に用いる。 |
| FPSCR | POWERのFloating-Point Status and Control Register。丸め、浮動小数点例外状態などの取得または設定に用いる。 |
| VSCR | POWERのVector Status and Control Register。結果マニフェストでは非IEEE動作に関係する`vscr_nj`を明示的に記録する。 |
| FTZ / DAZ | x86のFlush To ZeroおよびDenormals Are Zero。FTZはsubnormalとなる演算結果をゼロにし、DAZはsubnormal入力をゼロとして扱う制御である。 |
| テスト対象（SUT） | Intel実装またはPOWER実装のラッパー関数 |
| ケース定義 | 対応する両実装、型、ISA要件、入力制約、比較規則を宣言したメタデータ |
| テストベクトル | 1回の呼び出しに必要な入力値、即値、実行環境を直列化したもの |
| ランナー | テストベクトルを読み、ネイティブ実装を呼び出し、結果を出力する実行ファイル |
| コーディネーター | 入力生成、成果物検証、Intel結果とPOWER結果の比較、レポート生成を行うツール |
| 正規値形式 | CPUのエンディアンやベクトルABIに依存しない論理レーン表現 |
| 観測結果 | 戻り値、出力バッファ、およびケース定義の`environment.observe_fp_exceptions`が真の場合の正規化済み浮動小数点例外を含む比較対象 |

本文ではISA名とレジスタ名を表の大文字表記で記す。JSON/YAMLおよびコマンドラインの識別子は大文字・小文字を区別し、各スキーマまたは例で規定した綴りをそのまま使用しなければならない（MUST）。例えば、本文のSSE2とVSXは`required_isa`および`available_isa`ではそれぞれ`sse2`と`vsx`である。実装は識別子を大文字・小文字を区別せずに照合してはならない（MUST NOT）。

## 4. 全体アーキテクチャ

フレームワークは、次の4要素で構成しなければならない（MUST）。

1. ケースレジストリ
   - Intrinsicごとのシグネチャ、移植対応、入力制約、比較規則を保持する。
2. テストベクトル生成器
   - 決定的な境界値・構造化パターン・疑似乱数入力を生成する。
3. アーキテクチャ別ランナー
   - 同一形式の入力をIntel実装またはPOWER実装へ渡し、同一形式の結果を出力する。
4. 結果比較器
   - 入力IDで結果を突合し、ケース定義の比較規則に従って合否を判定する。

標準実行フローは次のとおりとする。

```text
ケース定義
    │
    ▼
入力生成 ── test-vectors.jsonl + input manifest
    │                         │
    ├──────────────┐          │ 同一ファイル
    ▼              ▼          │
x86_64 runner   ppc64le runner
    │              │
    ▼              ▼
intel-results   power-results
    └──────┬───────┘
           ▼
        comparator
           │
           ├─ summary.json
           ├─ junit.xml
           └─ failures/*.json
```

一方のアーキテクチャ上で他方の実装を再コンパイルしてはならない（MUST NOT）。各ランナーは対象ISAをネイティブに実行する。QEMU等のエミュレーターは開発時の補助には使用できる（MAY）が、リリース判定の唯一の根拠にはしてはならない（MUST NOT）。

## 5. 推奨ディレクトリ構成

```text
intrinsics-equivalence/
├── CMakeLists.txt
├── cases/
│   ├── integer.yaml
│   ├── floating_point.yaml
│   └── shuffle.yaml
├── include/framework/
│   ├── case_abi.h
│   ├── canonical_value.h
│   └── registry.h
├── adapters/
│   ├── intel/
│   └── openpower/
├── src/
│   ├── generator/
│   ├── runner/
│   └── comparator/
├── tests/
│   ├── framework/
│   └── fixtures/
└── artifacts/
```

`adapters/intel`と`adapters/openpower`は同じ論理ケースIDを公開しなければならない（MUST）。Intrinsics固有の型を共通ABI境界へ公開してはならない（MUST NOT）。

## 6. 共通ケースABI

### 6.1 ABI境界

共通ABIは、固定幅整数、バイト列、長さ、ステータスのみで構成する。`__m128i`、`__m256`、`vector unsigned int`などのネイティブベクトル型をABI境界に含めてはならない（MUST NOT）。

概念上のC ABIを次に示す。

```c
typedef struct {
    const uint8_t* data;
    size_t size;
} ioitf_bytes;

typedef struct {
    ioitf_bytes encoded_arguments;
    uint32_t rounding_mode;
    uint32_t fp_mode;
} ioitf_input;

typedef struct {
    uint8_t* encoded_return_value;
    size_t return_capacity;
    size_t return_size;
    uint8_t* encoded_memory_effects;
    size_t effects_capacity;
    size_t effects_size;
    uint32_t normalized_fp_exceptions;
    uint32_t status;
} ioitf_output;

int ioitf_run_<case_id>(const ioitf_input*, ioitf_output*);
```

各アダプターは、正規形式からネイティブベクトルを構築し、Intrinsic実装を呼び、結果を正規形式へ戻す。単純な`memcpy`だけで論理レーン順を決めてはならない（MUST NOT）。

### 6.2 移植ラッパーとケースアダプターの分離

Intel Intrinsic名をPOWER上でも公開する互換層をテストする場合、実装を次の3層へ分離しなければならない（MUST）。

1. **移植ラッパー**: アプリケーションへIntel互換APIを公開する実際のテスト対象。POWER側ではVMX/VSX Intrinsicsまたはコンパイラのベクトル演算で実装する。
2. **ケースアダプター**: 6.1節の共通ABIと移植ラッパーのネイティブ型を接続する。Intel側とPOWER側で別のケースシンボルを公開する。
3. **正規値変換**: 8.3節の生ビット列とネイティブベクトルを、14節の論理レーン規則に従って相互変換する。

ケースアダプターは、入力検証と正規値変換を終えた後に、登録されたテスト対象を1回呼び出し、その戻り値と宣言された副作用だけを符号化しなければならない（MUST）。テスト対象と同じ演算をスカラー式や別のPOWER Intrinsicでケースアダプター内へ再実装してはならない（MUST NOT）。この分離により、テスト用の再実装同士が一致して移植ラッパーの欠陥を見逃すことを防ぐ。

#### POWER互換層のC言語例

次は[POWER移植ラッパーの紹介例](https://qiita.com/daisukeokaoss/items/c0b94eb86a42c28dcce8)と同じ構造をGNU Cで示した、POWER専用互換ヘッダーの抜粋である。`__m128d`と`__v2df`の型定義は、プロジェクトが採用した互換ヘッダーがこの抜粋より前に提供するものとする。x86_64用の`<emmintrin.h>`と同じ翻訳単位へこの定義を入れてはならない（MUST NOT）。

```c
/* project_power_sse2_compat.h: ppc64le向けビルドだけで使用する。 */
#define IOITF_POWER_SSE2_COMPAT 1

extern __inline __m128d
__attribute__((__gnu_inline__, __always_inline__, __artificial__))
_mm_add_pd(__m128d a, __m128d b)
{
    return (__m128d)((__v2df)a + (__v2df)b);
}

extern __inline __m128d
__attribute__((__gnu_inline__, __always_inline__, __artificial__))
_mm_set1_pd(double value)
{
    return __extension__ (__m128d){value, value};
}
```

`_mm_add_pd`は2本のbinary64論理レーンを独立に加算する。正規形式のレーン番号を`j`とすると、丸め後の結果は`r[j] = a[j] + b[j]`（`j = 0, 1`）であり、レーン間の加算ではない。`_mm_set1_pd`は1個の`double`を複製し、`r[0] = value`かつ`r[1] = value`とする。`__gnu_inline__`、`__always_inline__`、`__artificial__`はインライン化とデバッグ表示を制御するGNU属性であり、演算の等価性を定義するものではない。別コンパイラでは同等の属性または`static inline`へ置換できる（MAY）が、ケース定義に採用コンパイラと必要ISAを記録しなければならない（MUST）。

ケースアダプターは次のように、ホストごとに異なる翻訳単位で実際のIntrinsicまたは移植ラッパーを呼ぶ。ここに示す関数はネイティブ型を使うため、6.1節の共通ABIから直接呼び出さず、正規値変換の内側だけで使用する。

```c
/* adapters/intel/sse2_pd.c: x86_64でだけコンパイルする。 */
#include <emmintrin.h>

#if !defined(__x86_64__)
#error "this adapter must be compiled for x86_64"
#endif

_Static_assert(sizeof(__m128d) == 16, "__m128d must be 128 bits");

static __m128d intel_call_mm_add_pd(__m128d a, __m128d b)
{
    return _mm_add_pd(a, b);
}

static __m128d intel_call_mm_set1_pd(double value)
{
    return _mm_set1_pd(value);
}
```

```c
/* adapters/openpower/sse2_pd.c: ppc64leでだけコンパイルする。 */
#include "project_power_sse2_compat.h"

#if !defined(__powerpc64__) || !defined(__BYTE_ORDER__) || \
    !defined(__ORDER_LITTLE_ENDIAN__) || \
    __BYTE_ORDER__ != __ORDER_LITTLE_ENDIAN__
#error "this adapter must be compiled for ppc64le"
#endif

#if !defined(IOITF_POWER_SSE2_COMPAT) || IOITF_POWER_SSE2_COMPAT != 1
#error "the project POWER SSE2 compatibility header is required"
#endif

_Static_assert(sizeof(__m128d) == 16, "__m128d must be 128 bits");

static __m128d power_call_mm_add_pd(__m128d a, __m128d b)
{
    return _mm_add_pd(a, b); /* 上記の移植ラッパーがテスト対象。 */
}

static __m128d power_call_mm_set1_pd(double value)
{
    return _mm_set1_pd(value); /* 上記の移植ラッパーがテスト対象。 */
}
```

#### ケースアダプターのコード解説と確認条件

2個の翻訳単位で関数本体が似ているのは意図した構造であり、同じ実装を二重にテストしているわけではない。Cプリプロセッサが読み込むヘッダーにより、呼び出し式の`_mm_add_pd`と`_mm_set1_pd`が次のように異なる定義へ結び付く。

| 項目 | x86_64翻訳単位 | ppc64le翻訳単位 |
|---|---|---|
| 読み込むヘッダー | コンパイラ付属の`<emmintrin.h>` | プロジェクトの`project_power_sse2_compat.h` |
| `_mm_add_pd`の実体 | x86 SSE2 Intrinsic | POWERへ移植したインラインラッパー |
| `_mm_set1_pd`の実体 | x86 SSE2 Intrinsic | POWERへ移植したインラインラッパー |
| アダプターの役割 | Intel基準実装への転送 | 検証対象であるPOWER実装への転送 |

`intel_call_mm_add_pd`と`power_call_mm_add_pd`は、受け取った2個の`__m128d`を変更せず対応する`_mm_add_pd`へ1回渡し、その戻り値を変更せず返す。各`__m128d`は2本の64ビット浮動小数点レーンを持ち、演算結果は`[a[0] + b[0], a[1] + b[1]]`である。`intel_call_mm_set1_pd`と`power_call_mm_set1_pd`は1個の`double`を対応する`_mm_set1_pd`へ渡し、同じ値を2レーンへ複製した結果を返す。これらの関数は`static`なので翻訳単位内だけで使用され、ケースレジストリへ公開する共通ABIシンボルではない。JSONの解析、レーン変換、比較も行わない。

アーキテクチャガードは誤ったホスト向けのコンパイルを失敗させ、`_Static_assert`は両ヘッダーの`__m128d`が16バイトであることを検証する。`IOITF_POWER_SSE2_COMPAT`はPOWERアダプターがプロジェクトの互換ヘッダーを経由したことを検証する識別マクロであり、演算の正しさそのものを保証する機能フラグではない。互換ヘッダーは、このマクロを`1`として定義しなければならない（MUST）。

ビルドはシステムヘッダーを含むヘッダー依存関係ファイルを翻訳単位ごとに生成しなければならない（MUST）。Intel側の依存関係にはコンパイラが報告する標準インクルード検索パス配下の`emmintrin.h`が存在し、`project_power_sse2_compat.h`が存在しないことを確認する。POWER側の依存関係にはプロジェクト内の正規化済み絶対パスで`project_power_sse2_compat.h`が存在し、basenameが`emmintrin.h`であるファイルが存在しないことを確認する。いずれかを満たさないビルドを拒否しなければならない（MUST）。

インライン化後は`_mm_add_pd`や`_mm_set1_pd`の外部シンボルがオブジェクトファイルに残らないことがあるため、シンボル名の有無だけを正しい配線の証拠にしてはならない（MUST NOT）。少なくとも次の3段階をすべて通過した場合に、この例の配線が正しいと判定する。

1. **ビルド検査**: 上記ガード、静的表明、ヘッダー依存関係検査が両ホストで成功する。
2. **正規値変換検査**: 14節の自己テストにより、`f64x2`のレーン0とレーン1をネイティブ型へ変換して再抽出した生ビット列が入力と一致する。
3. **既知結果検査**: `_mm_add_pd`へ`a = [1.0, 10.0]`と`b = [2.0, 20.0]`を与え、両ホストで`[3.0, 30.0]`を得る。正規形式の期待ビット列は`["0x4008000000000000", "0x403e000000000000"]`とする。`_mm_set1_pd`へ`-0.0`（`0x8000000000000000`）を与え、両出力レーンがともに`0x8000000000000000`であることを`bit_exact`で確認する。

既知結果検査は配線、基本演算、レーン独立性を確認する最小検査であり、全入力に対する等価性の証明ではない。境界値、NaN、丸めモード、浮動小数点例外は9節、13.2節、13.5節に従って別のテストベクトルで検証しなければならない（MUST）。

#### 誤ったレーン処理を検出する否定対照

上記の正常系が通ることだけでは、比較器が常に一致を返す欠陥や、結果の一部を比較しない欠陥を除外できない。フレームワーク自身のテストでは、POWER側テストフィクスチャの「ネイティブ戻り値を正規レーンへ抽出した直後、JSONへ符号化する前」に限って、次の故障注入関数を適用できるようにしなければならない（MUST）。本番アダプターおよび移植ラッパーへこの関数を組み込んではならない（MUST NOT）。

```c
#include <stdint.h>

/* テスト専用: f64x2の論理レーン0と1を故意に入れ替える。 */
static void ioitf_fault_swap_f64x2(uint64_t lane_bits[2])
{
    uint64_t temporary = lane_bits[0];
    lane_bits[0] = lane_bits[1];
    lane_bits[1] = temporary;
}

/* テスト専用: 論理レーン1の符号ビットを故意に反転する。 */
static void ioitf_fault_flip_f64x2_lane1_sign(uint64_t lane_bits[2])
{
    lane_bits[1] ^= UINT64_C(0x8000000000000000);
}
```

ここで`lane_bits[0]`と`lane_bits[1]`は、14節の正規レーン順で抽出済みのIEEE 754 binary64生ビット列である。この位置で故障を注入するため、C配列の物理バイト順やPOWERのネイティブ要素番号は故障内容へ影響しない。故障注入は1回の自己テストにつき次の片方だけを有効にし、Intel側の結果は変更しない。

1. `_mm_add_pd`の既知入力`a = [1.0, 10.0]`、`b = [2.0, 20.0]`に`ioitf_fault_swap_f64x2`を適用した実行は不一致にならなければならない（MUST）。`mismatch_count`は`2`、`first_difference`は`kind: return`、`lane: 0`、Intel値`0x4008000000000000`、POWER値`0x403e000000000000`でなければならない（MUST）。
2. `_mm_set1_pd`の既知入力`-0.0`に`ioitf_fault_flip_f64x2_lane1_sign`を適用した実行は`bit_exact`で不一致にならなければならない（MUST）。`mismatch_count`は`1`、`first_difference`は`kind: return`、`lane: 1`、Intel値`0x8000000000000000`、POWER値`0x0000000000000000`でなければならない（MUST）。

各否定対照の直前または直後に、同じバイナリ、入力、ケース定義を故障注入なしで実行し、一致することも確認しなければならない（MUST）。否定対照が一致する、指定外のレーンが異なる、または`first_difference`が上記と異なる場合は、移植ラッパーの合否を判定せずフレームワーク自身のテスト失敗とする。この手順は18節の「意図的に誤ったPOWER実装」と20節のレーン逆転・1ビット誤り検出を、この`f64x2`例について具体化する。

ケース定義の`intel.symbol`と`openpower.symbol`には、それぞれ共通ABIまで含む公開ケースシンボル（例: `intel_mm_add_pd`と`power_mm_add_pd`）を指定する。公開ケースシンボルは、正規形式の`f64x2`を復号し、上記の`*_call_*`を呼び、結果の各論理レーンをIEEE 754生ビット列へ戻す。

`_mm_add_pd`ケースの`standard`プロファイルは、`a[0] != a[1]`かつ`b[0] != b[1]`となり、2本の加算結果も異なる有限値入力を少なくとも1件含めなければならない（MUST）。正規値変換の自己テストは、変換直後のレーン0とレーン1を個別に表明しなければならない（MUST）。`_mm_set1_pd`ケースの`standard`プロファイルは`+0`、`-0`、0以外の有限値をそれぞれ入力し、各入力について両出力レーンの生ビット列が入力と一致することを`bit_exact`で検証しなければならない（MUST）。加算結果のNaN、符号付きゼロ、丸め、および例外フラグは、ケース定義で選択した13.2節と13.5節の規則に従って判定する。

### 6.3 ケースID

ケースIDは、次の形式を推奨する（SHOULD）。

```text
<family>.<operation>.<element-type>.<logical-width>.<variant>
```

例:

```text
avx2.add.i32x8.wrap
sse41.max.f32x4.default
ssse3.shuffle.u8x16.masked
```

ケースIDは一度公開した後に意味を変更してはならない（MUST NOT）。意味を変更する場合は新しいIDを追加する。

## 7. ケース定義

ケース定義はYAMLまたはJSONで管理し、少なくとも次のフィールドを持たなければならない（MUST）。

```yaml
schema_version: 1
id: avx2.add.i32x8.wrap
description: signed 32-bit lane-wise wrapping addition
intel:
  symbol: intel_mm256_add_epi32
  required_isa: [avx2]
openpower:
  symbol: power_mm256_add_epi32
  required_isa: [power8, vsx]
signature:
  arguments:
    - {name: a, type: vector, element: i32, lanes: 8}
    - {name: b, type: vector, element: i32, lanes: 8}
  return: {type: vector, element: i32, lanes: 8}
input_domain:
  exclude: []
comparison:
  mode: bit_exact
environment:
  fp_rounding_modes: [nearest_even]
  observe_fp_exceptions: false
```

### 7.1 即値オペランド

Intrinsicがコンパイル時定数を要求する引数は、`signature.arguments`で`type: immediate`として宣言し、同じ引数名を`immediates`のキーにしなければならない（MUST）。`type: immediate`の引数は8.2節の`operands`へ格納せず、入力レコードの`immediates`へ格納する。

```yaml
schema_version: 1
id: sse2.shuffle.i32x4.imm8
description: shuffle four 32-bit lanes using an imm8 control
intel:
  symbol: intel_mm_shuffle_epi32
  required_isa: [sse2]
openpower:
  symbol: power_mm_shuffle_epi32
  required_isa: [power8, vsx]
signature:
  arguments:
    - {name: a, type: vector, element: i32, lanes: 4}
    - {name: imm8, type: immediate, element: u8}
  return: {type: vector, element: i32, lanes: 4}
immediates:
  imm8:
    values: [0, 1, 27, 255]
    compile_time: true
input_domain:
  exclude: []
comparison:
  mode: bit_exact
environment:
  fp_rounding_modes: [nearest_even]
  observe_fp_exceptions: false
```

`immediates`のキー集合は`type: immediate`の引数名の集合と完全に一致しなければならない（MUST）。各定義は`values`と`compile_time`だけを持ち、次の制約を満たす。

- `values`は空でない昇順のJSON整数配列とし、重複を許さない。各値は対応する`element`で表現可能でなければならない（MUST）。
- `compile_time`は真でなければならない（MUST）。ランタイム値を受け取れる引数は`type: scalar`とし、`immediates`へ宣言してはならない（MUST NOT）。
- 複数の即値引数がある場合、宣言した`values`の直積を許容即値タプルとする。アダプターはその全タプルを処理できなければならない（MUST）。直積の一部だけを対象にする場合は、対象タプルごとに別ケースIDを定義する。

生成器は宣言値以外を出力してはならず（MUST NOT）、ランナーはIntrinsic呼び出し前に値を宣言済み集合と照合しなければならない（MUST）。値を要素幅でマスクする、符号変換する、剰余を取るなどして未宣言値を宣言値へ変換してはならない（MUST NOT）。未宣言値は`invalid_input`として記録する。

コンパイル時定数を要求するIntrinsicへ、JSONから読み取った変数を直接渡してはならない（MUST NOT）。アダプターはswitch分岐または許容値ごとに生成した関数を用い、Intrinsicの呼び出し式に整数リテラルが現れるようにする。

#### C言語による即値ディスパッチ例

次のIntelアダプター内部関数は、上記ケースの4個の宣言値をコンパイル時定数として`_mm_shuffle_epi32`へ渡す。ネイティブ型を使うこの関数は共通ABIの外側に置く。POWERアダプターも同じ4個の入力値を受理してIntel側と同じ論理shuffleを実装するが、使用するPOWER Intrinsicがランタイム制御値を受理する場合はswitchを省略できる（MAY）。

```c
#include <immintrin.h>
#include <stdbool.h>
#include <stdint.h>

bool intel_mm_shuffle_epi32(__m128i a, uint32_t imm8, __m128i *out) {
    if (out == NULL) {
        return false;
    }

    switch (imm8) {
    case 0:
        *out = _mm_shuffle_epi32(a, 0);
        return true;
    case 1:
        *out = _mm_shuffle_epi32(a, 1);
        return true;
    case 27:
        *out = _mm_shuffle_epi32(a, 27);
        return true;
    case 255:
        *out = _mm_shuffle_epi32(a, 255);
        return true;
    default:
        return false;
    }
}
```

このコードでは、`a`が4本の32ビット論理レーンを持つ入力、`imm8`が入力JSONLから検証済みの即値、`out`が結果の格納先である。戻り値の`true`はIntrinsicを実行して`out`を書き込んだこと、`false`は出力先が無効または即値が未宣言で、Intrinsicを実行しなかったことを表す。ランナーは未宣言即値による`false`を`invalid_input`へ変換する。

`_mm_shuffle_epi32`では、出力レーン`j`が参照する入力レーン番号を次の式で求める。

```text
source_lane(j) = (imm8 >> (2 * j)) & 0x3,  j = 0, 1, 2, 3
```

例えば`imm8 = 27`は16進数で`0x1b`であり、論理レーン0から昇順に並べた結果は`[a[3], a[2], a[1], a[0]]`となる。つまり4レーンの順序を反転する。`imm8 = 0`では全出力レーンが`a[0]`を選び、`imm8 = 255`では全出力レーンが`a[3]`を選ぶ。これらの説明における`a[j]`は14節の論理レーンであり、メモリ上の生バイト位置を意味しない。

switchの各`case`に整数リテラルを直接書くことで、コンパイラはIntrinsicの即値制約を満たせる。`default`で値をマスクせず拒否するため、例えば`256`が`0`として実行されることはない。POWERアダプターは同じ`source_lane`の式で得られる論理結果を返さなければならず（MUST）、POWER命令の制御ビット配置が異なる場合はアダプター内で変換する。

ビルド時にディスパッチを生成する場合は、検証済みケース定義の`values`だけを生成元にしなければならない（MUST）。ディスパッチを手書きする場合は、アダプター自己テストで全許容即値タプルを呼び出せなければならない（MUST）。`standard`プロファイルでは各許容タプルを少なくとも1回実行し、未宣言値をランナーが呼び出し前に拒否する負例をフレームワーク自身のテストへ含めなければならない（MUST）。

仕様上結果が未定義または不定となる入力は、`input_domain.exclude`または生成器の制約述語で明示しなければならない（MUST）。例として、ゼロ除算、範囲外シフト、範囲外インデックス、アラインメント要件違反がある。

## 8. 正規入力形式

### 8.1 物理形式

初期実装はUTF-8 JSON Lines（JSONL）を使用する（MUST）。BOM、空行、CRLFを許可せず、1行を1テストベクトルとする。各行は[RFC 8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785.html)で直列化したJSONオブジェクトとし、行末にLFを1個付けなければならない（MUST）。JSONの重複キー、RFC 8785で表現できない値、行末以外の空白を拒否する。

大規模実行でI/Oが問題となる場合は、同じ論理モデルを持つCBOR等を追加できる（MAY）。ただし、同一テスト実行内で形式を混在させてはならない（MUST NOT）。

### 8.2 入力レコード

`schema_version: 1`の入力レコードは、次のフィールドを持たなければならない（MUST）。条件付きフィールドを除く未定義フィールドは拒否する。

| フィールド | JSON型 | 制約 |
|---|---|---|
| `schema_version` | integer | `1` |
| `case_id` | string | ケースレジストリに存在する空でないID |
| `input_id` | string | 8.4節で導出する小文字64桁のSHA-256 |
| `sequence` | integer | ファイル先頭を1とする連続した正整数。行番号と一致 |
| `environment` | object | `fp_mode`と`rounding`を格納 |
| `operands` | object | 即値を除く全引数を引数名で格納 |
| `generation` | object | 入力の生成クラスと疑似乱数由来情報を格納 |
| `immediates` | object | ケース定義に`immediates`がある場合だけ必須 |
| `buffers` | object | ポインター引数がある場合だけ必須 |

`environment`は`fp_mode: ieee`と`rounding`だけを持つ。`rounding`は`nearest_even`、`toward_zero`、`toward_positive`、`toward_negative`のいずれかで、ケース定義の`environment.fp_rounding_modes`に含まれなければならない（MUST）。

`generation.class`は`boundary`、`structured`、`exhaustive`、`random`、`regression`のいずれかとする。`random`の場合だけ`algorithm`と`seed`を追加し、`algorithm`はケース生成時に固定した疑似乱数アルゴリズム名、`seed`は`^0x[0-9a-f]{16}$`に一致する64ビット値とする（MUST）。他のクラスでは`algorithm`と`seed`を格納してはならない（MUST NOT）。

`operands`のキー集合は、ケース定義の`signature.arguments`から即値を除いた引数名の集合と一致しなければならない（MUST）。`immediates`のキー集合はケース定義の`immediates`と一致し、各値はJSON整数かつ宣言された`values`のいずれかでなければならない（MUST）。

例:

```json
{"case_id":"avx2.add.i32x8.wrap","environment":{"fp_mode":"ieee","rounding":"nearest_even"},"generation":{"class":"structured"},"input_id":"7479d5e1b297269d6df7214a15826a2c6f8658b28b4b61128e7681eec3af0472","operands":{"a":{"element":"i32","lanes":["0x00000000","0x00000001","0x7fffffff","0x80000000","0xffffffff","0x55555555","0xaaaaaaaa","0x12345678"]},"b":{"element":"i32","lanes":["0x00000000","0xffffffff","0x00000001","0xffffffff","0x00000001","0xaaaaaaaa","0x55555555","0xedcba988"]}},"schema_version":1,"sequence":1}
```

### 8.3 正規値とメモリ引数

- ベクトルは論理レーン番号0から昇順の配列として表す（MUST）。
- 整数スカラーは`{"bits":"0x...","element":"i32"}`の形、浮動小数点スカラーは`{"bits":"0x...","element":"f32"}`の形で表す（MUST）。整数の`element`は`i8`、`i16`、`i32`、`i64`、`u8`、`u16`、`u32`、`u64`、浮動小数点の`element`は`f32`または`f64`とする。
- ベクトルとマスクは`{"element":"...","lanes":["0x...", ...]}`と表し、`lanes`の要素数をケース定義のレーン数と一致させる（MUST）。
- 整数値は2の補数による生ビット列とし、要素幅を保持する小文字16進文字列として表す（MUST）。各文字列は`0x`に続けて要素幅の4分の1個の16進数字を持たなければならない（MUST）。
- 浮動小数点値は10進文字列ではなくIEEE 754の生ビット列として表す（MUST）。これにより、NaN payload、符号付きゼロ、subnormalを保持する。
- マスクは各論理レーンの全ビットを含む値として表し、単なる真偽値へ縮約してはならない（MUST NOT）。
- `element`、要素幅、レーン数は対応するケース定義と一致しなければならない（MUST）。値のビット列にJSON数値を使用してはならない（MUST NOT）。

#### C言語による`i32x8`復号例

次の例は、JSONパーサーが返した8本のレーン文字列を検証し、符号付きC演算を経由せず`uint32_t`の生ビット列へ復号する。IntelアダプターとPOWERアダプターは、この同じ`lane_bits`を14節の論理レーン規則に従ってネイティブベクトルへ構築しなければならない（MUST）。

```c
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>
#include <string.h>

static int ioitf_hex_nibble(char c) {
    if (c >= '0' && c <= '9') {
        return c - '0';
    }
    if (c >= 'a' && c <= 'f') {
        return c - 'a' + 10;
    }
    return -1;
}

static bool ioitf_parse_u32_bits(const char *text, uint32_t *value) {
    uint32_t parsed = 0;

    if (text == NULL || value == NULL || strlen(text) != 10 ||
        text[0] != '0' || text[1] != 'x') {
        return false;
    }

    for (size_t i = 0; i < 8; ++i) {
        int nibble = ioitf_hex_nibble(text[i + 2]);
        if (nibble < 0) {
            return false;
        }
        parsed = (parsed << 4) | (uint32_t)nibble;
    }

    *value = parsed;
    return true;
}

bool ioitf_decode_i32x8(const char *const lanes[8],
                        uint32_t lane_bits[8]) {
    uint32_t decoded[8];

    if (lanes == NULL || lane_bits == NULL) {
        return false;
    }
    for (size_t i = 0; i < 8; ++i) {
        if (!ioitf_parse_u32_bits(lanes[i], &decoded[i])) {
            return false;
        }
    }

    memcpy(lane_bits, decoded, sizeof(decoded));
    return true;
}
```

例えば`0xffffffff`は`UINT32_C(0xffffffff)`として保持し、`int32_t`の`-1`へ変換してから再符号化してはならない（MUST NOT）。この例はJSON構文解析そのものを規定せず、構文解析後の文字列検証とビット復号の規則を示す。

メモリアドレスそのものは保存しない。ポインター引数があるレコードでは、`buffers`へ割り当て単位を定義し、ポインターオペランドからそれを参照する（MUST）。

```json
{"buffers":{"buf0":{"alignment":32,"bytes":"0x00010203aabbccdd"}},"operands":{"dst":{"buffer":"buf0","offset":4},"src":{"buffer":"buf0","offset":0}}}
```

バッファIDは`^[A-Za-z_][A-Za-z0-9_]*$`に一致しなければならない。`alignment`は正の2の累乗であり、割り当て先頭アドレスに要求するアラインメントをバイト単位で表す。`bytes`は割り当て全体の初期内容を`0x`に続く偶数個の小文字16進数字で表し、空バッファは`0x`とする。ポインター値は`buffer`と0以上の`offset`だけを持ち、`offset`は参照先のバイト長以下でなければならない（MUST）。アクセス範囲が割り当て内に収まるかはケース定義のシグネチャと`input_domain`に従って検証する。

複数のポインターが同じバッファIDを参照する場合は別名参照、異なるIDを参照する場合は互いに重ならない割り当てとする（MUST）。これ以外の方法でホスト固有アドレスやポインター関係を推測してはならない（MUST NOT）。canaryを含む割り当て全体を`bytes`へ格納し、実行後は13.4節に従って割り当て全体を観測する。

### 8.4 `input_id`の導出と検証

`input_id`は推奨値ではなく、次の手順で必ず導出しなければならない（MUST）。

1. 入力レコードから`input_id`、`schema_version`、`sequence`、`generation`を除く。
2. 残った`case_id`、`environment`、`operands`と、存在する場合は`buffers`、`immediates`を1個のJSONオブジェクトにする。
3. そのオブジェクトをRFC 8785で直列化する。末尾にLFを加えない。
4. 直列化したUTF-8バイト列のSHA-256を計算し、小文字64桁の16進文字列を`input_id`とする。

この導出により、生成方法やJSONL内の位置が異なっても、SUTへ渡す入力と実行環境が同じレコードは同じIDになる。生成器と両ランナーは`input_id`を再計算し、レコード値と一致しない場合はSUTを呼び出さず、入力成果物を仕様・入力エラーとして拒否しなければならない（MUST）。同じ`input_id`を持つレコードを1個の入力成果物へ複数格納してはならない（MUST NOT）。

### 8.5 入力成果物マニフェスト

`generate-vectors`は`test-vectors.jsonl`とともに`test-vectors.manifest.json`を生成しなければならない（MUST）。マニフェストは再帰的にキーを辞書順へ整列した単一のUTF-8 JSONオブジェクトとし、末尾にLFを1個付ける。`schema_version: 1`の構造は次のとおりとする。

```json
{
  "artifact_type": "ioitf.test-vectors",
  "case_definitions_sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "complete": true,
  "profile": "smoke",
  "schema_version": 1,
  "test_vectors": {
    "byte_length": 123456,
    "file": "test-vectors.jsonl",
    "record_count": 1000,
    "sha256": "fedcba9876543210fedcba9876543210fedcba9876543210fedcba9876543210"
  }
}
```

- `artifact_type`は`ioitf.test-vectors`、`schema_version`は整数`1`、`complete`は真でなければならない（MUST）。`profile`は9.4節のプロファイル名のいずれかとする。
- `case_definitions_sha256`は、検証済みケース定義をケースID昇順のJSON配列へ変換し、RFC 8785で直列化して末尾にLFを1個付けたバイト列のSHA-256とする（MUST）。YAML入力はJSONデータモデルへ変換し、重複キーまたはRFC 8785で表現できない値を拒否する。
- `test_vectors.file`は`test-vectors.jsonl`、`byte_length`はファイルの全バイト数、`record_count`は1以上かつJSONLの行数と同じ整数、`sha256`は末尾LFを含むファイル全体のSHA-256としなければならない（MUST）。SHA-256値は小文字64桁の16進文字列とする。

生成器はJSONLを一時名へ書いて全レコードと`input_id`を検証し、最終名へ確定してからマニフェストを最後に公開しなければならない（MUST）。両ランナーはSUT実行前にマニフェストのスキーマ、行数、バイト数、SHA-256、ケース定義のSHA-256を検証する。欠落または不一致は`infrastructure_error`として入力成果物全体を拒否し、結果マニフェストの`input_sha256`には検証済みの`test_vectors.sha256`を複写しなければならない（MUST）。

## 9. テストデータ生成

生成器は、次の入力クラスを組み合わせなければならない（MUST）。

### 9.1 決定的境界値

- 符号付き整数: `0`、`1`、`-1`、最小値、最大値、最小値±1、最大値±1
- 符号なし整数: `0`、`1`、最大値、最大値-1、各単一ビット
- 浮動小数点:
  - `+0`、`-0`
  - 最小/最大subnormal
  - 最小/最大normal
  - `+∞`、`-∞`
  - quiet NaN、signaling NaN（対象環境が安全に扱える場合）
  - `±1`、丸め境界、演算でoverflow/underflowを起こす組み合わせ
- マスク: 全0、全1、交互ビット、単一レーン有効、単一レーン無効
- シフト・シャッフル: `0`、要素幅-1、要素幅、最大許容即値、および仕様で定義された範囲

### 9.2 構造化ベクトル

- 全レーン同値
- 単調増加・単調減少
- レーン番号を値にしたパターン
- 上位半分と下位半分で異なる値
- 交互符号
- 桁上がり・借り・飽和がレーン境界ごとに発生する値
- バイト順の誤りを検出できる非対称値（例: `0x01234567`）

### 9.3 疑似乱数

- 乱数列はアルゴリズム名とシードを成果物へ保存する（MUST）。
- 実装言語や標準ライブラリに依存しないよう、SplitMix64等、出力アルゴリズムを固定した生成器を使用する（SHOULD）。
- 既定シードをリポジトリへ固定し、CIでは固定シード群を使用する（MUST）。
- 手動・夜間テストでは追加シードを指定できる（MAY）。失敗時は必ずシードを記録する。

### 9.4 組合せ数

既定プロファイルを次のように定義する。

| プロファイル | 目的 | 1ケース当たり目安 |
|---|---|---:|
| smoke | PRごとの早期検出 | 32〜128 |
| standard | 通常CI | 1,000〜10,000 |
| exhaustive-small | 8/16ビット等の全探索可能領域 | ケース依存 |
| stress | 夜間・リリース前 | 100,000以上 |

## 10. 実行環境の正規化

各ランナーは、実行結果とともに次の情報を出力しなければならない（MUST）。

- OSとカーネル
- CPUアーキテクチャ、CPUモデル、利用可能ISA
- コンパイラ名・バージョン
- コンパイルオプション
- git commit
- ケース定義のSHA-256
- 入力成果物のSHA-256
- ランナーのビルドID
- エンディアン
- 浮動小数点丸めモード
- x86側のMXCSRにおけるFTZ/DAZ設定
- POWER側で使用した対応浮動小数点設定

初期既定では、丸めモードをround-to-nearest ties-to-evenとし、x86のFTZとDAZを無効化してgradual underflowを使用する（MUST）。異なるモードを検証する場合は別テストベクトルとして明示する。

コンパイラによる演算の融合・再関連付け・fast-mathは結果を変え得るため、既定ビルドでは禁止する（MUST）。FMAそのものを対象とするケースでは、FMAとして明示した実装だけを比較する。

## 11. ランナー要件

各ランナーは次を満たさなければならない（MUST）。

1. 起動時に必要ISAを検出し、非対応ケースを実行しない。
2. 入力をストリーミングで読み、各入力に対して結果を1件出力する。
3. 異常終了したケースを他のケースから可能な範囲で分離する。
4. 入力順序に依存しない結果を生成する。
5. 同一ビルド・同一入力を複数回実行した際に、`duration_ns`を除く`status`と観測結果が同じである。
6. 結果の欠落、重複、不明な`input_id`を検出する。
7. `SIGILL`、`SIGSEGV`、`SIGFPE`等をケース失敗として記録するか、少なくとも実行単位の異常として報告する。
8. 結果JSONLを途中まで書いた状態でも、完了マニフェストがなければ完全な成果物として扱わない。

ランナーは、ISA別に別バイナリを生成しても、同一ソースを条件付きコンパイルしてもよい（MAY）。ただし、同じバイナリ内でIntel実装をスカラー参照実装へ置換するなど、実際の移植元Intrinsicを迂回してはならない（MUST NOT）。

## 12. 結果形式

各入力に対応する結果レコードは、少なくとも`schema_version`、`case_id`、`input_id`、`runner`、`status`、`duration_ns`を含まなければならない（MUST）。`status`が`ok`の場合は`observed`も必須とし、ケース定義で宣言した戻り値、メモリ副作用、浮動小数点例外のうち観測対象となる値をすべて格納する。`duration_ns`は0以上の整数とする。

`duration_ns`は診断用メタデータであり、Intel結果とPOWER結果の等価性比較および11節の再実行決定性判定から除外しなければならない（MUST）。

```json
{"case_id":"avx2.add.i32x8.wrap","duration_ns":87,"input_id":"7479d5e1b297269d6df7214a15826a2c6f8658b28b4b61128e7681eec3af0472","observed":{"return":{"element":"i32","lanes":["0x00000000","0x00000000","0x80000000","0x7fffffff","0x00000000","0xffffffff","0xffffffff","0x00000000"]}},"runner":"intel","schema_version":1,"status":"ok"}
```

`status`は次のいずれかとする。

- `ok`: 正常に観測結果を取得
- `unsupported`: 必要ISAまたはコンパイラ機能がない
- `invalid_input`: ケース定義に反する入力
- `signal`: シグナルで停止
- `runtime_error`: ラッパーまたはランナーのエラー
- `infrastructure_error`: 入出力、成果物、環境のエラー

`unsupported`は不一致成功として数えてはならない（MUST NOT）。必須ケースが`unsupported`の場合、テストスイート全体を環境エラーとする。

### 12.1 結果成果物の完了マニフェスト

各ランナーの結果成果物は、結果JSONLと1個の完了マニフェストJSONで構成しなければならない（MUST）。ファイル名はIntel側を`intel-results.jsonl`と`intel-results.manifest.json`、POWER側を`power-results.jsonl`と`power-results.manifest.json`とする。マニフェストは、キーを再帰的に辞書順へ整列した単一のUTF-8 JSONオブジェクトとし、末尾にLFを1個付ける。`schema_version: 1`では次の構造を持たなければならない（MUST）。

```json
{
  "artifact_type": "ioitf.runner-results",
  "case_definitions_sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "complete": true,
  "environment": {
    "architecture": "x86_64",
    "available_isa": ["avx2", "sse2"],
    "compile_options": ["-O2", "-mavx2", "-fno-fast-math"],
    "compiler": {"name": "gcc", "version": "15.1.0"},
    "cpu_model": "example-cpu",
    "endianness": "little",
    "fp_controls": {"exception_traps_enabled": false, "mxcsr_daz": false, "mxcsr_ftz": false},
    "fp_rounding_modes": ["nearest_even"],
    "git_commit": "0123456789abcdef0123456789abcdef01234567",
    "kernel": "6.x.y",
    "os": "linux"
  },
  "input_sha256": "fedcba9876543210fedcba9876543210fedcba9876543210fedcba9876543210",
  "results": {
    "byte_length": 12345,
    "file": "intel-results.jsonl",
    "record_count": 1000,
    "sha256": "0011223344556677001122334455667700112233445566770011223344556677"
  },
  "runner": {"build_id": "intel-runner-2026-09-01.1", "role": "intel"},
  "schema_version": 1
}
```

マニフェストの各値には次の制約を適用する。

- `artifact_type`は`ioitf.runner-results`、`schema_version`は整数`1`、`complete`は真でなければならない（MUST）。
- 3個のSHA-256値は小文字の64桁16進文字列とする。`input_sha256`は末尾改行を含む入力JSONLの全バイト、`results.sha256`は末尾改行を含む結果JSONLの全バイトを対象とする（MUST）。
- `case_definitions_sha256`は8.5節と同じ正規化済みケース定義のSHA-256でなければならない（MUST）。元ファイルの空白、コメント、ファイル列挙順をハッシュへ含めてはならない（MUST NOT）。
- `runner.role`は`intel`または`openpower`とし、前者は`environment.architecture: x86_64`および`intel-results.jsonl`、後者は`environment.architecture: ppc64le`および`power-results.jsonl`と組み合わせなければならない（MUST）。`runner.build_id`は空でない文字列とする。
- `available_isa`はランナー起動時に検出した利用可能ISA、`fp_rounding_modes`は実行した全入力で使用した丸めモードを、それぞれ重複なしの辞書順で格納する。丸めモード名は`nearest_even`、`toward_zero`、`toward_positive`、`toward_negative`のいずれかとする（MUST）。`compile_options`はコンパイラへ渡した引数を、シェル結合前の文字列配列として順序を保持して格納する（MUST）。
- `fp_controls`には全ケースに共通する浮動小数点制御状態を格納する。x86_64では`exception_traps_enabled`、`mxcsr_daz`、`mxcsr_ftz`、ppc64leでは`exception_traps_enabled`、`fpscr_ni`、`vscr_nj`を真偽値で格納しなければならない（MUST）。入力ごとに変わる丸めモードと例外フラグは`fp_controls`へ格納しない。
- `results.file`はパス区切りと`..`を含まない上記のファイル名、`byte_length`と`record_count`は0以上の整数とする。`record_count`は空行を許さない結果JSONLの行数であり、入力JSONLの行数と一致しなければならない（MUST）。
- `environment.os`、`kernel`、`architecture`、`cpu_model`、`compiler.name`、`compiler.version`は10節で要求した実測値を表す空でない文字列とする。`git_commit`は小文字の40桁または64桁16進文字列、`endianness`は`little`または`big`とする（MUST）。

ランナーは結果JSONLを一時名へ書き、全入力に対する結果を出力してファイルを閉じた後に、行数、バイト数、SHA-256を計算しなければならない（MUST）。結果JSONLを最終名へ確定してから完了マニフェストを作成し、ローカルファイルシステムでは同一ディレクトリ内の一時名からのアトミックなrename、CI成果物ストアでは結果JSONLのアップロード完了後のマニフェストアップロードによって、マニフェストを最後に公開しなければならない（MUST）。`complete: false`のマニフェストを公開してはならない（MUST NOT）。

比較器は、完了マニフェストの欠落、スキーマ違反、ファイル名・行数・バイト数・SHA-256の不一致を`infrastructure_error`として比較前に拒否しなければならない（MUST）。さらに、両マニフェストの`input_sha256`と`case_definitions_sha256`がそれぞれ一致し、各結果の`runner`がマニフェストの`runner.role`と一致することを確認する。これらの検証が完了するまで、値の一致件数または不一致件数を報告してはならない（MUST NOT）。

## 13. 比較規則

### 13.1 整数・ビット演算

戻り値が整数要素（`i8`、`i16`、`i32`、`i64`、`u8`、`u16`、`u32`、`u64`）またはマスクであるケースは、`comparison`として`mode: bit_exact`だけを持たなければならない（MUST）。追加フィールド、`ieee_value`、`ulp`、`abs_rel`、`classification`を指定したケース定義を拒否する。整数演算、論理演算、シフト、シャッフル、permute、pack、unpack、merge、extract、insert、およびマスクの結果は、正規化後の全ビットを比較する。Intel仕様上の動作がビットの複写または並べ替えだけであるケースは、戻り値の要素が`f32`または`f64`でも`mode: bit_exact`を指定しなければならない（MUST）。

比較器は値比較前に、両結果の`element`、要素幅、レーン数、および16進文字列幅がケース定義と一致することを検証しなければならない（MUST）。不一致は値の不一致ではなく、不正な結果成果物として`infrastructure_error`にする。形式検証後は論理レーン番号の昇順に固定幅の生ビット列を比較し、1ビットでも異なればケース不一致とする。符号付きと符号なしの数値へ変換した値が等しいかどうかは判定に使用しない。

戻り値が`f32`または`f64`であるケースでは、13.2節の`comparison.mode`を浮動小数点戻り値にだけ適用する。同じケースが整数値、マスク、またはメモリ副作用も観測する場合、それらは指定された浮動小数点モードにかかわらず`bit_exact`で比較しなければならない（MUST）。したがって、出力バッファへ格納された浮動小数点値にULPまたは絶対・相対許容差を適用するケースは、型付きメモリ観測を定義していない`schema_version: 1`では登録できない。

`schema_version: 1`は整数結果の無視ビット、無視レーン、比較前マスクを定義しない。許容入力に対してIntel仕様が未定義または不定としている出力ビットが1個でもあるケースを登録してはならない（MUST NOT）。入力に依存して未定義となる場合は、該当入力を`input_domain.exclude`または生成器の制約述語ですべて除外する。除外できないIntrinsicは初期対象外とする。比較器またはアダプターが結果を0クリアする、要素幅でマスクする、符号拡張する、真偽値へ縮約するなどして不一致を隠してはならない（MUST NOT）。

Intrinsicが`w`ビット整数の剰余演算を定義する場合、移植ラッパーは対応する`w`ビット符号なし型の演算、または同じ剰余動作を保証するIntrinsicを使用する。C/C++の符号付きオーバーフロー、負のシフト回数、左オペランドの型幅以上のシフト回数、負の符号付き値の左シフト、または負の符号付き値の右シフトにおける言語依存の動作を経由して結果を生成してはならない（MUST NOT）。比較対象は演算後の下位`w`ビットとする。

### 13.2 浮動小数点

比較前に、両結果の要素形式とビット幅がケース定義に一致することを検証しなければならない（MUST）。不一致は値の不一致ではなく、不正な結果成果物として扱う。

ケース定義の`comparison`は、次のモードを1つ指定し、表中の追加フィールドを持たなければならない（MUST）。適用されない追加フィールドは、設定ミスを検出するため拒否しなければならない（MUST）。

| モード | 必須追加フィールド | 判定 |
|---|---|---|
| `bit_exact` | なし | NaN payloadと符号付きゼロを含む全ビットが一致 |
| `ieee_value` | `signed_zero`、`nan` | 有限値は数学的な値が一致。特殊値は後述の先行規則で判定 |
| `ulp` | `max_ulps`、`signed_zero`、`nan` | 有限値のULP距離が`max_ulps`以下 |
| `abs_rel` | `abs_tolerance`、`rel_tolerance`、`signed_zero`、`nan` | 有限値が後述の絶対誤差または相対誤差条件を満たす |
| `classification` | `signed_zero`、`nan` | IEEE 754分類が一致し、NaNとゼロ以外では符号も一致 |

`signed_zero`は`equal`または`distinct`とする。`nan`は次の4フィールドをすべて持つ。

```yaml
nan:
  both_nan: equal        # equal | unequal
  quiet_signaling: match # match | ignore
  payload: ignore        # match | ignore
  sign: ignore           # match | ignore
```

`both_nan: unequal`の場合、両方がNaNでも不一致とし、残りの3フィールドは評価しない。`both_nan: equal`の場合、`match`を指定した属性は一致を要求し、`ignore`を指定した属性は判定から除外する。複数の`match`条件はすべて満たさなければならない。`payload`はquiet/signaling判別ビットを除いた仮数部と定義する。`bit_exact`ではこれらのポリシーを使わず、生ビット列だけを比較する。

例えば、最大1 ULPを許容し、NaNのquiet/signalingだけを区別するケースは次のように宣言する。

```yaml
comparison:
  mode: ulp
  max_ulps: 1
  signed_zero: equal
  nan:
    both_nan: equal
    quiet_signaling: match
    payload: ignore
    sign: ignore
```

`max_ulps`は0以上かつ`2^64 - 1`以下の整数とする。`abs_tolerance`と`rel_tolerance`は、正規表現`^[0-9]+(\.[0-9]+)?([eE][+-]?[0-9]+)?$`に一致する非負かつ有限の10進文字列とし、比較器はこれを厳密な10進有理数として解釈する。ホストの浮動小数点型へ丸めてから閾値を判定してはならない（MUST NOT）。

`bit_exact`以外のモードは、レーンごとに次の順序で判定しなければならない（MUST）。

1. 片方だけがNaNなら不一致とする。両方がNaNなら`nan`の4フィールドだけで判定し、数値誤差は計算しない。
2. NaNでない無限大は、両方が同じ符号の無限大の場合だけ一致とする。有限値との誤差は計算しない。
3. 両方がゼロなら、`signed_zero: equal`では符号にかかわらず一致、`distinct`では符号が同じ場合だけ一致とする。
4. `classification`では、`zero`、`subnormal`、`normal`の分類が一致し、かつゼロ以外では符号が一致する場合だけ一致とする。
5. `ieee_value`では、残った有限値をIEEE 754生ビット列から厳密な有理数へ変換し、その値が等しい場合だけ一致とする。
6. `ulp`では、要素幅を`n`、生ビット列を符号なし整数`u`、`S = 2^(n-1)`、`M = 2^n - 1`とし、順序キー`K(u)`を、符号ビットが1なら`M - u`、0なら`u | S`として計算する。まず`D = abs(K(a) - K(b))`とし、`signed_zero: equal`かつ両値の符号ビットが異なる場合は`D = max(D - 1, 0)`として、`-0`と`+0`の間を0 ULPに折り畳む。`D <= max_ulps`なら一致とする。
7. `abs_rel`では、残った有限値を厳密な有理数`a`、`b`として、`d = abs(a - b)`、`m = max(abs(a), abs(b))`を計算する。`d <= abs_tolerance`または`d <= rel_tolerance * m`の少なくとも一方を満たす場合だけ一致とする。途中計算も厳密な有理数演算で行う。

単純な加減算、ビット操作、比較など、両ISAで同じIEEE演算を表現できる場合は`bit_exact`を既定とする（SHOULD）。近似逆数、近似平方根、変換、異なる融合規則を持つ処理は、移植要件に基づいて閾値と特殊値ポリシーを明示する。

### 13.3 マスク

Intel Intrinsicsが「真」を全ビット1で返す場合、POWER実装も同じレーン幅の全ビット1へ正規化しなければならない（MUST）。真偽だけを比較して上位ビットの誤りを見逃してはならない（MUST NOT）。

### 13.4 メモリ副作用

load/store、masked store、scatter相当の処理では、戻り値だけでなく次を13.1節の`bit_exact`で比較する（MUST）。ケースの`comparison.mode`が`ulp`、`abs_rel`、`ieee_value`、`classification`のいずれでも、この規則は変わらない。

- 書き込み対象バッファの全内容
- 書き込み範囲外に置いたcanary
- 書き込みバイト数とオフセット
- 仕様で観測対象としたアラインメント

### 13.5 浮動小数点例外

ケース定義の`environment.observe_fp_exceptions`は真偽値として必須とし、省略を許可しない（MUST）。ケース雛形の既定値は偽とする。真の場合、例外フラグは戻り値やメモリ副作用とは独立した観測結果として合否判定へ含めなければならない（MUST）。偽の場合、ランナーは6.1節の`normalized_fp_exceptions`を0に設定し、結果JSONの`observed`へ`fp_exceptions`を格納してはならない（MUST NOT）。

正規化ビットとJSON名を次のように固定する。アーキテクチャ固有ビット列を共通ABIまたは結果JSONへ直接出力してはならない（MUST NOT）。

| 共通ABIビット | JSON名 | x86 MXCSRの入力 | POWER FPSCRの入力 |
|---:|---|---|---|
| `1u << 0` | `invalid` | invalid-operation flag | invalid-operation exception summary（VX） |
| `1u << 1` | `divide-by-zero` | divide-by-zero flag | zero-divide exception（ZX） |
| `1u << 2` | `overflow` | overflow flag | overflow exception（OX） |
| `1u << 3` | `underflow` | underflow flag | underflow exception（UX） |
| `1u << 4` | `inexact` | precision flag | inexact exception（XX） |

`normalized_fp_exceptions`の上記以外のビットは0でなければならない（MUST）。`observe_fp_exceptions: true`の場合、`observed.fp_exceptions`は発生したJSON名だけを、表の上から下の順序で重複なく格納した配列とする。例外が発生しなかった場合も空配列を格納する（MUST）。例えばinvalidとinexactが発生した結果は次の形になる。

```json
{"observed":{"fp_exceptions":["invalid","inexact"],"return":{"element":"f64","lanes":["0x7ff8000000000000","0x3ff0000000000000"]}}}
```

各入力の採取手順を次の順序に固定する（MUST）。

1. 入力の検証、正規値からネイティブ型への変換、丸めモードの設定を完了する。
2. 例外トラップが無効であることを確認する。有効な場合はSUTを呼び出さず、その入力を`infrastructure_error`とする。
3. MXCSRまたはFPSCRにあり、明示的に消去するまで保持される上記5個の例外状態フラグ（sticky flag）をすべて消去する。
4. 消去後から採取までに、テスト対象をちょうど1回だけ呼び出す。他の浮動小数点演算を実行してはならない（MUST NOT）。
5. 戻り値の正規値変換や10進診断値の計算より前に例外フラグを読み取り、上表の共通ABIビットへ正規化する。
6. 戻り値と副作用を符号化し、共通ABIビットから`observed.fp_exceptions`を生成する。

次のCコードは、対象環境の`<fenv.h>`がSIMD演算に使用されるMXCSRまたはFPSCRへ接続される場合の正規化例である。実装は同じ共通ABIビットを生成するアーキテクチャ固有処理へ置換できる（MAY）。

```c
#include <fenv.h>
#include <stdbool.h>
#include <stdint.h>

#pragma STDC FENV_ACCESS ON

#if !defined(FE_ALL_EXCEPT) || !defined(FE_INVALID) || \
    !defined(FE_DIVBYZERO) || \
    !defined(FE_OVERFLOW) || !defined(FE_UNDERFLOW) || \
    !defined(FE_INEXACT)
#error "all five IEEE 754 exception classes are required"
#endif

enum {
    IOITF_FP_INVALID        = UINT32_C(1) << 0,
    IOITF_FP_DIVIDE_BY_ZERO = UINT32_C(1) << 1,
    IOITF_FP_OVERFLOW       = UINT32_C(1) << 2,
    IOITF_FP_UNDERFLOW      = UINT32_C(1) << 3,
    IOITF_FP_INEXACT        = UINT32_C(1) << 4
};

static bool ioitf_fp_observation_begin(void)
{
    return feclearexcept(FE_ALL_EXCEPT) == 0;
}

static uint32_t ioitf_fp_observation_end(void)
{
    int raised = fetestexcept(FE_INVALID | FE_DIVBYZERO | FE_OVERFLOW |
                              FE_UNDERFLOW | FE_INEXACT);
    uint32_t normalized = 0;

    if ((raised & FE_INVALID) != 0) {
        normalized |= IOITF_FP_INVALID;
    }
    if ((raised & FE_DIVBYZERO) != 0) {
        normalized |= IOITF_FP_DIVIDE_BY_ZERO;
    }
    if ((raised & FE_OVERFLOW) != 0) {
        normalized |= IOITF_FP_OVERFLOW;
    }
    if ((raised & FE_UNDERFLOW) != 0) {
        normalized |= IOITF_FP_UNDERFLOW;
    }
    if ((raised & FE_INEXACT) != 0) {
        normalized |= IOITF_FP_INEXACT;
    }
    return normalized;
}
```

各ランナーは例外トラップを無効にした起動時自己テストで5個の例外を1個ずつ設定し、採取結果が対応する共通ABIビットだけを含むこと、および消去後の採取結果が0であることを確認しなければならない（MUST）。`<fenv.h>`が対象のSIMD状態へ接続されない場合は、MXCSRまたはFPSCRを直接操作する実装を使用する。自己テストを満たす採取機能がないランナーでは`observe_fp_exceptions: true`のケースを`unsupported`とし、採取開始または実行時検証に失敗した入力は`infrastructure_error`とする。

比較器は、`observe_fp_exceptions: true`の入力が1件以上ある場合、両結果マニフェストの`fp_controls.exception_traps_enabled`が偽であることを検証する。さらに、該当する両結果に`fp_exceptions`が存在し、配列が許可された順序、値、重複禁止を満たすことを値比較前に検証する。欠落またはスキーマ違反は結果成果物の`infrastructure_error`とする。妥当な配列同士は集合ではなく正規順序の配列として完全一致を要求し、不一致の場合は値比較モードにかかわらずケース不一致としなければならない（MUST）。

## 14. エンディアンとレーン順

POWERのベクトル要素順とload/store Intrinsicsはコンパイラ・モードの影響を受けるため、次を必須とする。

1. 正規形式のレーン0は、Intrinsic仕様上の論理レーン0を意味する。
2. POWERアダプターは`vec_extract`等の論理要素操作または明示した変換関数でレーンを構築・抽出する。
3. 生の128ビット値を整数として解釈し、ホストエンディアンだけでレーン番号を決定しない。
4. shuffle、permute、pack、unpack、merge、extract、insert、load/storeは「endianness-sensitive」タグを付け、非対称バイトパターンを必須入力とする。
5. コンパイラのベクトル要素順オプションと関連マクロを実行マニフェストへ記録する。

ppc64 big-endianを追加する際は、既存のppc64le結果と単にバイト列比較せず、同じ正規レーンへ変換してから比較する。

### 14.1 256ビット値のPOWER側チャンク規約

初期対象のAVX/AVX2 256ビット値は、POWERアダプター内部で、順序付きの2本の128ビットチャンク`chunk[0]`、`chunk[1]`として表現しなければならない（MUST）。このチャンク表現はアダプター内部に限り、6.1節の共通ABIへ公開してはならない（MUST NOT）。

要素幅を`w`ビット（`w`は8、16、32、64のいずれか）、1チャンク当たりのレーン数を`L = 128 / w`としたとき、正規形式の`lanes[i]`とチャンク内の論理レーンの対応は次で一意に決める。

```text
chunk_index = floor(i / L)
lane_in_chunk = i mod L
chunk[chunk_index].lane[lane_in_chunk] = lanes[i]
```

したがって、`chunk[0]`は`lanes[0]`から`lanes[L-1]`、`chunk[1]`は`lanes[L]`から`lanes[2L-1]`を保持する（MUST）。`chunk[0]`はx86上で256ビット値から取り出した下位128ビット、`chunk[1]`は上位128ビットの論理内容に対応する。この「下位」「上位」は数値として解釈したホストメモリ上のアドレスやPOWERレジスタの表示順を意味しない。

POWERアダプターは、各チャンクについて14節の規則に従い、明示的な構築・抽出関数で正規レーンとの変換を行わなければならない（MUST）。2本のPOWERベクトルを連続メモリへ`memcpy`し、その物理バイト順だけからチャンク順またはレーン順を決めてはならない（MUST NOT）。戻り値、入力値、マスクのいずれにも同じ対応を適用する。

Intrinsicの仕様が演算範囲を各128ビット部分に限定している場合、`chunk[0]`と`chunk[1]`へ独立にその演算を適用する（MUST）。256ビット全体を対象とするpermute、extract、insert等では、Intel Intrinsicの論理インデックスを上式の`chunk_index`と`lane_in_chunk`へ分解して適用する（MUST）。POWER命令の物理要素番号をIntel側の論理インデックスとして直接使用してはならない（MUST NOT）。

フレームワーク自身の既知ベクトルテストは、少なくとも`u8x32`、`u32x8`、`u64x4`について、全レーンが異なる値を使い、正規形式から2チャンクへの変換とその逆変換が恒等であることを検証しなければならない（MUST）。往復変換だけでなく、変換直後に`chunk[0].lane[L-1] == lanes[L-1]`および`chunk[1].lane[0] == lanes[L]`であることを個別に表明し、128ビット境界でのチャンク逆転を検出しなければならない（MUST）。

## 15. 不一致レポートと再現バンドル

比較器は不一致となった`input_id`ごとに、単一入力を両ホストで再実行できる失敗再現バンドルを作成しなければならない（MUST）。単一の入力に複数の差異があってもバンドルは1個とし、`failures/<input_id>/`へ次の構成で格納する。

```text
failures/<input_id>/
├── failure.json
├── test-vectors.jsonl
├── test-vectors.manifest.json
└── baseline/
    ├── intel/
    │   ├── intel-results.jsonl
    │   └── intel-results.manifest.json
    └── openpower/
        ├── power-results.jsonl
        └── power-results.manifest.json
```

`test-vectors.jsonl`は元の入力レコードを1件だけ含む。比較器は`sequence`を`1`へ置換し、その他のフィールドを変更せずRFC 8785で再直列化してLFを付けなければならない（MUST）。8.4節の導出から`sequence`は除外されるため、再計算した`input_id`は元の値と一致しなければならない。一致しない場合は失敗バンドルを公開せず、比較実行を`infrastructure_error`とする。

`test-vectors.manifest.json`は8.5節のスキーマに従い、`record_count: 1`、再直列化後のバイト数とSHA-256、および元の入力マニフェストと同じ`profile`と`case_definitions_sha256`を持たなければならない（MUST）。各`baseline`結果JSONLは、該当する元の結果レコードを1件だけRFC 8785で直列化してLFを付ける。対応する結果マニフェストは12.1節に従い、元マニフェストの`environment`と`runner`を保持したまま、`record_count`、`byte_length`、結果SHA-256、単一入力JSONLのSHA-256を再計算しなければならない（MUST）。したがって、バンドル内の入力および2個の基準結果は、通常のランナーと比較器でそのまま検証できる完全な1レコード成果物となる。

### 15.1 `failure.json`

`failure.json`は再帰的にキーを辞書順へ整列した単一のUTF-8 JSONオブジェクトをRFC 8785で直列化し、末尾にLFを1個付けなければならない（MUST）。`schema_version: 1`では次のフィールドだけを持つ。

```json
{
  "artifact_type": "ioitf.failure",
  "baseline": {
    "intel_manifest": "baseline/intel/intel-results.manifest.json",
    "openpower_manifest": "baseline/openpower/power-results.manifest.json"
  },
  "case_definitions_sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "case_id": "ssse3.shuffle.u8x16.masked",
  "comparison": {"mode": "bit_exact"},
  "first_difference": {"intel": "0x7f", "kind": "return", "lane": 3, "openpower": "0x00"},
  "input_id": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "mismatch_count": 1,
  "reproduce": {
    "intel": ["ioitf", "replay", "--failure", "failure.json", "--role", "intel", "--output", "replay/intel"],
    "openpower": ["ioitf", "replay", "--failure", "failure.json", "--role", "openpower", "--output", "replay/openpower"],
    "verify": ["ioitf", "verify-replay", "--failure", "failure.json", "--intel", "replay/intel/intel-results.manifest.json", "--openpower", "replay/openpower/power-results.manifest.json"]
  },
  "schema_version": 1,
  "source_artifacts": {
    "input_sha256": "1111111111111111111111111111111111111111111111111111111111111111",
    "intel_results_sha256": "2222222222222222222222222222222222222222222222222222222222222222",
    "openpower_results_sha256": "3333333333333333333333333333333333333333333333333333333333333333"
  },
  "test_vectors_manifest": "test-vectors.manifest.json"
}
```

各フィールドへ次の制約を適用する。

- `artifact_type`は`ioitf.failure`、`schema_version`は整数`1`とする。`case_id`と`input_id`は、単一入力レコードおよび両基準結果レコードの値と一致しなければならない（MUST）。
- `case_definitions_sha256`はバンドル内の3個のマニフェストと一致する小文字64桁のSHA-256とする。`source_artifacts`の3値は、バンドル抽出前の入力JSONL、Intel結果JSONL、POWER結果JSONLの各完了マニフェストに記録されたSHA-256を複写する（MUST）。
- `comparison`は該当ケース定義の比較オブジェクト全体を複写する。`mismatch_count`は比較規則に従って異なった原子的な比較位置の総数を表す1以上の整数とする。
- `test_vectors_manifest`と`baseline`のパスは例に示した固定値とし、絶対パス、バックスラッシュ、空のパス要素、`.`、`..`を含めてはならない（MUST NOT）。
- `reproduce.intel`、`reproduce.openpower`、`reproduce.verify`は例に示した引数を同じ順序で持つ文字列配列とする。シェルコマンド文字列へ結合せず、各要素を1個のプロセス引数として扱わなければならない（MUST）。相対パスは`failure.json`を含むディレクトリを基準に解決する。

`first_difference.kind`は`status`、`return`、`buffer`、`fp_exceptions`のいずれかとし、`intel`と`openpower`には差異位置の正規化済みJSON値を格納する。`return`がベクトルの場合は0以上の`lane`を必須とし、スカラーでは`lane`を格納しない。`buffer`では`buffer`にバッファID、`byte_offset`に0以上のバイト位置を必須とする。`status`と`fp_exceptions`では位置フィールドを追加しない。

複数の差異がある場合、比較器は最初の差異を、`status`、戻り値、バッファ、浮動小数点例外の順で選ぶ。戻り値は論理レーン番号の昇順、バッファはバッファIDの辞書順の後にバイトオフセットの昇順で走査する（MUST）。浮動小数点の`first_difference`には生ビット列に加えて、Intel値とPOWER値それぞれの分類、診断用10進値、および適用できる場合は13.2節のULP距離を`diagnostic`オブジェクトへ格納しなければならない（MUST）。この順序と形式により、同じ成果物から常に同じ`first_difference`を生成できなければならない（MUST）。

比較器はすべてのバンドルファイルを一時ディレクトリへ作成して各マニフェストを検証し、参照先ファイルを確定した後に`failure.json`を最後に公開しなければならない（MUST）。`failure.json`がないディレクトリを完成した失敗バンドルとして扱ってはならない（MUST NOT）。

### 15.2 別ホストでの再実行と再現判定

`reproduce.intel`はx86_64ホスト、`reproduce.openpower`はppc64leホストで、同一バイト列の失敗バンドルを使って実行する。`ioitf replay`はSUTを呼び出す前に、バンドル内の全マニフェストとSHA-256、単一入力の`input_id`、ローカルケース定義のSHA-256、`--role`とホストアーキテクチャ、およびローカルランナーの`git_commit`と対応する基準結果マニフェストの`git_commit`が一致することを検証しなければならない（MUST）。不一致または必要ISAの不足時はSUTを呼び出してはならない（MUST NOT）。

検証後、`ioitf replay`はバンドルの単一入力をちょうど1回実行し、`--output`で指定したディレクトリへ12.1節の通常の1レコード結果JSONLと完了マニフェストを出力する。保存済みの基準結果を実行結果として複写してはならない（MUST NOT）。2個の再実行結果を比較ホストへ集めた後、`reproduce.verify`を実行する。

`ioitf verify-replay`は次をすべて満たした場合だけ`reproduced`を報告し、終了コード0を返さなければならない（MUST）。

1. 2個の再実行結果成果物が12.1節の検証を通過し、同じ単一入力SHA-256とケース定義SHA-256を持つ。
2. 各ロールの再実行結果について、`duration_ns`を除く`case_id`、`input_id`、`runner`、`status`、`observed`が対応する基準結果と完全一致する。
3. 再実行結果同士を元のケース定義で比較した`mismatch_count`と`first_difference`が`failure.json`と完全一致する。

いずれかを満たさない場合は終了コードを0以外とし、`not_reproduced`、`invalid_bundle`、`unsupported`、`runner_error`のいずれかと、差異または失敗した検証段階を報告する。再実行時のCPU、コンパイラ、ビルドID、コンパイルオプションが基準マニフェストと異なる場合は、その差を診断へ含める。これらの環境差は結果の完全一致を免除する理由にしてはならない（MUST NOT）。

任意機能として、失敗入力を単純化するshrinkerを提供できる（MAY）。shrinkerの出力は元とは異なる`input_id`を持つ新しい失敗再現バンドルとし、15.2節の両ランナー再実行と`verify-replay`によって不一致を確認しなければならない（MUST）。元のバンドルを上書きしてはならない（MUST NOT）。

## 16. CI連携

### 16.1 必須ジョブ

1. `generate-vectors`
   - ケース定義を検証し、入力JSONLとマニフェストを生成する。
2. `run-x86_64`
   - Intelランナーをビルド・実行し、結果を成果物として公開する。
3. `run-ppc64le`
   - 同じ入力成果物を使用してPOWERランナーをビルド・実行する。
4. `compare-results`
   - 両入力ハッシュを検証し、比較結果とJUnit XMLを生成する。

各実行ジョブは、`generate-vectors`が公開した同一の不変成果物を取得する。各ジョブが独自に乱数入力を再生成してはならない（MUST NOT）。

### 16.2 合否

CIは次の場合に失敗しなければならない（MUST）。

- 1件以上の結果不一致
- 必須ケースの結果欠落または重複
- 必須ケースが`unsupported`
- 入力またはケース定義のハッシュ不一致
- ランナー異常終了
- 比較規則・入力制約の不正
- 必須メタデータの欠落

終了コードは次を推奨する（SHOULD）。

| コード | 意味 |
|---:|---|
| 0 | 全必須ケース一致 |
| 1 | 結果不一致 |
| 2 | 仕様・入力エラー |
| 3 | 実行環境・ISA非対応 |
| 4 | ランナー異常 |

## 17. ビルド要件

- 警告を有効化し、警告をCIでエラーとして扱う（SHOULD）。
- `-fno-strict-aliasing`へ安易に依存せず、型変換は`memcpy`、`std::bit_cast`、または定義済みIntrinsicで行う（SHOULD）。
- 必要ISAフラグをケースまたはターゲット単位で分離する（MUST）。全ソースへ一律に最上位`-mcpu`を指定し、ISA検出前のコードで不正命令を発生させてはならない（MUST NOT）。
- Release相当の最適化ビルドを合否対象とする（MUST）。Debugビルドも補助的に実行する（SHOULD）。
- Sanitizerが利用可能な各アーキテクチャでは、アダプターとフレームワークへASan/UBSanを実行する（SHOULD）。

参考となる典型的フラグ例:

```text
x86_64 AVX2:  -O2 -mavx2 -fno-fast-math
ppc64le P8:   -O2 -mcpu=power8 -maltivec -mvsx -fno-fast-math
```

実際のフラグは採用コンパイラの仕様に合わせ、結果マニフェストへ完全な形で記録する。

## 18. フレームワーク自身のテスト

フレームワーク実装は少なくとも次の自己テストを持たなければならない（MUST）。

- JSONLの正規化と往復変換
- 整数・浮動小数点ビット列の往復変換
- エンディアン変換の既知ベクトル
- 各比較モードのpass/fail境界
- NaN、符号付きゼロ、subnormalの比較
- 入力・結果の欠落、重複、ハッシュ不一致検出
- 不正なケース定義と未定義入力の拒否
- 意図的に誤ったPOWER実装を使った不一致検出
- 途中で停止した結果成果物の拒否
- 再現コマンドによる単一入力の再実行

比較器の期待結果は、アーキテクチャに依存しない単体テストとして両ホストで同じ内容を実行する。

## 19. トレーサビリティ

各移植対象Intrinsicは、次の情報へ追跡可能でなければならない（MUST）。

- Intel Intrinsic名と参照仕様の版
- Intel側ラッパーのソース位置
- POWER側ラッパーのソース位置
- 使用したPower Intrinsicまたは命令
- 意味上の差異と補正方法
- ケース定義ID
- 対応テストプロファイル
- 最終検証日とCI実行URL

`coverage.json`等で、対象Intrinsicの「未登録」「登録済み未実装」「実行済み一致」「実行済み不一致」「対象外」を一覧化することを推奨する（SHOULD）。

## 20. 受け入れ基準

フレームワークの初期版は、以下をすべて満たした時点で受け入れ可能とする。

1. x86_64とppc64leの別ホストで、同一SHA-256の入力成果物を処理できる。
2. 整数演算、浮動小数点演算、shuffle/permuteを各1ケース以上登録できる。
3. `bit_exact`、`ulp`、NaNポリシーを含む比較が動作する。
4. 100件以上の意図的な一致ケースをすべてpassできる。
5. レーン逆転、1ビット誤り、NaNポリシー違反をそれぞれ確実にfailできる。
6. 不一致から単一ケースを再現できる。
7. smokeプロファイルをCIで実行し、JUnit XMLと失敗成果物を取得できる。
8. Intel/POWERのコンパイラ、ISA、エンディアン、浮動小数点モードを結果から確認できる。

## 21. 実装開始前に確定すべき事項

次の項目はケース登録開始前にプロジェクトとして確定する。

- 対象とするIntel Intrinsicsの具体的一覧と最低ISA
- POWER8、POWER9、POWER10のどれを最低実行環境とするか
- 採用コンパイラとサポートバージョン
- 浮動小数点Intrinsicごとの許容差
- signaling NaNを必須対象にするか
- 浮動小数点例外フラグを合否へ含めるか
- 必須CI用ppc64le実機または信頼できるネイティブ実行環境

## 22. 参考仕様

- [Intel Intrinsics Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html)
- [Power ISA Specifications](https://openpowerfoundation.org/specifications/isa/)
- [OpenPOWER Vector Intrinsic Programming Reference Compliance Specification](https://openpowerfoundation.org/compliance/vectorintrinsicprogrammingreference/)
- [GCC PowerPC AltiVec/VSX Built-in Functions](https://gcc.gnu.org/onlinedocs/gcc/PowerPC-AltiVec_002fVSX-Built-in-Functions.html)
- [IBM Open XL C/C++: AltiVec compatibility](https://www.ibm.com/docs/en/openxl-c-and-cpp-lop/17.1.1?topic=compilers-altivec-compatibility)
- [Intel: FTZ and DAZ flags](https://www.intel.com/content/www/us/en/docs/dpcpp-cpp-compiler/developer-guide-reference/2025-0/set-the-ftz-and-daz-flags.html)

参照仕様間で記述が異なる場合は、対象コンパイラと対象ISAの版をケース定義に固定し、最小再現コードによる実機結果を設計レビューへ添付する。
