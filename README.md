# S7 过程值超限判断子过程

对一个 REAL 类型的过程值做 HH / H / L / LL 四级超限判断，适用于不是来自 AI 通道的数值，例如通过 485/Modbus 读到的仪表值。AI 通道请用 AI_Proc，它在同样的超限判断之外还负责原始值到工程值的转换。

## 说明

- 用于博途的SCL源码：Limit_Proc(portal).scl
- 用于Step7的SCL源码：Limit_Proc(step7).scl
- 用于PCS7的SCL源码：Limit_Proc(pcs7).scl

## 处理逻辑

- 超限：PV > HH_limit 置 HH，PV > H_limit 置 H，PV < L_limit 置 L，PV < LL_limit 置 LL；每级须由对应的 `enable_XX` 允许
- 互斥：同一时刻最多一个超限标志，HH 优先于 H，LL 优先于 L
- 死区：高侧超限在 PV 回落到 `限值 - dead_zone` 以下才恢复，低侧超限在 PV 回升到 `限值 + dead_zone` 以上才恢复
- 容错时间：`FT_time`（毫秒）不为 0 时，超限条件持续 `FT_time` 才置位标志，恢复条件持续 `FT_time` 才复位
- 限值顺序：已允许的限值须满足 `LL ≤ L ≤ H ≤ HH`，否则置 `SP_error`
- 取消判断：`invalid`、`SP_error` 为真或四级都未允许（`no_limit`）时，所有超限标志清零
- 溢出：`overflow_SP`、`underflow_SP` 为相对量程（`span - zero`）的比例，默认 0.02、-0.02，即超出量程上/下限 2% 时置 `overflow`/`underflow`；溢出不影响超限判断
- 每次超限时，把当时的 PV 记录到对应的 `HH_PV`/`H_PV`/`L_PV`/`LL_PV`

## 调用示例

调用时，先建立对应的背景数据块，比如"LIT001"，调用时所有参数可省略，直接操纵背景块。

注：下方_name_表示具体变量

1. 在 TIA Portal 中调用示例：

    ```Pascal
    "LIT001"(
             enable_HH:=TRUE,           // 允许高高限判断
             enable_H:=TRUE,            // 允许高限判断
             enable_L:=TRUE,            // 允许低限判断
             enable_LL:=TRUE,           // 允许低低限判断
             invalid:=FALSE,            // 过程值无效，为真时取消超限判断
             PV:=_real_in_,             // 过程值
             zero:=0.0,                 // 量程低值
             span:=100.0,               // 量程高值
             overflow_SP:=0.02,         // 上溢出比例
             underflow_SP:=-0.02,       // 下溢出比例
             HH_limit:=80.0,            // 高高限设定值
             H_limit:=70.0,             // 高限设定值
             L_limit:=2.0,              // 低限设定值
             LL_limit:=1.0,             // 低低限设定值
             dead_zone:=0.5,            // 死区 （赋值0.0时无死区）
             FT_time:=0,                // 容错时间 (单位毫秒 赋值0时无容错时间)
             HH_flag=>_bool_out_,       // 高高超限标志
             H_flag=>_bool_out_,        // 高超限标志
             L_flag=>_bool_out_,        // 低超限标志
             LL_flag=>_bool_out_,       // 低低超限标志
             no_limit=>_bool_out_,      // 四级超限判断都未允许
             input_ok=>_bool_out_,      // 输入有效，即 NOT invalid
             overflow=>_bool_out_,      // 高溢出
             underflow=>_bool_out_,     // 低溢出
             SP_error=>_bool_out_,      // 限值设置错误
             HH_PV=>_real_out_,         // 最近一次高高超限时的过程值
             H_PV=>_real_out_,          // 最近一次高超限时的过程值
             L_PV=>_real_out_,          // 最近一次低超限时的过程值
             LL_PV=>_real_out_);        // 最近一次低低超限时的过程值
    ```

