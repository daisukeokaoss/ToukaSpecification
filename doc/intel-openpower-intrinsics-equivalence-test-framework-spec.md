# Intel Intrinsics / OpenPOWER移植 実装等価性テストフレームワーク仕様書

- 文書ID: IOITF-SPEC-001
- バージョン: 1.1.0-draft
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
- 入力値から実行時にアクセス範囲が変わるgather/scatter等のポインター演算。`schema_version: 1`では7.3節の固定範囲として記述できるケースだけを扱う

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
| 安全整数 | 本仕様のJSON numberとして損失なく交換できる整数。RFC 7493の相互運用可能範囲である`-(2^53 - 1)`以上`2^53 - 1`以下を指す。 |
| 正規u64十進文字列 | 先頭ゼロのない非負整数のJSON文字列。`^(0|[1-9][0-9]*)$`に一致し、10進値が`18446744073709551615`以下であるもの。 |
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

`schema_version: 1`のケース定義、入力・結果JSONL、各マニフェスト、`failure.json`および本仕様が定義するその他のJSON/YAMLデータでは、JSON numberとして表現する値は整数だけとする（MUST）。その整数は`-(2^53 - 1)`以上`2^53 - 1`以下、非負整数フィールドは0以上`9007199254740991`以下でなければならない（MUST）。個別フィールドにより狭い範囲がある場合はそちらを優先する。範囲外の値をbinary64へ変換した後に検査すること、または丸め、飽和、切り捨てによって受理することを禁止する（MUST NOT）。この制約はC ABI内の固定幅整数、16進文字列、正規u64十進文字列には適用しない。正規u64十進文字列はchecked u64演算または任意精度整数で解釈し、浮動小数点型を経由してはならない（MUST NOT）。

本書が`draft`である間、`schema_version: 1`は実装開始前の暫定スキーマであり、文書のdraft版更新によって互換性のない訂正を受けることがある。仕様確定後は、互換性のない変更で`schema_version`を増加させなければならない（MUST）。

本仕様で「辞書順」と記す場合、RFC 8785 §3.2.3と同じく、Unicode文字列をUTF-16 code unit列として比較したlocale非依存の昇順を意味する（MUST）。正規化、case folding、自然順sortまたはロケール照合を行ってはならない（MUST NOT）。配列はJCS自身では並べ替えられないため、生成側がこの順序へ整列し、検証側が順序違反を拒否する。

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
│   ├── shuffle.yaml
│   └── isa-registry.json
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

共通ABIは、固定幅整数、バイト列、長さ、ステータスだけで構成する。`__m128i`、`__m256`、`vector unsigned int`などのネイティブベクトル型をABI境界に含めてはならない（MUST NOT）。これは各ホスト内でランナーとアダプターを接続するC ABIであり、構造体の生バイト列をホスト間で転送する形式ではない。

以下を`IOITF_ABI_VERSION == 1`のC ABIとして固定する。JSON/YAML成果物の`schema_version`とは独立した版であり、C型、フィールド配置、列挙値、またはABIバイト符号化に非互換変更を加える場合は`IOITF_ABI_VERSION`を増加させなければならない（MUST）。実装はこの宣言と同じ型およびフィールド順を持つ共有ヘッダーを使用し、対応対象である64ビットLP64 ABI上で`sizeof`と`offsetof`を静的表明しなければならない（MUST）。構造体へ`packed`属性を付けてはならない（MUST NOT）。

```c
#include <stdint.h>

typedef struct {
    const uint8_t* data;
    uint64_t size;
} ioitf_bytes;

typedef struct {
    uint32_t abi_version;
    uint32_t struct_size;
    ioitf_bytes encoded_arguments;
    uint32_t rounding_mode;
    uint32_t fp_mode;
} ioitf_input;

typedef struct {
    uint32_t abi_version;
    uint32_t struct_size;
    uint8_t* encoded_return_value;
    uint64_t return_capacity;
    uint64_t return_size;
    uint8_t* encoded_memory_effects;
    uint64_t effects_capacity;
    uint64_t effects_size;
    uint32_t normalized_fp_exceptions;
    uint32_t status;
} ioitf_output;

enum {
    IOITF_ABI_VERSION = 1,
    IOITF_ROUND_NEAREST_EVEN = 0,
    IOITF_ROUND_TOWARD_ZERO = 1,
    IOITF_ROUND_TOWARD_POSITIVE = 2,
    IOITF_ROUND_TOWARD_NEGATIVE = 3,
    IOITF_FP_MODE_IEEE = 0
};

enum {
    IOITF_OUTPUT_OK = 0,
    IOITF_OUTPUT_UNSUPPORTED = 1,
    IOITF_OUTPUT_INVALID_INPUT = 2,
    IOITF_OUTPUT_RUNTIME_ERROR = 3
};

enum {
    IOITF_CALL_OK = 0,
    IOITF_CALL_INVALID_ABI = 1,
    IOITF_CALL_OUTPUT_TOO_SMALL = 2,
    IOITF_CALL_RESOURCE_ERROR = 3
};

typedef int (*ioitf_case_fn)(const ioitf_input*, ioitf_output*);
```

対応対象のx86_64 System V ABIとppc64le ELFv2 ABIでは、`sizeof(void*) == 8`かつ`sizeof(uint64_t) == 8`を前提とし、次の配置を静的表明する。1項目でも異なるABIを`IOITF_ABI_VERSION == 1`として実行してはならない（MUST NOT）。

| 型 | `sizeof` | フィールドoffset（バイト） |
|---|---:|---|
| `ioitf_bytes` | 16 | `data=0`, `size=8` |
| `ioitf_input` | 32 | `abi_version=0`, `struct_size=4`, `encoded_arguments=8`, `rounding_mode=24`, `fp_mode=28` |
| `ioitf_output` | 64 | `abi_version=0`, `struct_size=4`, `encoded_return_value=8`, `return_capacity=16`, `return_size=24`, `encoded_memory_effects=32`, `effects_capacity=40`, `effects_size=48`, `normalized_fp_exceptions=56`, `status=60` |

C++でアダプターを実装する場合、公開ケースシンボルを`extern "C"`で宣言・定義し、C++のname manglingをABIへ持ち込んではならない（MUST NOT）。

ケースの公開関数名は、ケース定義の`intel.symbol`または`openpower.symbol`に記録した完全シンボルと一致し、`^[A-Za-z_][A-Za-z0-9_]*$`に一致しなければならない（MUST）。宣言は`int <case_symbol>(const ioitf_input*, ioitf_output*);`に相当する。ケースIDから`ioitf_run_<case_id>`を合成してはならない（MUST NOT）。ケースIDには`.`などC識別子に使えない文字が含まれるためである。

ABIのバイト列は次のように符号化する。

- `encoded_arguments`は、入力レコードの`operands`と、存在する場合だけ`immediates`および`buffers`を同名キーで含む1個のJSONオブジェクトを、RFC 8785で直列化したUTF-8バイト列とする。末尾LF、パディング、BOM、NUL終端を含めない（MUST）。SUTの引数順はJSONオブジェクトのキー順ではなく`signature.arguments`の宣言順とする。
- `rounding_mode`は上記4値、`fp_mode`は`IOITF_FP_MODE_IEEE`だけを許可する。その他の値は不正なABI入力である。
- `encoded_return_value`は12節の`observed.return`に格納するJSON値をRFC 8785で直列化したUTF-8バイト列とする。戻り値が`void`なら`return_size`を0とし、バイトを出力しない。
- `encoded_memory_effects`は12節の`observed.buffers`オブジェクトそのものをRFC 8785で直列化したUTF-8バイト列とする。ポインター引数がなければ`effects_size`を0とし、バイトを出力しない。
- 2個の出力バイト列にも末尾LF、パディング、BOM、NUL終端を含めてはならない（MUST NOT）。

呼び出し側は`ioitf_output`全体をいったんゼロ初期化し、両構造体の`abi_version`を`IOITF_ABI_VERSION`、`struct_size`をそれぞれの`sizeof`へ設定した後、使用する出力バッファのポインターと容量を設定する。容量が0なら対応ポインターはNULLでもよい。容量が1以上なら対応ポインターはNULLであってはならない（MUST）。入力ポインター、出力ポインター、version、size、入力バイト列、列挙値その他のABI自体が不正な場合、アダプターはSUTを呼ばず`IOITF_CALL_INVALID_ABI`を返す。

アダプターはSUT呼び出し前に必要な`return_size`と`effects_size`を決定する。どちらかの容量が不足する場合、必要サイズを両sizeフィールドへ設定し、SUTを呼ばず`IOITF_CALL_OUTPUT_TOO_SMALL`を返さなければならない（MUST）。ランナーは必要量を事前確保するか、この戻り値の後に必要量を確保して同じ入力をちょうど1回だけ再呼び出しできる（MAY）。最初の呼び出しではSUTが実行されていないため、再呼び出しを含めてもSUT実行は1回である。2回目も容量不足になる、または出力領域を確保できない場合は、その入力の`infrastructure_error`とする。

アダプターが8.3節のsymbolic bufferを実メモリへ構築できない場合はSUTを呼ばず`IOITF_CALL_RESOURCE_ERROR`を返す。ランナーはこれをその入力の`infrastructure_error`とする。継続不能なresource failureは11.1節のrunner-attemptへ昇格する。未知のcall戻り値は継続不能なABI protocol failureとする。

`IOITF_CALL_OK`の場合だけ`ioitf_output.status`を解釈する。`IOITF_OUTPUT_OK`では、各sizeは実際に出力した正確なバイト数、`normalized_fp_exceptions`は13.5節のビットマスクでなければならない（MUST）。その他のoutput statusでは両sizeと`normalized_fp_exceptions`を0とする。output statusは順に結果JSONの`ok`、`unsupported`、`invalid_input`、`runtime_error`へ写像する。シグナルはランナーが隔離境界で検出して`signal`へ、ABI・容量・resource・成果物・preflightの失敗は`infrastructure_error`へ写像する。アダプターは`signal`または`infrastructure_error`をoutput statusとして返さない。

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

ビルドはシステムヘッダーを含むヘッダー依存関係ファイルを翻訳単位ごとに生成しなければならない（MUST）。Intel側は、コンパイラが実際に解決したx86 Intrinsicヘッダーの正規パスとSHA-256が、採用コンパイラ版の許可リストと一致し、POWER互換ヘッダーを含まないことを確認する。POWER側は、実際に解決したプロジェクト互換ヘッダーまたは登録済みPOWER互換ヘッダーの正規パスとSHA-256が許可リストと一致し、x86ターゲット用ヘッダーを含まないことを確認する。basenameだけをヘッダーのrole判定または拒否へ使用してはならない（MUST NOT）。GCC rs6000の`emmintrin.h`を直接使用する構成では、そのversionが要求する`-DNO_WARN_X86_INTRINSICS`を明示しないと`#error`になるため、defineを12.1節の該当build unitの`compile_options`と`feature_macros`へ記録する。いずれかのidentity検査を満たさないビルドを拒否しなければならない（MUST）。

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

ケース定義はYAMLまたはJSONで管理し、schema version 1では次のbase fieldと、後述する条件付きfieldだけを持つ閉じたオブジェクトでなければならない（MUST）。

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
tags: []
```

YAMLを使用する場合はYAML 1.2のJSON schemaだけを受理し、anchor、alias、merge key、明示tag、重複key、およびJSONデータモデルにない値を拒否しなければならない（MUST）。YAMLからこのJSONデータモデルへ変換した後に閉じたケーススキーマを検証し、RFC 8785のハッシュを計算する。処理系固有のYAML 1.1暗黙型付けを使用してはならない（MUST NOT）。

base fieldは`schema_version`、`id`、`description`、`intel`、`openpower`、`signature`、`input_domain`、`comparison`、`environment`、`tags`である。`immediates`、`memory_contract`、`regressions`は、それぞれ7.1、7.3、7.4節の条件を満たす場合だけ追加できる。

- `schema_version`は整数1、`id`は`^[a-z][a-z0-9.-]*$`に一致する未使用のcase ID、`description`は空でない文字列とする。
- `intel`と`openpower`は`symbol`と`required_isa`だけを持つ。`symbol`は`^[A-Za-z_][A-Za-z0-9_]*$`に一致する6.1節の公開C symbol、`required_isa`は7.2節の登録tokenを辞書順・重複なしで1件以上持つ配列とする。
- `signature`は`arguments`と`return`だけを持つ。`arguments`は宣言順の配列で、各`name`はC識別子かつケース内で一意とする。`type: scalar`は`name`、`type`、`element`、`vector`または`mask`はさらに正の安全整数`lanes`、`immediate`は`name`、`type`、`element`、`pointer`は`name`と`type`だけを持つ。`element`は8.3節の整数型または`f32`、`f64`とし、immediateだけは7.1節の`u8`または`i8`に限定する。
- `return`は`type: void`なら`type`だけ、`scalar`なら`type`と`element`、`vector`または`mask`ならさらに正の安全整数`lanes`だけを持つ。pointerまたはimmediateをreturn typeにしてはならない（MUST NOT）。
- `input_domain`は`exclude`だけを持ち、schema version 1では`exclude`を空配列に固定する。一般のpredicate言語を定義していないため、未定義入力を即値集合、型、固定memory contractその他の閉スキーマで構造的に排除できないケースは登録してはならない（MUST NOT）。将来predicateを追加する場合は新しいschema versionとする。
- `environment`は`fp_rounding_modes`と`observe_fp_exceptions`だけを持つ。前者は4種の丸めモードを辞書順・重複なしで1件以上持ち、後者は真偽値とする。
- `tags`は`^[a-z][a-z0-9_.-]*$`に一致する文字列の辞書順・重複なし配列とする。14節の対象ケースでは`endianness-sensitive`を必須とする。
- `comparison`は13節のmode別閉スキーマとする。

上記の全objectは列挙したキーだけを持ち、条件外または未知のキーを再帰的に拒否しなければならない（MUST）。

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
tags: []
```