1. 在 Step7 V5.5 中SCL调用示例：

    ```Pascal
    Limit_Proc.LIT001(
        enable_HH           := TRUE,                     // 允许高高限判断
        enable_H            := TRUE,                     // 允许高限判断
        enable_L            := TRUE,                     // 允许低限判断
        enable_LL           := TRUE,                     // 允许低低限判断
        invalid             := FALSE,                    // 过程值无效，为真时取消超限判断
        PV                  := _real_in_,                // 过程值
        zero                := 0.0,                      // 零点值
        span                := 100.0,                    // 量程值
        overflow_SP         := 0.02,                     // 上溢出比例 默认0.02
        underflow_SP        := -0.02,                    // 下溢出比例 默认-0.02
        HH_limit            := 100.0,                    // 高高限设定值
        H_limit             := 100.0,                    // 高限设定值
        L_limit             := 0.0,                      // 低限设定值
        LL_limit            := 0.0,                      // 低低限设定值
        dead_zone           := 0.5,                      // 死区 (赋值0.0时无死区)
        FT_time             := 0);                       // 容错时间 (单位毫秒 赋值0时无容错时间)
    _bool_out_ := LIT001.HH_flag;                        // 高高超限标志
    _bool_out_ := LIT001.H_flag;                         // 高超限标志
    _bool_out_ := LIT001.L_flag;                         // 低超限标志
    _bool_out_ := LIT001.LL_flag;                        // 低低超限标志
    _bool_out_ := LIT001.no_limit;                       // 四级超限判断都未允许
    _bool_out_ := LIT001.input_ok;                       // 输入有效
    _bool_out_ := LIT001.overflow;                       // 高溢出
    _bool_out_ := LIT001.underflow;                      // 低溢出
    _bool_out_ := LIT001.SP_error;                       // 限值设置错误
    _real_out_ := LIT001.HH_PV;                          // 最近一次高高超限时的过程值
    ```

1. 在 Step7 V5.5 中STL调用示例：

    ```Pascal
    CALL  "Limit_Proc" , "LIT001"(
        enable_HH           := TRUE,              // 允许高高限判断
        enable_H            := TRUE,              // 允许高限判断
        enable_L            := TRUE,              // 允许低限判断
        enable_LL           := TRUE,              // 允许低低限判断
        invalid             := FALSE,             // 过程值无效，为真时取消超限判断
        PV                  := _real_in_,         // 过程值
        zero                := 0.0,               // 零点值
        span                := 100.0,             // 量程值
        overflow_SP         := 0.02,              // 上溢出比例 默认0.02
        underflow_SP        := -0.02,             // 下溢出比例 默认-0.02
        HH_limit            := 100.0,             // 高高限设定值
        H_limit             := 100.0,             // 高限设定值
        L_limit             := 0.0,               // 低限设定值
        LL_limit            := 0.0,               // 低低限设定值
        dead_zone           := 0.5,               // 死区 (赋值0.0时无死区)
        FT_time             := L#0,               // 容错时间 (单位毫秒 赋值0时无容错时间)
        HH_flag             := _bool_out_,        // 高高超限标志
        H_flag              := _bool_out_,        // 高超限标志
        L_flag              := _bool_out_,        // 低超限标志
        LL_flag             := _bool_out_,        // 低低超限标志
        no_limit            := _bool_out_,        // 四级超限判断都未允许
        input_ok            := _bool_out_,        // 输入有效
        overflow            := _bool_out_,        // 高溢出
        underflow           := _bool_out_,        // 低溢出
        SP_error            := _bool_out_,        // 限值设置错误
        HH_PV               := _real_out_,        // 最近一次高高超限时的过程值
        H_PV                := _real_out_,        // 最近一次高超限时的过程值
        L_PV                := _real_out_,        // 最近一次低超限时的过程值
        LL_PV               := _real_out_);       // 最近一次低低超限时的过程值
    ```