`immediates`のキー集合は`type: immediate`の引数名の集合と完全に一致しなければならない（MUST）。各定義は`values`と`compile_time`だけを持ち、次の制約を満たす。

- `schema_version: 1`では`type: immediate`の`element`を`u8`または`i8`に限定し、その他の要素型を拒否する（MUST）。
- `values`は空でない昇順のJSON整数配列とし、重複を許さない。`u8`では0以上255以下、`i8`では-128以上127以下でなければならない（MUST）。
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

static bool intel_dispatch_mm_shuffle_epi32(__m128i a,
                                             uint32_t imm8,
                                             __m128i *out) {
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

仕様上結果が未定義または不定となる入力は、許容即値集合、型、7.3節の固定memory contractその他の閉じたケースfieldで生成不能かつ受理不能にしなければならない（MUST）。schema version 1で構造的に表せないゼロ除算、範囲外シフト、範囲外インデックスその他の条件付き定義域を持つケースは登録しない。アラインメント要件違反は7.3節でアダプター呼び出し前に拒否する。

### 7.2 ISA語彙レジストリ

`required_isa`は自由記述文字列ではなく、版管理したISA語彙レジストリに登録済みのトークン配列とする（MUST）。schema version 1のレジストリは`schema_version`と`tokens`だけ、各token entryは`architecture`、`detector`、`implies`、`token`だけ、`detector`は`id`と`version`だけを持つ閉じたJSONオブジェクトとする。

```json
{
  "schema_version": 1,
  "tokens": [
    {
      "architecture": "ppc64le",
      "detector": {"id": "linux.auxv.vsx", "version": 1},
      "implies": ["power8"],
      "token": "vsx"
    }
  ]
}
```

`token`と`detector.id`は`^[a-z][a-z0-9_.-]*$`に一致し、`architecture`は`x86_64`または`ppc64le`、`detector.version`は安全な正整数とする。`implies`はより基礎的なtokenを辞書順・重複なしで持つ。`tokens`はtoken昇順とする。トークン名と意味は公開後に変更してはならない（MUST NOT）。意味、implies、architectureまたは検出方法の規範的な変更には新しい版付きtokenまたは新しいschema versionを使用する。

レジストリ全体をRFC 8785で直列化してLFを1個付けたバイト列のSHA-256を`isa_registry_sha256`とする。used ISA contractは、入力JSONLが参照するケースごとの直接要求と、その要求の推移閉包に含まれる完全なtoken entryを次の閉じた形へ射影する。`cases`はID昇順、各role配列と`tokens`はtoken昇順で重複を許さない。

```json
{
  "cases": [
    {
      "id": "avx2.add.i32x8.wrap",
      "intel": ["avx2"],
      "openpower": ["power8", "vsx"]
    }
  ],
  "schema_version": 1,
  "tokens": [
    {
      "architecture": "ppc64le",
      "detector": {"id": "linux.auxv.vsx", "version": 1},
      "implies": ["power8"],
      "token": "vsx"
    }
  ]
}
```

例の`tokens`は説明用に一部を省略している。実ファイルでは、`cases`の直接要求と`implies`推移閉包に現れる全token entryをちょうど1回含めなければならない（MUST）。この完全なused objectをRFC 8785で直列化してLFを1個付けたバイト列のSHA-256を`used_isa_contract_sha256`とする。未使用トークンの追加は`isa_registry_sha256`を変えるが、used objectまたはそのハッシュを変えてはならない（MUST NOT）。入力・結果マニフェストは両方を記録し、再現判定の互換性には後者を使用する。登録されていないトークン、循環した`implies`、対象roleと矛盾するarchitectureを持つケースを拒否する。

`required_isa`は、そのケースを安全に実行する前提条件の宣言であって、特定の機械命令が生成された証明ではない。特定のVSX命令等を使うこと自体が製品要件なら、最終リンク対象の逆アセンブルを別途監査し、その成果物のSHA-256を19節の追跡情報へ記録しなければならない（MUST）。振る舞いテストの成功だけからVSX、VMXその他の命令選択を推定してはならない（MUST NOT）。

### 7.3 ポインター引数のメモリ契約

ポインター引数があるケースは`memory_contract`を持たなければならず、そのキー集合は`signature.arguments`で`type: pointer`とした引数名の集合と完全に一致しなければならない（MUST）。`schema_version: 1`の例を示す。

```yaml
signature:
  arguments:
    - {name: src, type: pointer}
    - {name: dst, type: pointer}
  return: {type: void}
memory_contract:
  src:
    access: read
    required_alignment: 1
    read_ranges: [{offset: 0, byte_length: 16}]
    write_ranges: []
  dst:
    access: write
    required_alignment: 16
    read_ranges: []
    write_ranges: [{offset: 0, byte_length: 16}]
```

各エントリは例に示した4フィールドだけを持つ。`access: read`ではread rangeを1件以上、write rangeを0件、`write`ではreadを0件、writeを1件以上、`read_write`では両方を1件以上持たなければならない（MUST）。`required_alignment`は実効ポインターに対するバイト単位の正の2の累乗とする。各rangeの`offset`は実効ポインターからの非負バイト位置、`byte_length`は正のバイト数とし、いずれも3節の安全整数契約を満たす。各range配列はoffset昇順で、重複または交差してはならない（MUST NOT）。初期schemaでは固定リテラル範囲だけを許し、入力値から動的に範囲を求める式を記述してはならない（MUST NOT）。

ランナーはABI呼び出し前に、8.3節のsymbolic buffer ID、別名関係、offset、および全read/write rangeが宣言バイト長内に収まることを整数演算で検証しなければならない（MUST）。入力が契約に反する場合は`invalid_input`とし、アダプターを呼び出さない。

実メモリの所有者はケースアダプターとする。アダプターは`encoded_arguments`を検証後、buffer IDごとに1回だけ宣言alignmentで割り当て、初期bytesを複写し、同じIDを参照する全pointerを同じ割り当ての`base + operand.offset`として構築する。SUT呼び出し前に、割り当て先頭の実測alignment、実効ポインターの`required_alignment`、整数overflow、および全rangeの実アドレス範囲を再検証しなければならない（MUST）。割り当てまたは実alignmentを満たせない場合は`IOITF_CALL_RESOURCE_ERROR`を返す。ホスト固有ポインターをJSONまたはABIのバイト列へ格納してはならない（MUST NOT）。入力生成器はこれらの不正入力を通常ベクトルとして生成してはならない（MUST NOT）。

SUT実行後は、アダプターが全入力バッファの最終内容を12節の`observed.buffers`形へ符号化する。複数pointerのwrite rangeを対応するbufferの絶対offsetへ解決し、buffer単位で和集合を作る。その和集合の外側は初期内容と完全一致しなければならず（MUST）、read-onlyバッファも同様とする。canaryはこの不変領域の一部として検査する。同じ値を書き戻した操作は最終内容から検出できないため、本仕様は「実際に書いたバイト数」を推測せず、宣言された許可範囲と最終バイト列を比較する。

### 7.4 回帰witnessのオラクル

9.5節で機械検証する回帰witnessを持つケースは、ケース定義へ`regressions`を追加する。`regressions`は回帰IDから`input_id`と`expected_intel`への閉じたオブジェクトであり、回帰IDは`^[a-z][a-z0-9_.-]*\.v[1-9][0-9]*$`に一致しなければならない（MUST）。各entryは次のキーだけを持つ。

```yaml
regressions:
  rounding-toward-zero-inexact.v1:
    input_id: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
    expected_intel:
      status: ok
      observed:
        return: {element: f32, bits: "0x3f800000"}
        fp_exceptions: [inexact]
```

`input_id`は8.4節の小文字64桁SHA-256とする。`expected_intel`は`status`と`observed`だけを持ち、`status`は`ok`に固定する。`observed`は12節の正常結果と同じ条件付きスキーマを使用し、case signature、buffer有無、および`environment.observe_fp_exceptions`と整合しなければならない（MUST）。回帰ID、期待値または対象入力を変更する場合は新しい版の回帰IDを追加し、公開済みentryの意味を変更してはならない（MUST NOT）。`regressions`全体はケース定義および`case_definitions_sha256`のハッシュ対象に含める。

## 8. 正規入力形式

### 8.1 物理形式

初期実装はUTF-8 JSON Lines（JSONL）を使用する（MUST）。BOM、空行、CRLFを許可せず、1行を1テストベクトルとする。各行は[RFC 8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785.html)で直列化したJSONオブジェクトとし、行末にLFを1個付けなければならない（MUST）。JSONの重複キー、RFC 8785で表現できない値、行末以外の空白、および3節の安全整数契約に反するJSON numberを拒否する。

大規模実行でI/Oが問題となる場合は、同じ論理モデルを持つCBOR等を追加できる（MAY）。ただし、同一テスト実行内で形式を混在させてはならない（MUST NOT）。

### 8.2 入力レコード

`schema_version: 1`の入力レコードは、次のフィールドを持たなければならない（MUST）。条件付きフィールドを除く未定義フィールドは拒否する。

| フィールド | JSON型 | 制約 |
|---|---|---|
| `schema_version` | integer | `1` |
| `case_id` | string | ケースレジストリに存在する空でないID |
| `input_id` | string | 8.4節で導出する小文字64桁のSHA-256 |
| `sequence` | integer | ファイル先頭を1とする連続した安全な正整数。行番号と一致 |
| `environment` | object | `fp_mode`と`rounding`を格納 |
| `operands` | object | 即値を除く全引数を引数名で格納 |
| `generation` | object | 入力の生成クラスと疑似乱数由来情報を格納 |
| `immediates` | object | ケース定義に`immediates`がある場合だけ必須 |
| `buffers` | object | ポインター引数がある場合だけ必須 |

`environment`は`fp_mode: ieee`と`rounding`だけを持つ。`rounding`は`nearest_even`、`toward_zero`、`toward_positive`、`toward_negative`のいずれかで、ケース定義の`environment.fp_rounding_modes`に含まれなければならない（MUST）。

`generation.class`は`boundary`、`structured`、`exhaustive`、`random`、`regression`のいずれかとする。`random`の場合だけ`algorithm`と`seed`を追加し、`algorithm`はケース生成時に固定した疑似乱数アルゴリズム名、`seed`は`^0x[0-9a-f]{16}$`に一致する64ビット値とする（MUST）。`regression`の場合だけ`regression_id`を必須とし、7.4節の当該ケースに存在するIDで、entryの`input_id`が入力レコードの`input_id`と一致しなければならない（MUST）。その他のクラスでは`algorithm`、`seed`、`regression_id`を格納してはならない（MUST NOT）。したがって`generation`は、通常クラスでは`class`だけ、randomでは`class`、`algorithm`、`seed`だけ、regressionでは`class`、`regression_id`だけを持つ閉じたオブジェクトである。

`operands`のキー集合は、ケース定義の`signature.arguments`から即値を除いた引数名の集合と一致しなければならない（MUST）。`immediates`のキー集合はケース定義の`immediates`と一致し、各値はJSON整数かつ宣言された`values`のいずれかでなければならない（MUST）。さらに、対応する`element`が`u8`なら0以上255以下、`i8`なら-128以上127以下であることを、値の変換前に検証する。

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

バッファIDは`^[A-Za-z_][A-Za-z0-9_]*$`に一致しなければならない。`alignment`は3節の安全整数契約を満たす正の2の累乗であり、割り当て先頭アドレスに要求するアラインメントをバイト単位で表す。`bytes`は割り当て全体の初期内容を`0x`に続く偶数個の小文字16進数字で表し、空バッファは`0x`とする。ポインター値は`buffer`と安全整数である0以上の`offset`だけを持ち、`offset`は参照先のバイト長以下でなければならない（MUST）。アクセス範囲と実効アドレスのalignmentは7.3節の`memory_contract`に従って検証する。

複数のポインターが同じバッファIDを参照する場合は別名参照、異なるIDを参照する場合は互いに重ならない割り当てとする（MUST）。これ以外の方法でホスト固有アドレスやポインター関係を推測してはならない（MUST NOT）。canaryを含む割り当て全体を`bytes`へ格納し、実行後は13.4節に従って割り当て全体を観測する。

### 8.4 `input_id`の導出と検証

`input_id`は推奨値ではなく、次の手順で必ず導出しなければならない（MUST）。

1. 入力レコードから`input_id`、`schema_version`、`sequence`、`generation`を除く。
2. 残った`case_id`、`environment`、`operands`と、存在する場合は`buffers`、`immediates`を1個のJSONオブジェクトにする。
3. そのオブジェクトをRFC 8785で直列化する。末尾にLFを加えない。
4. 直列化したUTF-8バイト列のSHA-256を計算し、小文字64桁の16進文字列を`input_id`とする。

この導出により、生成方法やJSONL内の位置が異なっても、SUTへ渡す入力と実行環境が同じレコードは同じIDになる。生成器と両ランナーは`input_id`を再計算し、レコード値と一致しない場合はSUTを呼び出さず、入力成果物を仕様・入力エラーとして拒否しなければならない（MUST）。同じ`input_id`を持つレコードを1個の入力成果物へ複数格納してはならない（MUST NOT）。

### 8.5 入力成果物マニフェスト

`generate-vectors`は`test-vectors.jsonl`とともに`test-vectors.manifest.json`を生成しなければならない（MUST）。マニフェストは単一のJSONオブジェクトをRFC 8785でUTF-8へ直列化し、末尾にLFを1個付ける（MUST）。次の例は可読性のため整形しているが、実ファイルに空白や改行を加えてはならない。`schema_version: 1`の構造は次のとおりとする。

```json
{
  "artifact_type": "ioitf.test-vectors",
  "case_definitions_sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "complete": true,
  "isa_registry_sha256": "123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0",
  "profile": "smoke",
  "schema_version": 1,
  "test_vectors": {
    "byte_length": 123456,
    "file": "test-vectors.jsonl",
    "record_count": 1000,
    "sha256": "fedcba9876543210fedcba9876543210fedcba9876543210fedcba9876543210"
  },
  "used_isa_contract_sha256": "23456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef01"
}
```

- `artifact_type`は`ioitf.test-vectors`、`schema_version`は整数`1`、`complete`は真でなければならない（MUST）。`profile`は9.4節のプロファイル名のいずれかとする。
- `case_definitions_sha256`は、その入力JSONLが実際に参照する重複なしのケース定義だけをケースID昇順のJSON配列へ変換し、RFC 8785で直列化して末尾にLFを1個付けたバイト列のSHA-256とする（MUST）。YAML入力はJSONデータモデルへ変換し、重複キーまたはRFC 8785で表現できない値を拒否する。未参照ケースの追加または変更によってこのハッシュを変えてはならない（MUST NOT）。
- `isa_registry_sha256`と`used_isa_contract_sha256`は7.2節で導出した小文字64桁のSHA-256とする（MUST）。未使用トークンだけの追加で前者が異なるローカル環境は、後者と参照トークンの射影が一致すれば互換とする。
- `test_vectors.file`は`test-vectors.jsonl`、`byte_length`はファイルの全バイト数、`record_count`は1以上`9007199254740991`以下かつJSONLの行数と同じ整数、`sha256`は末尾LFを含むファイル全体のSHA-256としなければならない（MUST）。`byte_length`も安全な非負整数、SHA-256値は小文字64桁の16進文字列とする。

トップレベルは例に示した8キーだけ、`test_vectors`は`byte_length`、`file`、`record_count`、`sha256`だけを持つ閉じたオブジェクトとし、その他のキーを拒否しなければならない（MUST）。

生成器はJSONLを一時名へ書いて全レコードと`input_id`を検証し、最終名へ確定してからマニフェストを最後に公開しなければならない（MUST）。両ランナーはSUT実行前にマニフェストのスキーマ、行数、バイト数、SHA-256、ケース定義のSHA-256、およびused ISA contractを検証する。欠落または不一致は`infrastructure_error`として入力成果物全体を拒否し、通常の結果JSONLまたは`ioitf.runner-results`を公開してはならない（MUST NOT）。検証に成功した場合だけ、結果マニフェストの`input_sha256`へ検証済みの`test_vectors.sha256`を複写する（MUST）。

## 9. テストデータ生成

生成器は、次の入力クラスを組み合わせなければならない（MUST）。

### 9.1 決定的境界値

- 符号付き整数: `0`、`1`、`-1`、最小値、最小値+1、最大値-1、最大値
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

### 9.5 丸め・例外の意味論的witness

既定以外の丸めモードを要求する浮動小数点ケースは、その丸めモードの期待結果が`nearest_even`と異なる`generation.class: regression`入力を少なくとも1件持たなければならない（MUST）。`observe_fp_exceptions: true`のケースは、Intrinsicが許容入力で発生させ得て合否対象となる各例外クラスについて、対応するregression入力を持たなければならない（MUST）。これらは6.1節の公開ケースシンボルを通る通常のテストベクトルであり、別のプローブ関数内に同じ演算を再実装してはならない（MUST NOT）。

例えばf32加算の丸めとinexactを同時に識別する入力として、`a = 0x3f800000`（1.0）、`b = 0x33c00000`（1.0における0.75 ULP）を使用できる。`nearest_even`の期待結果は`0x3f800001`、`toward_zero`の期待結果は`0x3f800000`で、どちらも`inexact`が発生する。このwitnessがPOWER側だけで失敗した場合は通常の意味的不一致であり、成果物全体の`infrastructure_error`ではない。

比較器はrole間比較の前に、`generation.class: regression`であるIntel結果の`schema_version`、`case_id`、`input_id`、`runner`および通常の結果スキーマを先に検証する。その後、Intel結果から`status`と`observed`だけを射影したオブジェクトを、7.4節の`expected_intel`と完全一致比較しなければならない（MUST）。`duration_ns`その他の共通結果fieldを射影へ含めない。一致しない場合はPOWER側へ原因を帰属せず、通常の不一致バンドルも生成しない。入力`sequence`昇順で最初に失敗した1件について、次の閉じた`ioitf.reference-error`を比較出力ディレクトリの`reference-error.json`としてRFC 8785 + LFで公開し、比較実行を`reference_error`として失敗させる。例は可読性のため整形しているが、実ファイルはJCSの1行と末尾LFである。すべてのSHA-256は対応する検証済み成果物の値とする。

```json
{
  "artifact_type": "ioitf.reference-error",
  "case_definitions_sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "input_id": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "input_sha256": "1111111111111111111111111111111111111111111111111111111111111111",
  "intel_results_sha256": "2222222222222222222222222222222222222222222222222222222222222222",
  "regression_id": "rounding-toward-zero-inexact.v1",
  "schema_version": 1
}
```

witnessの成功が示すのは、公開ケース経路で観測した意味論がその入力について充足したことだけである。使用された機械命令、VSXからVMXへの降格の有無、または一般の全入力に対する同値性を証明しない。

`reference-error.json`は同一ディレクトリの一時名へ全バイトを書いてcloseした後、ローカルではatomic rename、成果物storeでは完成objectの単一uploadで最後に公開する（MUST）。このファイルが存在する比較出力を正常な比較完了として扱ってはならない（MUST NOT）。

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

初期既定では、丸めモードをround-to-nearest ties-to-evenとし、x86のMXCSRにあるFTZとDAZを直接無効化して読み戻し、gradual underflowが有効であることを確認する（MUST）。POWER側もFPSCRの非IEEEモードおよびVSCRのNJを対象CPUとコンパイラの方法で無効化し、実測して記録する。コンパイラオプションまたはプロセス起動時の既定値だけから状態を推定してはならない（MUST NOT）。異なるモードを検証する場合は別テストベクトルとして明示し、SUT呼び出し前に設定値を読み戻す。

コンパイラによる演算の融合・再関連付け・fast-mathは結果を変え得るため、既定ビルドでは禁止する（MUST）。動的丸めと例外フラグの観測を行う翻訳単位では、採用コンパイラが保証するfenvアクセスを有効にしなければならない（MUST）。基準プロファイルをコンパイラ別に次のように定める。

- GCC: `-fno-fast-math -ffp-contract=off -frounding-math -ftrapping-math`。GCCは`#pragma STDC FENV_ACCESS ON`を実装していないため、pragmaだけを根拠にしてはならない（MUST NOT）。SNaNの例外意味論を必須とする翻訳単位では`-fsignaling-nans`も候補とし、採用するGCC版とtargetで9.5節の回帰witnessを実測してから有効とみなす。
- Clang: `-ffp-model=strict`。同等の個別指定を使う場合は少なくとも`-fno-fast-math -ffp-contract=off -frounding-math -ffp-exception-behavior=strict`を含める。
- Intel oneAPI DPC++/C++ Compiler: `-fp-model=strict -no-ftz`。

これらのフラグはコンパイラ変換の契約であり、実行時レジスタの値とは別に検証する。逆にMXCSR、FPSCR、VSCRのreadbackは制御状態しか証明せず、コンパイラが対象演算を保持して丸め・例外意味論を実現した証明にはならない（MUST NOT）。その意味論は9.5節の公開SUT経路のwitnessで検査する。`-no-ftz`も現在のMXCSR値を設定・保証する指定ではないため、いずれの構成でもSUT呼び出し前の設定とreadbackを省略してはならない（MUST NOT）。FMAそのものを対象とするケースでは、FMAとして明示した実装だけを比較する。

## 11. ランナー要件

各ランナーは次を満たさなければならない（MUST）。

1. 起動時に必要ISAを検出し、非対応ケースを実行しない。
2. 入力成果物の検証と11.1節のpreflightに成功した後、入力をストリーミングで読み、受理した各入力に対して結果を1件出力する。
3. 異常終了したケースを他のケースから可能な範囲で分離する。
4. 入力順序に依存しない結果を生成する。
5. 同一ビルド・同一入力を複数回実行した際に、`duration_ns`を除く`status`と、条件に応じた`observed`または`error.stage`/`error.code`が同じである。
6. 結果の欠落、重複、不明な`input_id`を検出する。
7. `SIGILL`、`SIGSEGV`、`SIGFPE`等をケース失敗として記録するか、少なくとも実行単位の異常として報告する。
8. 結果JSONLを途中まで書いた状態でも、完了マニフェストがなければ完全な成果物として扱わない。

ランナーは、ISA別に別バイナリを生成しても、同一ソースを条件付きコンパイルしてもよい（MAY）。ただし、同じバイナリ内でIntel実装をスカラー参照実装へ置換するなど、実際の移植元Intrinsicを迂回してはならない（MUST NOT）。

### 11.1 ランナーpreflight

ランナーは入力成果物の検証後、結果JSONLを作成する前に、SUTの意味ではなく測定経路自身を検査するpreflightを実行しなければならない（MUST）。probe suiteは、検証済み入力のうちランナーが実装済みと宣言して実行する能力の集合から決定する。

- 丸めprobeは、実行する入力が実際に使用する重複なしの丸めモードを、結果採取と同じ方法でset/readbackする。
- `observe_fp_exceptions: true`の実行入力が1件以上あれば、5例外sticky flagを個別にset/readし、全flagをclear/readする。
- 浮動小数点入力を1件以上実行する場合は、例外トラップが無効であること、およびx86ではMXCSRのFTZ/DAZ、POWERでは採用環境のFPSCR/VSCR非IEEE制御が無効であることをreadbackする。

ランナーがある能力を実装していないと宣言した場合、その能力を必要とする入力は12節の`unsupported`とし、その能力をprobe suiteへ入れない。実装済みと宣言した能力についてset/readbackまたは採取が矛盾した場合だけ、成果物全体のpreflight失敗とする。能力不足と測定経路の故障を混同してはならない（MUST NOT）。ランナーは実装済み能力の追加probeを含めてもよい（MAY）が、上記の関連能力probeを省略してはならない（MUST NOT）。このsuperset規則により、15節の単一入力baselineは元実行の成功preflightをそのまま保持できる。

preflightの正規結果は次の閉じた形とする。`probe_suite`はprobe ID昇順・重複なしの配列で、各entryは`architecture`、`expected`、`id`、`implementation_id`、`version`だけを持つ。`architecture`は`x86_64`または`ppc64le`でランナーと一致し、`id`と`implementation_id`は`^[a-z][a-z0-9_.-]*$`、`version`は安全な正整数とする。`id`末尾の`.vN`と`version: N`は一致しなければならない（MUST）。`expected`は`boolean_controls`、`fp_exception_flags`、`rounding_modes`だけを持つ。`boolean_controls`のキーは`exception_traps_enabled`、`mxcsr_daz`、`mxcsr_ftz`、`fpscr_ni`、`vscr_nj`のうち対象architectureに適用するものだけで、値は期待する真偽値とする。`fp_exception_flags`は13.5節の表順、`rounding_modes`は辞書順の重複なし配列とし、この3個の期待条件のうち少なくとも1個は空でないものとする（MUST）。flag配列は各flagの個別set/readと全flagのclear/read、丸め配列は各modeのset/readbackを要求する。

`probe_suite_sha256`は`probe_suite`配列そのものをRFC 8785で直列化してLFを付けたバイト列のSHA-256であり、検証側は埋め込まれた配列から再計算しなければならない（MUST）。`probes`はsuiteの全probe IDをID昇順でちょうど1回ずつ含み、各要素は`id`と`status`だけを持つ。probeは互いに状態を初期化し、1件が失敗しても残りをすべて実行する。`status`は`passed`または`failed`とする。preflight objectは`probe_suite`、`probe_suite_sha256`、`probes`、`status`だけを持つ。正常結果成果物では全probeと全体のstatusが`passed`でなければならない（MUST）。

```json
{
  "probe_suite": [
    {"architecture": "x86_64", "expected": {"boolean_controls": {}, "fp_exception_flags": ["invalid", "divide-by-zero", "overflow", "underflow", "inexact"], "rounding_modes": []}, "id": "fp-exception-clear-set-read.v1", "implementation_id": "runner.x86-fenv.v1", "version": 1},
    {"architecture": "x86_64", "expected": {"boolean_controls": {"mxcsr_daz": false, "mxcsr_ftz": false}, "fp_exception_flags": [], "rounding_modes": []}, "id": "fp-non-ieee-modes-disabled-readback.v1", "implementation_id": "runner.x86-fenv.v1", "version": 1},
    {"architecture": "x86_64", "expected": {"boolean_controls": {}, "fp_exception_flags": [], "rounding_modes": ["nearest_even"]}, "id": "fp-rounding-set-readback.v1", "implementation_id": "runner.x86-fenv.v1", "version": 1},
    {"architecture": "x86_64", "expected": {"boolean_controls": {"exception_traps_enabled": false}, "fp_exception_flags": [], "rounding_modes": []}, "id": "fp-traps-disabled-readback.v1", "implementation_id": "runner.x86-fenv.v1", "version": 1}
  ],
  "probe_suite_sha256": "3456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef012",
  "probes": [
    {"id": "fp-exception-clear-set-read.v1", "status": "passed"},
    {"id": "fp-non-ieee-modes-disabled-readback.v1", "status": "passed"},
    {"id": "fp-rounding-set-readback.v1", "status": "passed"},
    {"id": "fp-traps-disabled-readback.v1", "status": "passed"}
  ],
  "status": "passed"
}
```

測定経路が矛盾する、必要な状態を設定できない、または読み戻しが期待値と異なる場合、ランナーは通常の結果JSONLと`ioitf.runner-results`マニフェストを公開してはならない（MUST NOT）。代わりに次の`ioitf.runner-attempt`成果物をRFC 8785 + LFで公開する。例は可読性のため整形しているが、実ファイルはJCSの1行と末尾LFである。これは診断成果物の公開が完了したことを表し、`complete: true`はテスト成功を意味しない。

```json
{
  "artifact_type": "ioitf.runner-attempt",
  "case_definitions_sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "complete": true,
  "diagnostic": {"code": "preflight_probe_failed"},
  "failure_probe_id": "fp-rounding-set-readback.v1",
  "failure_stage": "preflight",
  "input_sha256": "fedcba9876543210fedcba9876543210fedcba9876543210fedcba9876543210",
  "outcome": "infrastructure_error",
  "preflight": {
    "probe_suite": [
      {"architecture": "ppc64le", "expected": {"boolean_controls": {}, "fp_exception_flags": ["invalid", "divide-by-zero", "overflow", "underflow", "inexact"], "rounding_modes": []}, "id": "fp-exception-clear-set-read.v1", "implementation_id": "runner.power-fenv.v1", "version": 1},
      {"architecture": "ppc64le", "expected": {"boolean_controls": {"fpscr_ni": false, "vscr_nj": false}, "fp_exception_flags": [], "rounding_modes": []}, "id": "fp-non-ieee-modes-disabled-readback.v1", "implementation_id": "runner.power-fenv.v1", "version": 1},
      {"architecture": "ppc64le", "expected": {"boolean_controls": {}, "fp_exception_flags": [], "rounding_modes": ["nearest_even"]}, "id": "fp-rounding-set-readback.v1", "implementation_id": "runner.power-fenv.v1", "version": 1},
      {"architecture": "ppc64le", "expected": {"boolean_controls": {"exception_traps_enabled": false}, "fp_exception_flags": [], "rounding_modes": []}, "id": "fp-traps-disabled-readback.v1", "implementation_id": "runner.power-fenv.v1", "version": 1}
    ],
    "probe_suite_sha256": "4456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef012",
    "probes": [
      {"id": "fp-exception-clear-set-read.v1", "status": "passed"},
      {"id": "fp-non-ieee-modes-disabled-readback.v1", "status": "passed"},
      {"id": "fp-rounding-set-readback.v1", "status": "failed"},
      {"id": "fp-traps-disabled-readback.v1", "status": "passed"}
    ],
    "status": "failed"
  },
  "runner": {"abi_version": 1, "build_id": "power-runner-2026-09-01.1", "role": "openpower"},
  "schema_version": 1,
  "used_isa_contract_sha256": "23456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef01"
}
```

`ioitf.runner-attempt`は、入力成果物全体または実行全体を継続できない場合の閉じた成果物でもある。全stage共通のbase fieldは`artifact_type`、`complete`、`diagnostic`、`failure_stage`、`outcome`、`runner`、`schema_version`とし、`artifact_type`は`ioitf.runner-attempt`、`complete`は真、`outcome`は`infrastructure_error`、`schema_version`は1に固定する。`runner`は`abi_version`、`build_id`、`role`だけを持ち、`abi_version`は整数`IOITF_ABI_VERSION`、`build_id`は空でない文字列、`role`は`intel`または`openpower`とする。`diagnostic`は`code`だけを持ち、codeは`^[a-z][a-z0-9_.-]*$`に一致する安定識別子とする。

`failure_stage`と条件付きフィールドは次のとおりとする。列挙外のstageまたは条件外のフィールドを拒否しなければならない（MUST）。

- `input_validation`: `preflight`より前の失敗。`input_sha256`、case/used contract hash、`preflight`、`failure_probe_id`を禁止する。候補JSONLのバイト列を読めた場合だけ、その生バイト列のSHA-256を`candidate_input_sha256`として必須にし、ファイル自体を読めなかった場合は同フィールドを禁止する。
- `preflight`: 例のように検証済みの`input_sha256`、`case_definitions_sha256`、`used_isa_contract_sha256`、失敗した`preflight`、`failure_probe_id`を必須とする。`failure_probe_id`はID昇順で最初のfailed probeとする。
- `execution`または`finalization`: 検証済みの3ハッシュと成功した`preflight`を必須とし、`failure_probe_id`と`candidate_input_sha256`を禁止する。前者は全入力の継続実行が不可能になった場合、後者はJSONL確定またはマニフェスト公開を完了できない場合に使用する。

preflight後に回復可能な入力固有エラーが起きた場合は、その入力に通常の`status: infrastructure_error`結果を1件出し、後続入力を継続する。allocator障害、未知のABI call code、出力容量protocol違反その他によって安全に継続できない場合は、途中の結果JSONLを正常成果物として公開せず、`failure_stage: execution`のattemptだけを公開する。入力検証または実行全体の失敗を、空の正常結果や入力ごとの偽の結果群として表してはならない（MUST NOT）。特定Intrinsicを呼ぶ9.5節のwitnessをpreflightへ入れてはならない（MUST NOT）。

attemptの規範ファイル名はrole別のランナー出力ディレクトリ直下の`runner-attempt.json`とする。同一ディレクトリの一時名へJCS + LFを全書込みしてcloseした後、ローカルではatomic rename、成果物storeでは完成objectの単一uploadで最後に公開する（MUST）。`runner-attempt.json`と正常な`*.manifest.json`を同じattemptから同時に公開してはならない（MUST NOT）。CIは正常マニフェストがない場合にこの固定名を探索し、存在すれば全体エラーとして検証する。

## 12. 結果形式

結果JSONLは8.1節と同じRFC 8785 + LFの物理形式を使用する（MUST）。各結果レコードは`schema_version`、`case_id`、`input_id`、`runner`、`status`、`duration_ns`だけを共通フィールドとして持ち、後述する`observed`または`error`を条件付きで追加する。その他のフィールドを拒否しなければならない（MUST）。`runner`は`intel`または`openpower`、`duration_ns`は3節の安全整数契約を満たす0以上の整数とする。

`duration_ns`は診断用メタデータであり、Intel結果とPOWER結果の等価性比較および11節の再実行決定性判定から除外しなければならない（MUST）。

```json
{"case_id":"avx2.add.i32x8.wrap","duration_ns":87,"input_id":"7479d5e1b297269d6df7214a15826a2c6f8658b28b4b61128e7681eec3af0472","observed":{"return":{"element":"i32","lanes":["0x00000000","0x00000000","0x80000000","0x7fffffff","0x00000000","0xffffffff","0xffffffff","0x00000000"]}},"runner":"intel","schema_version":1,"status":"ok"}
```

`status`は次のいずれかとする。

- `ok`: 正常に観測結果を取得
- `unsupported`: 必要ISAまたはコンパイラ機能がない
- `invalid_input`: ケース定義に反する入力
- `signal`: シグナルで停止
- `runtime_error`: ラッパーまたはSUT実行のエラー
- `infrastructure_error`: 入出力、成果物、環境のエラー

`status: ok`では`observed`を必須、`error`を禁止する。`observed`は次の条件付きキーだけを持つ閉じたオブジェクトとする。

- ケースの戻り値が`void`でなければ`return`を必須とし、8.3節とケース定義の型に従う。`void`なら格納してはならない（MUST NOT）。
- 入力に`buffers`があれば`buffers`を必須とする。バッファID集合は入力と完全一致し、各IDの値は`byte_offset`と`bytes`だけを持つオブジェクトとする。`byte_offset`は割り当て全体を格納するschema version 1では整数0に固定する。`bytes`はその割り当て全体の実行後内容を、入力と同じバイト長の`0x`に続く偶数個の小文字16進数字で表す。入力にバッファがなければ格納してはならない（MUST NOT）。ここでの`byte_offset: 0`は格納したsliceの割り当て内開始位置であり、ポインター引数の`offset`ではない。ポインターoffsetと割り当てalignmentは入力側の性質なので結果へ複写しない。
- `environment.observe_fp_exceptions: true`なら`fp_exceptions`を必須とし、13.5節の正規配列を格納する。偽なら格納してはならない（MUST NOT）。

例として、ポインター入力を含む正常結果のメモリ部分は次の形になる。

```json
{"observed":{"buffers":{"buf0":{"byte_offset":0,"bytes":"0x00010203aabbccdd"}}}}
```

`status`が`ok`以外なら`observed`を禁止し、`error`を必須とする。`error`は`stage`と`code`だけを持つ。両方とも`^[a-z][a-z0-9_.-]*$`に一致する安定した識別子とし、人間向けメッセージやホスト依存パスを含めてはならない（MUST NOT）。`stage`は`capability`、`input_validation`、`signal`、`execution`、`runner`のいずれかとし、順に`unsupported`、`invalid_input`、`signal`、`runtime_error`、`infrastructure_error`と組み合わせる。詳細な人間向け診断は別ログへ出力できる（MAY）が、結果の決定性判定へ含めない。

比較時に両statusが`ok`なら値比較へ進む。一方だけが非`ok`、またはstatusが異なる場合はstatus不一致とする。両側が同じ非`ok`でも実装等価性のpassとして数えてはならず、`not_comparable`としてCIを失敗させる（MUST）。特に`unsupported`は不一致成功として数えてはならない（MUST NOT）。必須ケースが`unsupported`の場合、テストスイート全体を環境エラーとする。

### 12.1 結果成果物の完了マニフェスト

各ランナーの結果成果物は、結果JSONLと1個の完了マニフェストJSONで構成しなければならない（MUST）。ファイル名はIntel側を`intel-results.jsonl`と`intel-results.manifest.json`、POWER側を`power-results.jsonl`と`power-results.manifest.json`とする。マニフェストは単一のJSONオブジェクトをRFC 8785でUTF-8へ直列化し、末尾にLFを1個付ける（MUST）。次の例は可読性のため整形している。`schema_version: 1`では次の構造を持たなければならない（MUST）。

```json
{
  "artifact_type": "ioitf.runner-results",
  "case_definitions_sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "complete": true,
  "environment": {
    "architecture": "x86_64",
    "assertions_enabled": false,
    "available_isa": ["avx2", "sse2"],
    "build_units": [
      {
        "assertions_enabled": false,
        "compile_options": ["-O2", "-DNDEBUG", "-mavx2", "-fno-fast-math", "-ffp-contract=off", "-frounding-math", "-ftrapping-math"],
        "compiler": {"name": "gcc", "target_triple": "x86_64-linux-gnu", "version": "15.1.0"},
        "feature_macros": {"NDEBUG": "1", "__AVX2__": "1"},
        "id": "adapter.avx2.add.i32x8.wrap",
        "kind": "adapter",
        "object_sha256": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
        "source_blob_sha256": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb"
      },
      {
        "assertions_enabled": false,
        "compile_options": ["-O2", "-DNDEBUG", "-fno-fast-math", "-ffp-contract=off", "-frounding-math", "-ftrapping-math"],
        "compiler": {"name": "gcc", "target_triple": "x86_64-linux-gnu", "version": "15.1.0"},
        "feature_macros": {"NDEBUG": "1"},
        "id": "runner.core",
        "kind": "runner",
        "object_sha256": "cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc",
        "source_blob_sha256": "dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd"
      }
    ],
    "case_build_units": {"avx2.add.i32x8.wrap": ["adapter.avx2.add.i32x8.wrap", "runner.core"]},
    "cpu_model": "example-cpu",
    "endianness": "little",
    "fp_controls": {"exception_traps_enabled": false, "mxcsr_daz": false, "mxcsr_ftz": false},
    "fp_rounding_modes": ["nearest_even"],
    "git_commit": "0123456789abcdef0123456789abcdef01234567",
    "kernel": "6.x.y",
    "link": {
      "binary_sha256": "eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
      "link_options": [],
      "loaded_libraries": [
        {"path": "/usr/lib64/libc.so.6", "sha256": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff"}
      ]
    },
    "os": "linux"
  },
  "input_sha256": "fedcba9876543210fedcba9876543210fedcba9876543210fedcba9876543210",
  "isa_registry_sha256": "123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0",
  "preflight": {
    "probe_suite": [
      {"architecture": "x86_64", "expected": {"boolean_controls": {}, "fp_exception_flags": ["invalid", "divide-by-zero", "overflow", "underflow", "inexact"], "rounding_modes": []}, "id": "fp-exception-clear-set-read.v1", "implementation_id": "runner.x86-fenv.v1", "version": 1},
      {"architecture": "x86_64", "expected": {"boolean_controls": {"mxcsr_daz": false, "mxcsr_ftz": false}, "fp_exception_flags": [], "rounding_modes": []}, "id": "fp-non-ieee-modes-disabled-readback.v1", "implementation_id": "runner.x86-fenv.v1", "version": 1},
      {"architecture": "x86_64", "expected": {"boolean_controls": {}, "fp_exception_flags": [], "rounding_modes": ["nearest_even"]}, "id": "fp-rounding-set-readback.v1", "implementation_id": "runner.x86-fenv.v1", "version": 1},
      {"architecture": "x86_64", "expected": {"boolean_controls": {"exception_traps_enabled": false}, "fp_exception_flags": [], "rounding_modes": []}, "id": "fp-traps-disabled-readback.v1", "implementation_id": "runner.x86-fenv.v1", "version": 1}
    ],
    "probe_suite_sha256": "3456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef012",
    "probes": [
      {"id": "fp-exception-clear-set-read.v1", "status": "passed"},
      {"id": "fp-non-ieee-modes-disabled-readback.v1", "status": "passed"},
      {"id": "fp-rounding-set-readback.v1", "status": "passed"},
      {"id": "fp-traps-disabled-readback.v1", "status": "passed"}
    ],
    "status": "passed"
  },
  "results": {
    "byte_length": 12345,
    "file": "intel-results.jsonl",
    "record_count": 1000,
    "sha256": "0011223344556677001122334455667700112233445566770011223344556677"
  },
  "runner": {"abi_version": 1, "build_id": "intel-runner-2026-09-01.1", "role": "intel"},
  "schema_version": 1,
  "used_isa_contract_sha256": "23456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef01"
}
```

マニフェストの各値には次の制約を適用する。

- `artifact_type`は`ioitf.runner-results`、`schema_version`は整数`1`、`complete`は真でなければならない（MUST）。
- すべてのSHA-256値は小文字の64桁16進文字列とする。`input_sha256`は末尾改行を含む入力JSONLの全バイト、`results.sha256`は末尾改行を含む結果JSONLの全バイトを対象とする（MUST）。
- `case_definitions_sha256`は8.5節と同じ正規化済みケース定義のSHA-256でなければならない（MUST）。元ファイルの空白、コメント、ファイル列挙順をハッシュへ含めてはならない（MUST NOT）。
- `used_isa_contract_sha256`は入力マニフェストと一致しなければならない（MUST）。`isa_registry_sha256`は実行時に使用したレジストリ全体の監査用provenanceであり、未使用トークンだけの追加による差は値比較またはreplayの拒否条件にしてはならない（MUST NOT）。
- `preflight`は11.1節の正常形と完全一致し、`status: passed`でなければならない（MUST）。preflight失敗時にこのマニフェストを生成してはならない（MUST NOT）。
- `runner`は`abi_version`、`build_id`、`role`だけを持つ。`abi_version`は整数`IOITF_ABI_VERSION`、`role`は`intel`または`openpower`とし、前者は`environment.architecture: x86_64`および`intel-results.jsonl`、後者は`environment.architecture: ppc64le`および`power-results.jsonl`と組み合わせなければならない（MUST）。`build_id`は空でない文字列とする。
- `available_isa`は、実行時の`isa_registry_sha256`に対応するレジストリでroleのarchitectureと一致するtokenのうち、7.2節のdetectorが成功した集合と、その`implies`推移閉包を、文字列の辞書順・重複なし配列として格納する（MUST）。`fp_rounding_modes`は実行した全入力で使用した丸めモードを重複なしの辞書順で格納する。丸めモード名は`nearest_even`、`toward_zero`、`toward_positive`、`toward_negative`のいずれかとする（MUST）。
- `build_units`は最終binaryへlinkした全翻訳単位を`id`昇順で1回ずつ持つ。各unitは例の8キーだけを持ち、`id`は`^[a-z][a-z0-9_.-]*$`、`kind`は`adapter`、`runner`、`sut`、`support`のいずれか、2個のdigestは小文字64桁SHA-256とする。`compile_options`はコンパイラへ渡した引数をシェル結合前の順序で持ち、`compiler`は`name`、`target_triple`、`version`だけを持つ。`feature_macros`はその翻訳単位で実際に定義された追跡対象マクロを、名前からreplacement token列の文字列へ対応付ける。空replacementは空文字列、未定義はキー欠落で表す。
- `case_build_units`は入力成果物が参照するcase IDだけを動的キーとしてちょうど1回持ち、各値は、そのcaseの実行結果へ影響し得る全build unit IDの辞書順・重複なし配列とする。未知unit、未参照caseまたは影響unitの欠落を拒否する（MUST）。`link`は`binary_sha256`、順序を保持した`link_options`、`loaded_libraries`だけを持つ。`loaded_libraries`は、preflight開始から結果ファイル確定までにランナーのプロセスへ一度でもロードされた、メイン実行ファイル以外の全file-backed共有ライブラリを、正規絶対POSIX `path`昇順で重複なく持つ。各要素は`path`とファイル全バイトの`sha256`だけを持つ閉じたobjectとし、静的リンクだけなら空配列とする。
- ppc64leの`environment`はさらに`vector_semantics`を必須とし、`altivec_src_compat`を`gcc`、`xl`、`mixed`、`unknown`のいずれか、`element_reg_order`を`little`、`big`、`unknown`のいずれかとして格納する。`vector_semantics`はこの2キーだけを持つ。x86_64では`vector_semantics`を格納してはならない（MUST NOT）。`-faltivec-src-compat`等の指定は該当build unitの`compile_options`へ、実際の定義済みマクロはそのunitの`feature_macros`へ格納する。前者は比較結果型とスカラー初期化等の互換モードであり、要素順そのものの根拠にしてはならない（MUST NOT）。
- `fp_controls`には全ケースに共通する浮動小数点制御状態を格納する。x86_64では`exception_traps_enabled`、`mxcsr_daz`、`mxcsr_ftz`、ppc64leでは`exception_traps_enabled`、`fpscr_ni`、`vscr_nj`を真偽値で格納しなければならない（MUST）。入力ごとに変わる丸めモードと例外フラグは`fp_controls`へ格納しない。
- `results.file`はパス区切りと`..`を含まない上記のファイル名、`byte_length`は安全な非負整数、`record_count`は1以上`9007199254740991`以下とする。`record_count`は空行を許さない結果JSONLの行数であり、入力JSONLの行数と一致しなければならない（MUST）。
- `environment.os`は`linux`、`architecture`は`x86_64`または`ppc64le`とし、schema version 1ではどちらも`endianness: little`と組み合わせる。`kernel`と`cpu_model`は10節で要求した実測値を表す空でない文字列とする。各build unitの`compiler.name`、`version`、`target_triple`も空でない文字列とする。`git_commit`は小文字の40桁または64桁16進文字列とする（MUST）。`big`その他のOS/architecture/endianness組合せをschema version 1で受理してはならない（MUST NOT）。`environment.assertions_enabled`は全SUT・アダプターunitで`<assert.h>`を処理したときのassert有効性が一致する場合だけその真偽値を格納し、各対象unitの同名fieldと一致させる。`NDEBUG`は全対象unitで統一し、ソース内の`#define NDEBUG`または`#undef NDEBUG`を禁止する（MUST）。一致を証明できないbuildは拒否する。「Release」というビルド名から値を推定してはならない（MUST NOT）。

マニフェストのトップレベル、`environment`、各`build_units`要素とその`compiler`、`link`とその`loaded_libraries`要素、`fp_controls`、`preflight`とその`probe_suite`/`probes`要素、`probe_suite.expected`、`results`、`runner`は例および上記のarchitecture条件で列挙したキーだけを持つ閉じたオブジェクトとする。`case_build_units`のcase ID、各unitの`feature_macros`、および`probe_suite.expected.boolean_controls`だけは規定した動的キーを許す。x86_64の`fp_controls`は`exception_traps_enabled`、`mxcsr_daz`、`mxcsr_ftz`、ppc64leでは`exception_traps_enabled`、`fpscr_ni`、`vscr_nj`だけを持つ。列挙外のキーを再帰的に拒否しなければならない（MUST）。

ランナーは結果JSONLを一時名へ書き、全入力に対する結果を出力してファイルを閉じた後に、行数、バイト数、SHA-256を計算しなければならない（MUST）。結果JSONLを最終名へ確定してから完了マニフェストを作成し、ローカルファイルシステムでは同一ディレクトリ内の一時名からのアトミックなrename、CI成果物ストアでは結果JSONLのアップロード完了後のマニフェストアップロードによって、マニフェストを最後に公開しなければならない（MUST）。`complete: false`のマニフェストを公開してはならない（MUST NOT）。

比較器は、完了マニフェストの欠落、スキーマ違反、ファイル名・行数・バイト数・SHA-256の不一致を`infrastructure_error`として比較前に拒否しなければならない（MUST）。さらに、両マニフェストの`input_sha256`、`case_definitions_sha256`、`used_isa_contract_sha256`がそれぞれ一致し、各結果の`runner`がマニフェストの`runner.role`と一致することを確認する。`isa_registry_sha256`だけの差は診断へ記録する。これらの検証が完了するまで、値の一致件数または不一致件数を報告してはならない（MUST NOT）。

## 13. 比較規則

### 13.1 整数・ビット演算

戻り値が整数要素（`i8`、`i16`、`i32`、`i64`、`u8`、`u16`、`u32`、`u64`）またはマスクであるケースは、`comparison`として`mode: bit_exact`だけを持たなければならない（MUST）。追加フィールド、`ieee_value`、`ulp`、`abs_rel`、`classification`を指定したケース定義を拒否する。整数演算、論理演算、シフト、シャッフル、permute、pack、unpack、merge、extract、insert、およびマスクの結果は、正規化後の全ビットを比較する。Intel仕様上の動作がビットの複写または並べ替えだけであるケースは、戻り値の要素が`f32`または`f64`でも`mode: bit_exact`を指定しなければならない（MUST）。

`return.type: void`のケースも`comparison`として`mode: bit_exact`だけを持つ。戻り値比較は行わず、status、memory contract、bufferおよび条件付きFP例外だけを13.4節・13.5節で比較する。他のcomparison modeまたは追加fieldを指定してはならない（MUST NOT）。

比較器は値比較前に、両結果の`element`、要素幅、レーン数、および16進文字列幅がケース定義と一致することを検証しなければならない（MUST）。不一致は値の不一致ではなく、不正な結果成果物として`infrastructure_error`にする。形式検証後は論理レーン番号の昇順に固定幅の生ビット列を比較し、1ビットでも異なればケース不一致とする。符号付きと符号なしの数値へ変換した値が等しいかどうかは判定に使用しない。

戻り値が`f32`または`f64`であるケースでは、13.2節の`comparison.mode`を浮動小数点戻り値にだけ適用する。同じケースが整数値、マスク、またはメモリ副作用も観測する場合、それらは指定された浮動小数点モードにかかわらず`bit_exact`で比較しなければならない（MUST）。したがって、出力バッファへ格納された浮動小数点値にULPまたは絶対・相対許容差を適用するケースは、型付きメモリ観測を定義していない`schema_version: 1`では登録できない。

`schema_version: 1`は整数結果の無視ビット、無視レーン、比較前マスクまたは一般の入力predicateを定義しない。許容入力に対してIntel仕様が未定義または不定としている出力ビットが1個でもあるケースを登録してはならない（MUST NOT）。入力に依存する未定義域を閉じたcase fieldで構造的に排除できないIntrinsicは初期対象外とする。比較器またはアダプターが結果を0クリアする、要素幅でマスクする、符号拡張する、真偽値へ縮約するなどして不一致を隠してはならない（MUST NOT）。

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
  max_ulps: "1"
  signed_zero: equal
  nan:
    both_nan: equal
    quiet_signaling: match
    payload: ignore
    sign: ignore
```

`max_ulps`は3節の正規u64十進文字列とする。比較器はこれを正確な符号なし64ビット整数として解釈し、JSON numberまたは浮動小数点型へ変換してはならない（MUST NOT）。`abs_tolerance`と`rel_tolerance`は、正規表現`^[0-9]+(\.[0-9]+)?([eE][+-]?[0-9]+)?$`に一致する非負かつ有限の10進文字列とし、比較器はこれを厳密な10進有理数として解釈する。ホストの浮動小数点型へ丸めてから閾値を判定してはならない（MUST NOT）。

`bit_exact`以外のモードは、レーンごとに次の順序で判定しなければならない（MUST）。

1. 片方だけがNaNなら不一致とする。両方がNaNなら`nan`の4フィールドだけで判定し、数値誤差は計算しない。
2. NaNでない無限大は、両方が同じ符号の無限大の場合だけ一致とする。有限値との誤差は計算しない。
3. 両方がゼロなら、`signed_zero: equal`では符号にかかわらず一致、`distinct`では符号が同じ場合だけ一致とする。
4. `classification`では、`zero`、`subnormal`、`normal`の分類が一致し、かつゼロ以外では符号が一致する場合だけ一致とする。
5. `ieee_value`では、残った有限値をIEEE 754生ビット列から厳密な有理数へ変換し、その値が等しい場合だけ一致とする。
6. `ulp`では、要素幅を`n`、生ビット列を符号なし整数`u`、`S = 2^(n-1)`、`M = 2^n - 1`とし、順序キー`K(u)`を、符号ビットが1なら`M - u`、0なら`u | S`として計算する。まず`D = abs(K(a) - K(b))`とし、`signed_zero: equal`かつ両値の符号ビットが異なる場合は`D = max(D - 1, 0)`として、`-0`と`+0`の間を0 ULPに折り畳む。正規u64十進文字列を正確に解釈した`parsed_max_ulps`に対して`D <= parsed_max_ulps`なら一致とする。
7. `abs_rel`では、残った有限値を厳密な有理数`a`、`b`として、`d = abs(a - b)`、`m = max(abs(a), abs(b))`を計算する。`d <= abs_tolerance`または`d <= rel_tolerance * m`の少なくとも一方を満たす場合だけ一致とする。途中計算も厳密な有理数演算で行う。

単純な加減算、ビット操作、比較など、両ISAで同じIEEE演算を表現できる場合は`bit_exact`を既定とする（SHOULD）。近似逆数、近似平方根、変換、異なる融合規則を持つ処理は、移植要件に基づいて閾値と特殊値ポリシーを明示する。

### 13.3 マスク

Intel Intrinsicsが「真」を全ビット1で返す場合、POWER実装も同じレーン幅の全ビット1へ正規化しなければならない（MUST）。真偽だけを比較して上位ビットの誤りを見逃してはならない（MUST NOT）。

### 13.4 メモリ副作用

ポインター引数を持つケースでは、戻り値に加えて次を検査する（MUST）。ケースの`comparison.mode`が`ulp`、`abs_rel`、`ieee_value`、`classification`のいずれでも、メモリは13.1節の`bit_exact`で扱う。

1. 各roleの`observed.buffers`について、ID集合と各バイト長が入力の`buffers`と完全一致することを検証する。不正な形は結果成果物の`infrastructure_error`とする。
2. 7.3節で求めたwrite rangeのbuffer単位の和集合外では、実行後バイトが入力の初期バイトと一致することを各roleで検証する。両roleが同じcanaryを壊していても、この検査を失敗させなければならない（MUST）。これはSUTのメモリ契約違反である。
3. 全バッファの実行後内容をIntelとPOWERの間でビット完全一致比較する。
4. 複数差異の順序はバッファIDの辞書順、次にbyte offsetの昇順とする。

実効ポインターのalignmentとアクセス範囲はSUT呼び出し前に検証し、結果から推測しない。最終バイト列だけから、同値store、write後の復元、実際のwrite回数またはwrite footprintを逆算してはならない（MUST NOT）。レポートでは初期値から異なる範囲を`changed range`と呼び、実store履歴を意味する`written range`と呼んではならない（MUST NOT）。

### 13.5 浮動小数点例外

ケース定義の`environment.observe_fp_exceptions`は真偽値として必須とし、省略を許可しない（MUST）。ケース雛形の既定値は偽とする。真の場合、例外フラグは戻り値やメモリ副作用とは独立した観測結果として合否判定へ含めなければならない（MUST）。偽の場合、ランナーは6.1節の`normalized_fp_exceptions`を0に設定し、結果JSONの`observed`へ`fp_exceptions`を格納してはならない（MUST NOT）。

許容入力と正規化済み浮動小数点環境ごとに、移植ラッパーはIntel Intrinsicが発生させない正規化済み例外を、補助演算によって追加で発生させてはならない（MUST NOT）。部分レーン演算を全レーン演算で代替する場合、未使用レーンを例外を発生させない値へ置換してから演算することは、この結果要件を満たす実装例である。特定の置換手法自体は規範要件ではない。戻り値ビットが一致する既知差異であっても例外フラグの一致を推定してはならず、`observe_fp_exceptions: true`では独立に比較する。

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

各ランナーは11.1節のpreflightで5個の例外を1個ずつ設定し、採取結果が対応する共通ABIビットだけを含むこと、および消去後の採取結果が0であることを確認しなければならない（MUST）。`<fenv.h>`が対象のSIMD状態へ接続されない場合は、MXCSRまたはFPSCRを直接操作する実装を使用する。選択された入力に`observe_fp_exceptions: true`がなく採取機能をビルドしていない場合は、このprobeをsuiteから省略できる（MAY）。採取機能自体がないランナーでは該当ケースを`unsupported`とし、採取機能があると宣言したのにpreflightまたは入力実行時の検証が失敗した場合は`infrastructure_error`とする。

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

比較器は、role間の不一致または13.4節のメモリ契約違反が1件以上ある`input_id`ごとに、単一入力を両ホストで再実行できる失敗再現バンドルを作成しなければならない（MUST）。両roleが同じ不正バイトへ変更してrole間の値だけは一致した場合も対象とする。単一の入力に複数の差異があってもバンドルは1個とし、`failures/<input_id>/`へ次の構成で格納する。

```text
failures/<input_id>/
├── failure.json
├── test-vectors.jsonl
├── test-vectors.manifest.json
├── contracts/
│   ├── case-definitions.json
│   └── isa-used.json
└── baseline/
    ├── intel/
    │   ├── intel-results.jsonl
    │   └── intel-results.manifest.json
    └── openpower/
        ├── power-results.jsonl
        └── power-results.manifest.json
```

`test-vectors.jsonl`は元の入力レコードを1件だけ含む。比較器は`sequence`を`1`へ置換し、その他のフィールドを変更せずRFC 8785で再直列化してLFを付けなければならない（MUST）。8.4節の導出から`sequence`は除外されるため、再計算した`input_id`は元の値と一致しなければならない。一致しない場合は失敗バンドルを公開せず、比較実行を`infrastructure_error`とする。

`contracts/case-definitions.json`は該当する1個の完全なケース定義を1要素JSON配列として、`contracts/isa-used.json`はそのケースの両roleが参照するISAトークンと7.2節の推移閉包をJSONオブジェクトとして、それぞれRFC 8785で直列化してLFを付ける（MUST）。前者からbundle用`case_definitions_sha256`、後者からbundle用`used_isa_contract_sha256`を再計算する。元成果物が複数ケースを含んでいた場合、元のcontract hashをそのまま複写してはならない（MUST NOT）。

`test-vectors.manifest.json`は8.5節のスキーマに従い、`record_count: 1`、再直列化後のバイト数とSHA-256、元のprofile、および上記2個のbundle用contract hashを持たなければならない（MUST）。`isa_registry_sha256`は元入力マニフェストのprovenance値を保持する。各`baseline`結果JSONLは、該当する元の結果レコードを1件だけRFC 8785で直列化してLFを付ける。対応する結果マニフェストは12.1節に従い、元マニフェストの`runner`、成功した`preflight`、`isa_registry_sha256`およびhost/build依存の`environment`フィールドを保持する。ただし入力集合依存の`environment.fp_rounding_modes`はbundle入力の`environment.rounding`だけを含む配列へ、`case_build_units`はbundleの単一case IDだけを持つobjectへ再計算しなければならない（MUST）。`build_units`と`link`は同じbinaryを表す元の値を保持する。2個のbundle用contract hash、`record_count`、`byte_length`、結果SHA-256、単一入力JSONLのSHA-256も再計算する。元マニフェストの`environment`オブジェクト全体を無条件に複写してはならない（MUST NOT）。したがって、バンドル内の入力および2個の基準結果は、通常のランナーと比較器でそのまま検証できる完全な1レコード成果物となる。

### 15.1 `failure.json`

`failure.json`は再帰的にキーを辞書順へ整列した単一のUTF-8 JSONオブジェクトをRFC 8785で直列化し、末尾にLFを1個付けなければならない（MUST）。次の例は可読性のため整形しているが、実ファイルはJCSの1行と末尾LFである。`schema_version: 1`では次のフィールドだけを持つ。

```json
{
  "abi_version": 1,
  "artifact_type": "ioitf.failure",
  "baseline": {
    "intel_manifest": "baseline/intel/intel-results.manifest.json",
    "openpower_manifest": "baseline/openpower/power-results.manifest.json"
  },
  "case_definitions_sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
  "case_id": "ssse3.shuffle.u8x16.masked",
  "comparison": {"mode": "bit_exact"},
  "contracts": {
    "case_definitions": {"file": "contracts/case-definitions.json", "sha256": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"},
    "used_isa": {"file": "contracts/isa-used.json", "sha256": "23456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef01"}
  },
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
    "case_definitions_sha256": "4444444444444444444444444444444444444444444444444444444444444444",
    "input_sha256": "1111111111111111111111111111111111111111111111111111111111111111",
    "input_isa_registry_sha256": "5555555555555555555555555555555555555555555555555555555555555555",
    "intel_isa_registry_sha256": "7777777777777777777777777777777777777777777777777777777777777777",
    "intel_results_sha256": "2222222222222222222222222222222222222222222222222222222222222222",
    "openpower_isa_registry_sha256": "8888888888888888888888888888888888888888888888888888888888888888",
    "openpower_results_sha256": "3333333333333333333333333333333333333333333333333333333333333333",
    "used_isa_contract_sha256": "6666666666666666666666666666666666666666666666666666666666666666"
  },
  "test_vectors_manifest": "test-vectors.manifest.json",
  "used_isa_contract_sha256": "23456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef01"
}
```

各フィールドへ次の制約を適用する。

- `artifact_type`は`ioitf.failure`、`schema_version`は整数`1`、`abi_version`は整数`IOITF_ABI_VERSION`とする。`case_id`と`input_id`は、単一入力レコードおよび両基準結果レコードの値と一致しなければならない（MUST）。
- `case_definitions_sha256`と`used_isa_contract_sha256`はバンドル内の3個のマニフェストおよび`contracts`の対応する`sha256`と一致する小文字64桁のSHA-256とする。`contracts`のfileは例の固定相対パスとする。`source_artifacts`の8値は、バンドル抽出前の入力・結果マニフェストに記録された3個の成果物SHA-256、case/used contract SHA-256、および入力・Intel結果・POWER結果それぞれの`isa_registry_sha256`を複写する（MUST）。
- `comparison`は該当ケース定義の比較オブジェクト全体を複写する。
- `test_vectors_manifest`、`baseline`、`contracts`のパスは例に示した固定値とし、絶対パス、バックスラッシュ、空のパス要素、`.`、`..`を含めてはならない（MUST NOT）。
- `reproduce.intel`、`reproduce.openpower`、`reproduce.verify`は例に示した引数を同じ順序で持つ文字列配列とする。シェルコマンド文字列へ結合せず、各要素を1個のプロセス引数として扱わなければならない（MUST）。相対パスは`failure.json`を含むディレクトリを基準に解決する。

`mismatch_count`は3節の安全整数契約を満たす1以上の整数とする。`first_difference`は`kind`ごとに次のキーだけを持つ閉じたオブジェクトとし、`lane`と`byte_offset`も安全な非負整数とする。

- `status`: `kind`、`intel`、`openpower`
- `return`: `kind`、`intel`、`openpower`に加え、vectorまたはmaskの場合だけ`lane`、浮動小数点の場合だけ`diagnostic`
- `buffer`: `kind`、`buffer`、`byte_offset`、`intel`、`openpower`
- `memory_contract`: `kind`、`runner`、`buffer`、`byte_offset`、`before`、`after`。13.4節の許可範囲外変更を表し、`runner`は`intel`または`openpower`とする
- `fp_exceptions`: `kind`、`intel`、`openpower`

`buffer`差異の`intel`/`openpower`と、`memory_contract`差異の`before`/`after`は、該当する1バイトを`^0x[0-9a-f]{2}$`に一致する文字列で表す（MUST）。

`status`差異の`intel`/`openpower`は12節のstatus文字列とする。`return`差異の`intel`/`openpower`は、scalarまたは該当vector/mask laneのケース定義幅を持つ小文字`0x`生ビット列とする。`fp_exceptions`差異の両値は13.5節の正規配列とする。追加の型情報は埋め込まれたケース定義から得て、`first_difference`へ未定義キーを追加してはならない（MUST NOT）。

`mismatch_count`の原子単位を次で固定する。

1. statusが異なる場合は1とし、`observed`の差を数えない。
2. 両statusが`ok`の場合、戻り値は異なるscalarを1、vectorまたはmaskは比較規則で異なる論理laneごとに1と数える。voidは0とする。
3. メモリ契約違反集合`V`は、許可write rangeの和集合外で初期値から変化した各`(runner, buffer, byte_offset)`を1原子とする。
4. role間バッファ差集合`D`は、最終byteが異なる各`(buffer, byte_offset)`を1原子とする。ただし同じ`buffer`と`byte_offset`についてどちらかのrunnerの`V`が存在する位置は`D`から除外し、同一原因を二重計数しない。
5. 浮動小数点例外は、正規5フラグのうち片方の配列だけに存在する各flagを1原子とする。

両statusが`ok`の場合の`mismatch_count`は、戻り値原子数、`|V|`、`|D|`、例外原子数の和とする。両側が同じ非`ok`なら12節の`not_comparable`であり、`mismatch_count`や通常の失敗再現バンドルを生成しない。

複数の差異がある場合、比較器は最初の差異を、`status`、メモリ契約違反、戻り値、バッファ間差異、浮動小数点例外の順で選ぶ。メモリ契約違反はrunnerを`intel`、`openpower`の順、バッファIDの辞書順、byte offsetの昇順、戻り値は論理レーン番号の昇順、バッファ間差異はバッファIDの辞書順とbyte offsetの昇順で走査する（MUST）。

浮動小数点returnの`diagnostic`は`intel`、`openpower`、および条件付き`ulp_distance`だけを持つ。各roleの値は`classification`と、有限な非ゼロ値の場合だけ`decimal`を持つ。`classification`は`positive_zero`、`negative_zero`、`subnormal`、`normal`、`positive_infinity`、`negative_infinity`、`quiet_nan`、`signaling_nan`のいずれかとする。`decimal`はf32をexactにbinary64へ拡張した後、または元のf64について、RFC 8785 §3.2.2.3が生成するJSON number tokenを文字列として格納する。ゼロ、無限大、NaNでは`decimal`を格納してはならない（MUST NOT）。生ビット列は`first_difference.intel`と`openpower`に保持する。

`diagnostic.ulp_distance`は、`comparison.mode: ulp`で両値が有限かつ両方がゼロではなく、13.2節の手順6を評価した場合だけ必須とし、それ以外で格納してはならない（MUST NOT）。値は13.2節の符号付きゼロ補正後の最終的な`D`を表す正規u64十進文字列とする。例えば符号反転の差異は安全整数上限を容易に超えるため、JSON numberへ格納してはならない（MUST NOT）。この順序と形式により、同じ成果物から常に同じ`mismatch_count`と`first_difference`を生成できなければならない（MUST）。

`failure.json`のトップレベルは例のフィールドだけを持つ。`baseline`は`intel_manifest`と`openpower_manifest`だけ、`contracts`は`case_definitions`と`used_isa`だけ、各contract参照は`file`と`sha256`だけ、`reproduce`は`intel`、`openpower`、`verify`だけ、`source_artifacts`は例に示した8キーだけを持つ閉じたオブジェクトとする。これらを含む全nested objectで列挙外のキーを拒否しなければならない（MUST）。

比較器はすべてのバンドルファイルを一時ディレクトリへ作成して各マニフェストを検証し、参照先ファイルを確定した後に`failure.json`を最後に公開しなければならない（MUST）。`failure.json`がないディレクトリを完成した失敗バンドルとして扱ってはならない（MUST NOT）。

### 15.2 別ホストでの再実行と再現判定

`reproduce.intel`はx86_64ホスト、`reproduce.openpower`はppc64leホストで、同一バイト列の失敗バンドルを使って実行する。`ioitf replay`はSUTを呼び出す前に、バンドル内の全マニフェストとファイルSHA-256、単一入力の`input_id`、埋め込まれた単一ケース定義とused ISA contractのSHA-256、schema version、`failure.json.abi_version`、両baselineの`runner.abi_version`およびローカルの`IOITF_ABI_VERSION`がすべて一致すること、`--role`とホストアーキテクチャ、当該roleの必要ISA、および11.1節のpreflight成功を検証しなければならない（MUST）。さらにローカルISAレジストリから、バンドル内の単一ケースが直接要求するtokenと`implies`推移閉包を7.2節どおり再射影し、そのJSONデータモデルと`used_isa_contract_sha256`がバンドル内のused ISA contractと完全一致しなければならない（MUST）。使用中tokenの欠落、意味、architecture、detectorまたは`implies`の変更を拒否し、不一致または必要ISAの不足時はSUTを呼び出してはならない（MUST NOT）。replayはバンドル内のケース定義を採点契約として使用し、未使用tokenの差を含むローカルレジストリ全体のハッシュをハードゲートにしてはならない（MUST NOT）。

ローカルランナーの`git_commit`、compiler、build ID、compile options、`isa_registry_sha256`が基準と異なる場合は診断へ記録するが、既定のreplayを拒否してはならない（MUST NOT）。これらを拒否条件にすると、無関係なソースまたは未使用ISAトークンの追加だけで旧バンドルを再実行できなくなるためである。完全に同じバイナリだけを許す任意の`--strict-build`を提供する場合は、git commitに加え、ローカル実行ファイルのSHA-256をbaselineの`environment.link.binary_sha256`へ、ローカルのロード対象共有ライブラリのSHA-256 multisetを`environment.link.loaded_libraries`のSHA-256 multisetへ一致させなければならない（MUST）。ライブラリの配置pathだけの差は診断へ残すが、同一digestなら拒否理由にしない。

検証後、`ioitf replay`はバンドルの単一入力をちょうど1回実行し、`--output`で指定したディレクトリへ12.1節の通常の1レコード結果JSONLと完了マニフェストを出力する。保存済みの基準結果を実行結果として複写してはならない（MUST NOT）。2個の再実行結果を比較ホストへ集めた後、`reproduce.verify`を実行する。

`ioitf verify-replay`は次をすべて満たした場合だけ`reproduced`を報告し、終了コード0を返さなければならない（MUST）。

1. 2個の再実行結果成果物が12.1節の検証を通過し、同じ単一入力SHA-256、単一ケース定義SHA-256、used ISA contract SHA-256を持ち、両preflightが成功している。
2. 各ロールの再実行結果について、`duration_ns`を除く`case_id`、`input_id`、`runner`、`status`と、条件に応じた`observed`または`error.stage`/`error.code`が対応する基準結果と完全一致する。
3. 再実行結果同士を元のケース定義で比較した`mismatch_count`と`first_difference`が`failure.json`と完全一致する。

いずれかを満たさない場合は終了コードを0以外とし、`not_reproduced`、`invalid_bundle`、`unsupported`、`runner_error`のいずれかと、差異または失敗した検証段階を報告する。再実行時のCPU、コンパイラ、ビルドID、コンパイルオプションが基準マニフェストと異なる場合は、その差を診断へ含める。これらの環境差は結果の完全一致を免除する理由にしてはならない（MUST NOT）。一方、環境差だけを理由に実行前拒否してもならない（MUST NOT）。

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
- Intel回帰oracle不一致による`reference_error`
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
| 5 | Intel参照経路・回帰oracleエラー |

## 17. ビルド要件

- 警告を有効化し、警告をCIでエラーとして扱う（SHOULD）。
- `-fno-strict-aliasing`へ安易に依存せず、型変換は`memcpy`、`std::bit_cast`、または定義済みIntrinsicで行う（SHOULD）。
- 必要ISAフラグをケースまたはターゲット単位で分離する（MUST）。全ソースへ一律に最上位`-mcpu`を指定し、ISA検出前のコードで不正命令を発生させてはならない（MUST NOT）。
- Release相当の最適化ビルドを合否対象とする（MUST）。Debugビルドも補助的に実行する（SHOULD）。
- 「Release」という構成名だけから`assert`が無効だと推定してはならない（MUST NOT）。`assert`の有効性は対象翻訳単位で`<assert.h>`を処理した時点の`NDEBUG`に依存する。schema version 1では全SUT・アダプター翻訳単位の`NDEBUG`をビルド指定で統一し、ソース内のdefine/undefを禁止して、12.1節の各build unitの`feature_macros`と`assertions_enabled`へ記録する（MUST）。入力妥当性、alignment、容量、アクセス範囲または安全性を`assert`へ委ねず、SUT呼び出し前に検証しなければならない（MUST）。
- Sanitizerが利用可能な各アーキテクチャでは、アダプターとフレームワークへASan/UBSanを実行する（SHOULD）。

参考となる典型的フラグ例:

```text
GCC x86_64 AVX2:  -O2 -mavx2 -fno-fast-math -ffp-contract=off -frounding-math -ftrapping-math
GCC ppc64le P8:   -O2 -mcpu=power8 -maltivec -mvsx -fno-fast-math -ffp-contract=off -frounding-math -ftrapping-math
Clang x86_64 AVX2: -O2 -mavx2 -ffp-model=strict
Intel oneAPI:      -O2 -mavx2 -fp-model=strict -no-ftz
```

実際のフラグは採用コンパイラの仕様に合わせ、結果マニフェストへ完全な形で記録する。

## 18. フレームワーク自身のテスト

フレームワーク実装は少なくとも次の自己テストを持たなければならない（MUST）。

- JSONLの正規化と往復変換
- JSON安全整数の`±(2^53 - 1)`受理と範囲外拒否、および正規u64十進文字列の0・最大値・先頭ゼロ・範囲外検査
- 整数・浮動小数点ビット列の往復変換
- エンディアン変換の既知ベクトル
- 各比較モードのpass/fail境界
- NaN、符号付きゼロ、subnormalの比較
- binary64で安全整数を超える符号反転等のULP距離を、正規u64十進文字列のまま丸めず往復する検査
- 入力・結果の欠落、重複、ハッシュ不一致検出
- preflight失敗時に`ioitf.runner-attempt`だけを公開し、空の正常結果として受理しない検査
- 別名バッファ、write range外変更、同値storeを実write履歴と誤推定しないメモリ契約検査
- 不正なケース定義と未定義入力の拒否
- 意図的に誤ったPOWER実装を使った不一致検出
- 途中で停止した結果成果物の拒否
- 再現コマンドによる単一入力の再実行
- 未使用ケースまたは未使用ISAトークン追加後も旧バンドルをreplayでき、使用中ISAトークンの意味変更は拒否する検査

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
- SUTとアダプターのcommitまたはblob SHA、runner build ID
- 実際に読み込まれた互換ヘッダーの正規パスとSHA-256
- 互換ヘッダーの上流由来がある場合、そのURL、revision、上流内パス
- コンパイラ名・版・target triple・完全な引数、および`NDEBUG`を含む実効feature macro
- 命令選択へ依存する判断では、対象objectまたは逆アセンブル成果物のSHA-256
- 既知差異ID、適用する実装版・ISA・エンディアン・入力条件、両roleの期待結果と例外、および`fixed`または`known_failure`の処置。`not_registered`はcoverage成果物で管理する
- 根拠種別を`normative_specification`または`empirical_observation`として分離し、資料の版・節・取得日、または再現CI URL

これらは`traceability.json`へ保存する。ファイルはRFC 8785 + LFで、トップレベルに`artifact_type: ioitf.traceability`、`entries`、`schema_version: 1`だけを持つ閉じたオブジェクトとする。`entries`はcase ID昇順で、各entryは次のキーだけを持つ。

- `case_id`
- `intrinsic`: `intel_name`と、`reference`（`retrieved_on`、`section`、`url`、`version`）
- `implementation_source`: `intel`と`openpower`。各roleは`adapter`、`kind`、`project_header`、`upstream`だけを持つ。`adapter`は`path`と`blob_sha256`、`project_header`は`path`と`sha256`、`upstream`はnullまたは`path`、`revision`、`url`だけを持つ。`kind`は`adopted_upstream`、`independent`、`native_intrinsic`のいずれかとする
- `power_mapping`: `intrinsics`と`instructions`の重複なし辞書順文字列配列
- `semantic_mapping`: `correction`と`differences`の文字列
- `profiles`: 9.4節の重複なし辞書順配列
- `build`: `intel`と`openpower`。各roleは`build_units`と`runner_build_id`だけを持ち、`build_units`は当該caseが12.1節の`case_build_units`で参照した完全なbuild unit objectをID昇順で複写する
- `instruction_evidence`: 0件以上の配列。各要素は`artifact_sha256`、`artifact_url`、`kind`、`role`、`tool`だけを持ち、`kind`は`object`または`disassembly`、`role`は`intel`または`openpower`、`tool`は`name`と`version`だけを持つ。配列は`role`、`kind`、`artifact_sha256`、`artifact_url`の順の複合キーで昇順とし、同じ複合キーを重複させない
- `known_differences`: `id`昇順の閉じたobject配列。各要素は`applies_to`、`conditions`、`disposition`、`evidence_ids`、`id`、`inputs`だけを持つ。`id`は版付き識別子、`disposition`は`fixed`または`known_failure`とし、`not_registered`は`coverage.json`で表す。`applies_to`は`intel`と`openpower`だけを持ち、少なくとも一方を非nullとする。非nullのrole objectは`build_units`、`endianness`、`fp_controls`、`implementation_ref`、`isa`だけを持つ。`implementation_ref`は`kind`と`sha256`だけを持つ閉じたobjectで、`kind`は`adapter`または`project_header`とする。その`sha256`は、同じroleの`implementation_source`にある、`kind: adapter`なら`adapter.blob_sha256`、`kind: project_header`なら`project_header.sha256`と一致しなければならない（MUST）。`build_units`は対象条件に関与する12.1節の完全なbuild unit objectをID昇順で1件以上持ち、`fp_controls`はarchitecture別の閉スキーマ、`isa`は7.2節のtokenを辞書順・重複なしで持つ。`conditions`は空でない条件説明文字列を辞書順・重複なしで1件以上持つ。`inputs`は`input_id`昇順・重複なしで1件以上持ち、各要素は`expected`と`input_id`だけを持つ。`expected`は`intel`と`openpower`だけを持ち、各roleは12節の結果から共通fieldを除いた射影、すなわち`status: ok`なら`status`と`observed`、非`ok`なら`status`と`error`だけを持つ。`evidence_ids`は同entryの`evidence.id`を辞書順・重複なしで1件以上参照する
- `evidence`: 各要素が`id`、`kind`、`locator`、`retrieved_on`、`sha256`、`url`、`version`だけを持つ`id`昇順配列。`kind`は`normative_specification`または`empirical_observation`とする
- `last_verified`: `date`と`ci_url`

日付は`YYYY-MM-DD`、SHA-256は小文字64桁、URLは取得可能な`https` URLとする。`adapter.path`はリポジトリ相対POSIX path、`project_header.path`はビルド時に解決した正規絶対POSIX path、`upstream.path`は固定revision内のリポジトリ相対POSIX pathとする。`upstream.revision`は小文字40桁または64桁の不変なfull commit IDに限定し、`upstream.url`もbranch先頭ではなく同じcommitを直接指さなければならない（MUST）。全nested objectで上記以外のキーを拒否する（MUST）。`traceability.json`自身のSHA-256をCIレポートから参照しなければならない（MUST）。`implementation_source`は実際のproject headerと任意のupstream sourceを分け、GCC互換ヘッダーを参考にした事実と、プロジェクトのSUTがそのコードを実際に採用した事実を混同してはならない（MUST NOT）。objectまたは逆アセンブルに依存する判断ではSHA-256だけでなく、取得先URLと生成toolの名前・版を必須とする。

`coverage.json`等で、対象Intrinsicの「未登録」「登録済み未実装」「実行済み一致」「実行済み不一致」「対象外」を一覧化することを推奨する（SHOULD）。

### 19.1 GCC rs6000互換ヘッダーを由来とする既知差異

この節はフレームワーク一般または全POWER実装の性質ではない。対象プロジェクトの実際の互換ヘッダーが、追跡したGCC `gcc/config/rs6000/emmintrin.h`の該当実装を採用・転記していることをパス、SHA-256、上流revisionで確認した場合だけ適用する（MUST）。単に同名Intrinsicがあることを根拠に適用してはならない（MUST NOT）。

- `_mm_comi*_sd` / `_mm_ucomi*_sd`: revisionを固定した当該GCCヘッダーには、POWER向けGCCがorderedとunorderedで同じ比較を生成するためCOMIとUCOMIを同じ実装にしている、というFIXMEがある。trap無効環境でもordered COMIはQNaNで`invalid`例外フラグを立てる必要があるが、対象実装がその差を満たすかはbackendとオプションにも依存する。固定したcompiler、ISA、オプションでQNaN/SNaNの戻り値と`invalid`を実測した後にだけ既知失敗として登録し、許容差で隠してはならない（MUST NOT）。閲覧用のmoving branch URLを根拠revisionとして記録してはならず（MUST NOT）、19節の証拠ではcommitを固定する。
- `_mm_load_pd` / `_mm_store_pd`: 対象実装が`assert`と`vec_ld`/`vec_st`を使用し、対象SUT翻訳単位でassertが無効であることを12.1節の記録で確認し、さらに最終命令が正確に`LVX`/`STVX`であり対象Power ISA版で実効アドレス下位4ビットを無視することを逆アセンブルと規範資料で確認した構成に限り、アクセス可能な整列済み領域に対する誤整列ポインターが整列切下げ位置を読み書きし得る。assert有効時は先にassertion failureとなる。どちらの場合もメモリ保護等によるfaultを否定しない。本仕様では誤整列入力を7.3節でSUT呼び出し前に拒否し、この実装差を許容差にしてはならない（MUST NOT）。
- `_mm_cvttpd_epi32`: 対象版が飽和命令`xvcvdpsxws`を使用する場合、例外trapを無効にしたときの変換対象2レーンの戻り値ビットでは、真の負方向overflow、`-infinity`、QNaN、SNaNはIntelのinteger indefinite `0x80000000`と一致する一方、正方向overflowと`+infinity`はPOWER側`0x7fffffff`、Intel側`0x80000000`となる。これは戻り値ビットだけの分類であり、例外フラグ、trap、SNaN副作用の同値性を意味しない。Intel側で結果が定義された入力なので`input_domain.exclude`で隠さず、補正、既知失敗、またはIntrinsic未登録のいずれかとする（MUST）。境界回帰入力には少なくとも`+2147483648.0`、`+infinity`、`-2147483648.0`（正確に表現可能）、`-2147483648.5`（toward-zero後は表現可能）、`-2147483649.0`（真の負方向overflow）、`-infinity`、QNaN、SNaNを別々に含め、戻り値と例外を独立に検証する。

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

- [RFC 7493: The I-JSON Message Format](https://www.rfc-editor.org/rfc/rfc7493.html)
- [RFC 8785: JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785.html)
- [Intel Intrinsics Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html)
- [Power ISA Specifications](https://openpowerfoundation.org/specifications/isa/)
- [OpenPOWER Vector Intrinsic Programming Reference Compliance Specification](https://openpowerfoundation.org/compliance/vectorintrinsicprogrammingreference/)
- [GCC PowerPC AltiVec/VSX Built-in Functions](https://gcc.gnu.org/onlinedocs/gcc/PowerPC-AltiVec_002fVSX-Built-in-Functions.html)
- [GCC rs6000 `emmintrin.h`（閲覧用moving branch。証拠はcommit固定）](https://github.com/gcc-mirror/gcc/blob/master/gcc/config/rs6000/emmintrin.h)
- [GCC Optimize Options](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html)
- [GCC Floating-point implementation](https://gcc.gnu.org/onlinedocs/gcc/Floating-point-implementation.html)
- [Clang Compiler User's Manual: Controlling Floating Point Behavior](https://clang.llvm.org/docs/UsersManual.html#controlling-floating-point-behavior)
- [IBM Open XL C/C++: AltiVec compatibility](https://www.ibm.com/docs/en/openxl-c-and-cpp-lop/17.1.1?topic=compilers-altivec-compatibility)
- [Intel: FTZ and DAZ flags](https://www.intel.com/content/www/us/en/docs/dpcpp-cpp-compiler/developer-guide-reference/2025-0/set-the-ftz-and-daz-flags.html)
- [Intel oneAPI DPC++/C++ Compiler: `ftz` / `no-ftz`](https://www.intel.com/content/www/us/en/docs/dpcpp-cpp-compiler/developer-guide-reference/2024-0/ftz-qftz.html)
- [Intel oneAPI DPC++/C++ Compiler: `fp-model`](https://www.intel.com/content/www/us/en/docs/dpcpp-cpp-compiler/developer-guide-reference/2025-1/fp-model-fp.html)

参照仕様間で記述が異なる場合は、対象コンパイラと対象ISAの版をケース定義に固定し、最小再現コードによる実機結果を設計レビューへ添付する。
