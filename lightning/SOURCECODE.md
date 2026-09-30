# 脚本源码（通用版 · lightning）

> 本文件是 lightning 版的「脚本源码」：把转换流水线全部脚本集中于此，便于随 SKILL.md 一起分发与重建。
> 通用版不含读者画像定制脚本（collect_profile.py / apply_profile.py）——那是 thunder 完整版的增量。
> 重建：把下面每个代码块存成同名文件放到 `scripts/`，`chmod +x md2a5.sh` 即可。

## md2a5.sh

```bash
#!/bin/bash
# Markdown → A5 可打印 DOCX（彩色语义版）
# 用法: bash md2a5.sh 输入.md [输出.docx]
# 原理: pandoc 搬运，AI 不重新打字，所以 ⇌ α ρ ₂ ₃ 一个字符都不掉
#
# 步骤（顺序不能换）：
#   0) prep_md.py      预处理：中文引号成对 + Unicode 上下标→^x^/~x~
#                      （不做的话宋体打印丢字；第10章漏过 275 处）
#   0b) audit_numbers.py 数值审计（实验性，默认关闭；AUDIT=1 启用，只报不改）
#   1) 检查 a5-ref.docx，没有就用 make_ref.py 现场重建（只需 pandoc）
#   2) pandoc 搬运 → 上下标变 Word 原生 vertAlign
#   3) fix_tables.py 修表格宽度/列宽/边框/表头底色
#   4) colorize.py   语义上色（颜色=功能，正文保持近黑）
#   5) add_pagenum.py 注入页脚页码（居中 X / Y）
#   5b) normalize_order.py schema 顺序归一化（兜底 pandoc / fix_tables 的顺序漂移）
#
# 注意：转换在 /tmp 里做，完成后 cp 回目标位置并验证文件存在（换环境常踩）
set -e
IN="$1"
[ -z "$IN" ] && { echo "用法: bash md2a5.sh 输入.md"; exit 1; }
OUT="${2:-${IN%.md}_A5打印版.docx}"

# --- 路径桥接（MSYS / Git Bash 专用，Linux 上自动跳过）---
# 症状：Windows 版 python3 收到 POSIX 路径 /c/Users/... 会当成 C:\c\Users\... → 找不到文件。
# 所以凡是**要交给 python3 或 pandoc 的绝对路径**，一律先转 Windows 形态。
# Linux / macOS 上没有 cygpath，towin 原样返回，行为不变。
towin() { if command -v cygpath >/dev/null 2>&1; then cygpath -w "$1"; else echo "$1"; fi; }

# --- 真实终端探测 ---
# 【踩过·Git Bash】/dev/tty 这个设备节点**永远存在**，但非交互进程打不开它。
# 旧写法 `[ -e /dev/tty ]` 会把它误判成"有终端"→ 进交互分支 → read 失败报
# "No such device or address" → 判 N → **整个转换中止**。
# 而 SKILL.md 写明本意是"无 tty 时（AI 跑）只醒目告警不中止"。
# 所以这里真的去打开一次，打得开才算有终端。
have_tty() { (exec 3</dev/tty) 2>/dev/null; }

bash_dir="$(towin "$(cd "$(dirname "$0")" && pwd)")"

# --- 0) 预处理：引号 + 上下标语法 ---
if [ -f "$bash_dir/prep_md.py" ]; then
  TMP="$(towin "$(mktemp -d)")"
  cp "$IN" "$TMP/prepped.md"
  python3 "$bash_dir/prep_md.py" "$TMP/prepped.md"
  WORK="$TMP/prepped.md"
else
  echo "  ⚠️ 缺 prep_md.py，跳过预处理（打印可能丢上下标）"
  WORK="$IN"
fi

# --- 0b) 数值审计（实验性，默认关闭）---
# 该脚本误报率较高且会拖慢流水线，v1 默认不跑。
# 需要时用 AUDIT=1 启用（仅报告不改）；AUDIT_STRICT=1 则发现可疑即中止。
if [ -n "$AUDIT" ]; then
  if [ -f "$bash_dir/audit_numbers.py" ]; then
    echo "  ↳ 数值审计..."
    if ! python3 "$bash_dir/audit_numbers.py" "$WORK"; then
      echo ""
      if [ -n "$AUDIT_STRICT" ]; then
        echo "✗ AUDIT_STRICT 模式：数值有可疑，已中止。改 md 再跑。"
        exit 1
      elif have_tty; then
        printf "⚠️ 数值审计发现可疑。仍要继续生成 docx 吗? [y/N] "
        read -r ans < /dev/tty 2>/dev/null || ans="N"
        [ "$ans" != "y" ] && [ "$ans" != "Y" ] && { echo "已中止。"; exit 1; }
      else
        echo "⚠️⚠️ 数值审计发现可疑（见上）——非交互环境，已继续生成。"
        echo "     请检查上面的告警，确认是误报就没问题，否则改 md 后重跑。"
      fi
    fi
  else
    echo "  ⚠️ 缺 audit_numbers.py，跳过数值审计"
  fi
else
  echo "  （数值审计默认跳过；需要时用 AUDIT=1 启用）"
fi

# --- 1) 确保 A5 模板存在 ---
if [ ! -f "$bash_dir/a5-ref.docx" ]; then
  echo "  ↳ 未找到 a5-ref.docx，现场重建..."
  python3 "$bash_dir/make_ref.py" "$bash_dir/a5-ref.docx"
fi

# --- 2) pandoc 搬运（在 /tmp 里做，避免目标目录权限/挂载问题） ---
STAGE="$(towin "$(mktemp -d)")/stage.docx"
pandoc "$WORK" -o "$STAGE" --reference-doc="$bash_dir/a5-ref.docx"

# --- 3) 4) 表格 + 配色 ---
python3 "$bash_dir/fix_tables.py" "$STAGE"
python3 "$bash_dir/colorize.py"   "$STAGE"
python3 "$bash_dir/add_pagenum.py" "$STAGE"
# --- 5b) schema 顺序归一化（兜底 pandoc/fix_tables 的顺序漂移，如 numPr→pStyle、tblPr 的 jc 错位）---
python3 "$bash_dir/normalize_order.py" "$STAGE"

# --- cp 回目标位置并验证（不做验证 = 换环境时静默失败） ---
cp "$STAGE" "$OUT"
[ -s "$OUT" ] || { echo "✗ 生成失败：$OUT 不存在或为空"; exit 1; }

echo "✓ 已生成: $OUT  (A5 148×210mm)"
```

## prep_md.py

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
prep_md.py —— md 预处理（pandoc 之前跑，必须）

做两件事，都是"不做就会出事"的：
  1. 中文引号成对修复
  2. Unicode 上下标 -> pandoc 语法 ^x^ / ~x~

为什么必须有第 2 步：
  宋体对 Unicode 上标区（⁻⁰¹²³⁴⁵⁶⁷⁸⁹）字形覆盖不全，打印会丢字。
  物理第10章实测漏转 275 处，全是这个风险。
  pandoc 只认 ^x^ / ~x~ 语法，遇到现成的 Unicode 上下标会原样放过
  （实测确认：⁻⁶ 直接进 docx，不转 vertAlign）。

  写作规范是"直接用 ^x^ / ~x~"，本脚本是兜底——
  扫到残留的 Unicode 就自动转，扫到旧的（已转过的）就跳过。

用法: python3 prep_md.py 输入.md [-o 输出.md]
      不给 -o 就原地改（先备份 .bak）
"""
import re, sys, shutil, argparse

# ---- Unicode 上下标字符表 ----
SUP = {'⁰':'0','¹':'1','²':'2','³':'3','⁴':'4','⁵':'5',
       '⁶':'6','⁷':'7','⁸':'8','⁹':'9','⁺':'+','⁻':'-',
       'ⁿ':'n','ⁱ':'i','⁴':'4'}
SUB = {'₀':'0','₁':'1','₂':'2','₃':'3','₄':'4','₅':'5',
       '₆':'6','₇':'7','₈':'8','₉':'9','₊':'+','₋':'-',
       'ₐ':'a','ₑ':'e','ₒ':'o','ₓ':'x'}
ALL_SUP = set(SUP) | set('ᵃᵇᶜᵈᵉᶠᵍʰⁱʲᵏˡᵐⁿᵒᵖʳˢᵗᵘᵛʷˣʸᶻ')
ALL_SUB = set(SUB) | set('ₕₖₗₘₙₚₛₜᵤᵥ')

def conv(m, table):
    return ''.join(table.get(c, c) for c in m.group(0))

def fix_quotes(t):
    """中文引号成对修复：数个数，奇数就把落单的按位置补成左/右"""
    # 已配对的先不动，只处理落单的
    t = t.replace('\u201c', '"').replace('\u201d', '"')
    out, stack = [], []
    for ch in t:
        if ch == '"':
            if stack:
                out.append('\u201d'); stack.pop()      # 右引号
            else:
                out.append('\u201c'); stack.append(1)  # 左引号
        else:
            out.append(ch)
    # 收尾还有没闭合的，说明是漏了右引号
    if stack:
        out.append('\u201d')
    return ''.join(out)

# $…$ 原生公式区段：里面的 ^ / ~ 是 LaTeX 语法（如 10^{2}、v_{0}），
# 不是 pandoc 的上下标标记，绝不能参与转义 / 保护 / 转换。
_MATH_SEG = re.compile(r'\$[^$\n]{1,300}\$')


def _split_math(t):
    """切成 [普通段, 公式段, 普通段, 公式段, …] 的交替列表。"""
    parts = _MATH_SEG.split(t)
    segs = _MATH_SEG.findall(t)
    out = []
    for i, seg in enumerate(parts):
        out.append(seg)
        if i < len(segs):
            out.append(segs[i])
    return out


def convert_sup_sub(t):
    """Unicode 上下标 -> pandoc 语法。**公式段原样放过**。"""
    out, n0 = [], 0
    for i, seg in enumerate(_split_math(t)):
        if i % 2 == 1:                      # 公式段：原样放过，只补统计
            out.append(seg)
            n0 += len(re.findall(r'[\u2070-\u209f\u00b2\u00b3\u00b9]', seg))
        else:
            s, n = _convert_plain(seg)      # n 已含该段的统计，别在外面再加
            out.append(s)
            n0 += n
    return "".join(out), n0


def _convert_plain(t):

    """Unicode 上下标 -> pandoc 语法。已经是 ^x^/~x~ 的不动。"""
    n0 = len(re.findall(r'[\u2070-\u209f\u00b2\u00b3\u00b9]', t))

    # 先把已经是语法形态的 ^...^ / ~...~ 保护起来，避免二次处理
    guard = {}
    def _guard(m):
        key = f'\x00{len(guard)}\x00'
        guard[key] = m.group(0)
        return key
    t = re.sub(r'\^[^\s^]{1,20}\^', _guard, t)
    t = re.sub(r'~[^\s~]{1,20}~', _guard, t)

    # Unicode 上标连续串 -> ^x^
    t = re.sub(r'[\u2070\u00b9\u00b2\u00b3\u2074-\u2079\u207a\u207b]+',
               lambda m: '^' + ''.join(SUP.get(c, c) for c in m.group(0)) + '^', t)
    # Unicode 下标连续串 -> ~x~
    t = re.sub(r'[\u2080-\u2089\u208a\u208b]+',
               lambda m: '~' + ''.join(SUB.get(c, c) for c in m.group(0)) + '~', t)

    # 还原保护区
    for k, v in guard.items():
        t = t.replace(k, v)
    return t, n0

CJK_RE = re.compile(r'[\u3000-\u303f\u4e00-\u9fff\uff00-\uffef]')
_BS = chr(92)          # 反斜杠（不在源码里写裸反斜杠，免得转义层数看错）
_P_TILDE = chr(4)      # 占位：已判定要转义的波浪号
_P_CARET = chr(5)      # 占位：已判定要转义的脱字符

def _escape_plain(t):
    """把"不属于合法上下标对"的 ~ 和 ^ 转义（前面加反斜杠），防止 pandoc 静默配错对。

    【为什么必须有·本机实测】
      范围号是正常写法（电压 3~5V、第 2~3 章），但 pandoc 的 subscript 扩展会把两个
      孤立的 ~ 配成一对，把中间的内容整段搬进下标。实测：
        '电压3~5V至2~3A之间。'         → '5V至2' 被吞成下标
        '半径r^(3/2)与r^{3/2}对比。'  → '(3/2)与r' 被吞成上标
      这种损坏特别阴：做完之后字面 ~ 残留数是 0，残留检查反而判为"正常"。

    【判据】
      ~  夹在数字之间的（1.5~2.5 / 3~5V / 20~30℃）→ 必是范围号，转义
      ~  成对且内容无空格、无中文的            → 合法下标，保留
      ^  成对且内容无空格、无中文的            → 合法上标，保留
      （上标内容出现中文必是误配对；中文下标按规范应平排，故也不保留）
      其余孤立的 ~ / ^ → 一律转义成字面符号（可见的符号，好过静默改内容）
    """
    guard = {}

    def _g(m):
        key = chr(1) + str(len(guard)) + chr(1)
        guard[key] = m.group(0)
        return key

    def _ok(seg):
        return not CJK_RE.search(seg)

    # ① 范围号：夹在数字之间的 ~（必须在"保护合法对"之前做，否则会被当成合法下标）
    t = re.sub(r'(?<=\d)~(?=\d)', _P_TILDE, t)
    # ② 保护合法下标对 ~...~
    t = re.sub(r'~[^\s~]{1,20}~', lambda m: _g(m) if _ok(m.group(0)[1:-1]) else m.group(0), t)
    # ③ 保护合法上标对 ^...^
    t = re.sub(r'\^[^\s\^]{1,20}\^', lambda m: _g(m) if _ok(m.group(0)[1:-1]) else m.group(0), t)
    # ④ 其余孤立的 ~ / ^ 一律转义
    t = t.replace('~', _P_TILDE).replace('^', _P_CARET)
    # ⑤ 还原保护区，占位符换回转义写法
    for k, v in guard.items():
        t = t.replace(k, v)
    return t.replace(_P_TILDE, _BS + '~').replace(_P_CARET, _BS + '^')


def escape_stray_marks(t):
    """按「公式段 / 普通段」切开，只让普通段走转义逻辑。

    ⚠️ 为什么不在函数内部用占位符保护公式：
    试过（guard + len(dict) 编号），在长文档上会串位——几百个公式时某个占位符
    被还原成别处的 ^2^，公式报废且不报错。分段处理没有中间态，也就不会串。"""
    out = []
    for i, seg in enumerate(_split_math(t)):
        out.append(seg if i % 2 == 1 else _escape_plain(seg))
    return "".join(out)

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument('src')
    ap.add_argument('-o', dest='out', default=None)
    a = ap.parse_args()

    t = open(a.src, encoding='utf-8').read()
    orig = t
    t = fix_quotes(t)
    t, n_uni = convert_sup_sub(t)
    t = escape_stray_marks(t)
    n_esc = t.count(_BS + '~') + t.count(_BS + '^')

    dst = a.out or a.src
    if dst == a.src and t != orig:
        shutil.copy(a.src, a.src + '.bak')
    open(dst, 'w', encoding='utf-8').write(t)

    # 计数时排除已转义的（\~ / \^），否则范围号会被误算成下标语法
    n_sup = len(re.findall(r'(?<!\\)\^[^\s\^]{1,20}(?<!\\)\^', t))
    n_sub = len(re.findall(r'(?<!\\)~[^\s~]{1,20}(?<!\\)~', t))
    print(f"✓ 预处理完成")
    print(f"  Unicode 上下标残留转换: {n_uni} 处")
    print(f"  现有上标语法 ^x^: {n_sup} 处   下标语法 ~x~: {n_sub} 处")
    if n_esc:
        print(f"  孤立 ~ / ^ 已转义: {n_esc} 处（范围号等，防 pandoc 误配成上下标）")
    print(f"  → pandoc 会转成 Word 原生 vertAlign，打印不丢字")
    if n_sup + n_sub < 10:
        print(f"  ⚠️ 警告：物理/化学一章正常应有 100+ 个上下标，"
              f"现在只有 {n_sup+n_sub} 个 —— 可能漏写或漏转")

main()
```

## audit_numbers.py

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
audit_numbers.py —— 数值自洽审计（转换前跑，只报不改）

为什么必须有：
  物理第9章 [1.4] 出过 1000 倍错——算式按 10^-9^ 算、电荷量却写成 10^-6^，
  两处是同一次生成里先后写的，中间隔了几百行。
  **靠回代检验抓不住**——回代是文内自己验自己，错的链条内部自洽。

做法：
  把 md 里所有 `A = B = C = D` 的数值等式链抽出来，逐节机器重算。
  相邻两节都算得出值、且相对误差 > 1% 的，报出来。

限制（要说清楚）：
  - 只审"能求值的数值等式"，`E = F/q` 这种字面式跳过（没数字）
  - 单位字母自动剥离，但**量纲不查**——1 m 和 1 cm 在它眼里都是 1
  - 误报难免，设计成"只报可疑，人来判断"，不自动改

用法: python3 audit_numbers.py 输入.md
退出码: 0=干净  1=有可疑（可用来卡流水线）
"""
import re, sys, warnings, math

# ---------- 归一化：把 md 里的数学写法变成 Python 能算的 ----------
def normalize(s):
    # 【必须第一步】剔除 [N.M] 编号锚点 / 章节号。
    # 不剔除的话 [1.4] 会留下 1.4，跟后面的 (9e9) 拼成 "1.4 (9e9)"
    # → Python 判为函数调用 → eval 失败 → 整条链跳过 = 静默漏审。
    # 实测：文档里几乎每个算式都带编号前缀，不修则九成算式漏审。
    s = re.sub(r'\[\d+(?:[.\-]\d+)*\]', ' ', s)   # [1.4] [2.3.1] [S1]
    s = re.sub(r'\^(-?[\d]+(?:\^\^?)?)\^', r'**(\1)', s)   # 10^-6^ -> 10**(-6)
    s = re.sub(r'\^\^\^?', '', s)
    s = s.replace('\u00d7', '*').replace('x', '*').replace('\u00b7', '*')
    s = s.replace('\u00f7', '/')
    # Unicode 上下标（万一没走 prep）
    for u, d in zip('⁰¹²³⁴⁵⁶⁷⁸⁹⁻', '0123456789-'):
        s = s.replace(u, d)
    s = s.replace('**(', '**(')
    # 【踩过】尾数必须整段抓，且不许从数字中间起匹配。
    # 旧写法 `(\d)` 在 "1.6*10**(-19)" 上会抓 6，产出 "1.(6e-19)" → 求值失败 → None
    # → 不进比较 → **已知的数值错静默通过**（失败模式⑫）。
    # 例：5 × 1.6×10^-19^ = 8×10^-18^ （写成 -18，实为 -19，10 倍错）改前抓不到。
    s = re.sub(r'(?<![\d.])(\d+(?:\.\d+)?)\s*\*\s*10\s*\*\*\s*\(?(-?\d+)\)?', r'(\1e\2)', s)  # 9*10**(9) -> (9e9)；1.6*10**(-19) -> (1.6e-19)
    # 补回隐式乘号：科学计数法加括号后 (9e9) (1e-9) 会被当函数调用
    s = re.sub(r'\)\s*\(', ')*(', s)
    s = re.sub(r'\)\s*(?=[\d\(])', ')*', s)
    s = re.sub(r'(\d)\s*\(', r'\1*(', s)
    return s

def strip_units(tok):
    """从片段里抠出'像算式'的部分：只留数字 运算符 括号 空白 e √

    √ 必须留下（否则 √(2×0.8/10) 会被剥成 0.16，与真值 0.4 硬比 → 误报）。
    """
    return re.sub(r'[^0-9eE\.\*\/\+\-\(\)\s√]', '', tok)

ALLOWED = re.compile(r'^(?:[0-9eE\.\*\/\+\-\(\)\s]|sqrt)+$')

# 片段里出现这些中文标点，说明它不是"纯数值等式链的一段"，而是夹着叙述的句子。
# 实测踩过：'…大 10^-4^ 倍：F电 = 2×10^-3^ N，G = …' 被按 = 切段后，
# 10^-4 与 2×10^-3 两个不相干的量硬比 → 95% 误报。
PROSE_PUNCT = re.compile(r'[，：；、]')

def evaluate(tok):
    if PROSE_PUNCT.search(tok):
        return None
    t0 = normalize(tok)
    # 符号的数值下标（v~0~ / t~1~ / F~合~）不是独立数字，必须连符号名一起丢掉。
    # 实测踩过：v~0~ + at 剥单位后剩 "0"，与右边的 14 硬比 → 100% 误报。
    t0 = re.sub(r'(?<=[A-Za-z\u4e00-\u9fff])~[^~]*~', '', t0)
    # 字母与数字粘连的 token（2as / m/s2）是符号式，不是纯数值式 → 整段弃掉。
    # 先把科学计数法 1.6e-19 挖掉，免得把 'e' 也当成变量字母。
    probe = re.sub(r'\d[eE][+-]?\d+', '', t0)
    for w in probe.replace('**', ' ').split():
        if re.search(r'[A-Za-z]', w) and re.search(r'\d', w):
            return None
    t = strip_units(t0).strip()
    if not t:
        return None
    # 根号：剥单位后 √ 还在，换成 sqrt() 交给 eval（eval 环境里注入 math.sqrt）
    t = re.sub(r'√\s*\(', 'sqrt(', t)
    t = re.sub(r'√\s*(\d+(?:\.\d+)?)', r'sqrt(\1)', t)
    if not ALLOWED.match(t):
        return None
    # 去掉孤立运算符结尾
    t = t.rstrip('*/+-').strip()
    if not t or len(t) < 1:
        return None
    # 必须含数字
    if not re.search(r'\d', t):
        return None
    try:
        with warnings.catch_warnings():
            # 含根号/变量的片段会退化成 X(Y) 形态，触发无意义告警，结果已被 try 兜住
            warnings.simplefilter('ignore')
            v = eval(t, {'__builtins__': {}}, {'sqrt': math.sqrt})
        return float(v) if isinstance(v, (int, float)) else None
    except Exception:
        return None

def audit(text):
    problems = []
    # 按行扫，避免跨行误拼
    for lineno, line in enumerate(text.split('\n'), 1):
        # 只处理含 = 且含数字的行
        if '=' not in line or not re.search(r'\d', line):
            continue
        # 去掉 markdown 表格分隔、代码块标记
        if line.strip().startswith('|') or line.strip().startswith('```'):
            pass
        # 按 = 切链（排除 == 、>= 、<= 、!= ）
        parts = re.split(r'(?<![<>=!])=(?!=)', line)
        if len(parts) < 2:
            continue
        vals = [evaluate(p) for p in parts]
        for i in range(len(vals) - 1):
            a, b = vals[i], vals[i + 1]
            if a is None or b is None:
                continue
            if a == 0 and b == 0:
                continue
            denom = max(abs(a), abs(b))
            if denom == 0:
                continue
            rel = abs(a - b) / denom
            if rel > 0.01:   # 1% 容差，容纳四舍五入
                problems.append({
                    'line': lineno,
                    'left': parts[i].strip()[-60:],
                    'right': parts[i + 1].strip()[:60],
                    'va': a, 'vb': b, 'rel': rel,
                    'raw': line.strip()[:110],
                })
                break   # 一条链只报第一节，避免雪崩
    return problems

def main():
    if len(sys.argv) < 2:
        print(__doc__); sys.exit(2)
    src = sys.argv[1]
    text = open(src, encoding='utf-8').read()
    probs = audit(text)

    print(f"=== 数值审计: {src} ===")
    if not probs:
        print("✓ 未发现数值不自洽（已检查所有可求值的数值等式链）")
        sys.exit(0)

    print(f"⚠️ 发现 {len(probs)} 处可疑：\n")
    for p in probs:
        print(f"[第 {p['line']} 行] 相对误差 {p['rel']*100:.1f}%")
        print(f"  左值 = {p['va']:.6g}   ← {p['left']}")
        print(f"  右值 = {p['vb']:.6g}   ← {p['right']}")
        print(f"  原文: {p['raw']}")
        print()
    print("说明：只报可疑，量纲不查、单位剥离，可能有误报——逐条人工确认。")
    sys.exit(1)

main()
```

## make_ref.py

```python
#!/usr/bin/env python3
# 从零重建 a5-ref.docx（A5 参考模板）
#
# 为什么需要这个：a5-ref.docx 是二进制文件，没法以纯文本传递。
# 有了这个脚本，只要有 pandoc，就能在任何机器上重建出一模一样的模板，
# 整套工具链因此变成"纯文本可搬运"的。
#
# 原理：
#   1) 从 pandoc 自带的默认参考文档起步（pandoc 自带，不需要额外传文件）
#   2) 把页面尺寸改成 A5（148×210mm）+ 页边距
#   3) 套用本脚本内置的 A5 样式配置（页边距 / 字体 / 颜色基线）
#
# 用法: python3 make_ref.py [输出路径]
#       默认输出 ./a5-ref.docx
import sys, os, re, zipfile, shutil, subprocess, tempfile

# ---------- A5 版面参数（twips，1mm = 56.7twips）----------
A5_W, A5_H = "8391", "11906"        # 148mm × 210mm
MARGIN = "680"                       # 12mm 页边距（A5 版心约 124mm）
HEADER_F = "720"

# ---------- 字体（Windows 自带，用户机器上真实存在）----------
SERIF = "宋体"
SANS  = "微软雅黑"
MONO  = "Consolas"

C_TITLE = "1F3864"; C_HEAD = "2E5496"; C_GREY = "404040"
BG_SOFT = "F2F2F2"; BG_CODE = "FAFAFA"

def spacing(line=None, before=None, after=None, rule="auto"):
    a = []
    if before is not None: a.append(f'w:before="{before}"')
    if after  is not None: a.append(f'w:after="{after}"')
    if line   is not None: a.append(f'w:line="{line}" w:lineRule="{rule}"')
    return f'<w:spacing {" ".join(a)}/>'

def set_pPr(body, xml):
    m = re.search(r'<w:pPr>(.*?)</w:pPr>', body, re.S)
    inner = m.group(1) if m else ""
    inner = re.sub(r'<w:spacing[^>]*/>', '', inner)
    inner = re.sub(r'<w:ind[^>]*/>', '', inner)
    new = f'<w:pPr>{xml}{inner}</w:pPr>'
    if m: return body[:m.start()] + new + body[m.end():]
    r = re.search(r'<w:rPr[ >]', body)
    if r: return body[:r.start()] + new + body[r.start():]
    return body + new

def set_rPr(body, fonts=None, sz=None, color=None, bold=None, italic=None, bg=None):
    m = re.search(r'<w:rPr>(.*?)</w:rPr>', body, re.S)
    inner = m.group(1) if m else ""
    for pat in [r'<w:rFonts[^>]*/>', r'<w:sz[^>]*/>', r'<w:szCs[^>]*/>',
                r'<w:color[^>]*/>', r'<w:shd[^>]*/>']:
        inner = re.sub(pat, '', inner)
    inner = re.sub(r'<w:b/>', '', inner); inner = re.sub(r'<w:i/>', '', inner)
    add = ""
    if fonts:
        ascii_, ea = fonts
        add += f'<w:rFonts w:ascii="{ascii_}" w:hAnsi="{ascii_}" w:eastAsia="{ea}" w:cs="{ascii_}"/>'
    if bold:   add += '<w:b/>'
    if italic: add += '<w:i/>'
    if sz:     add += f'<w:sz w:val="{sz}"/><w:szCs w:val="{sz}"/>'
    if color:  add += f'<w:color w:val="{color}"/>'
    if bg:     add += f'<w:shd w:val="clear" w:color="auto" w:fill="{bg}"/>'
    new = f'<w:rPr>{add}{inner}</w:rPr>'
    if m: return body[:m.start()] + new + body[m.end():]
    return body + new

def set_borders(body, left=None, top=None, bottom=None):
    m = re.search(r'<w:pBdr>(.*?)</w:pBdr>', body, re.S)
    inner = m.group(1) if m else ""
    for pat in [r'<w:top[^>]*/>', r'<w:left[^>]*/>', r'<w:bottom[^>]*/>']:
        inner = re.sub(pat, '', inner)
    add = ""
    if left:   add += f'<w:left w:val="single" w:sz="{left[0]}" w:space="8" w:color="{left[1]}"/>'
    if top:    add += f'<w:top w:val="single" w:sz="{top[0]}" w:space="4" w:color="{top[1]}"/>'
    if bottom: add += f'<w:bottom w:val="single" w:sz="{bottom[0]}" w:space="4" w:color="{bottom[1]}"/>'
    bdr = f'<w:pBdr>{add}{inner}</w:pBdr>'
    if m: return body[:m.start()] + bdr + body[m.end():]
    pm = re.search(r'<w:pPr>', body)
    if pm: return body[:pm.end()] + bdr + body[pm.end():]
    return body + bdr

CFG = {
    "Normal": dict(fonts=("Times New Roman", SERIF), sz=21,
        ppr=spacing(line=340, after=120)),
    "BodyText": dict(fonts=("Times New Roman", SERIF), sz=21,
        ppr=spacing(line=340, after=110)),
    "FirstParagraph": dict(fonts=("Times New Roman", SERIF), sz=21,
        ppr=spacing(line=340, before=60, after=110)),
    "Compact": dict(fonts=("Times New Roman", SERIF), sz=21,
        ppr=spacing(line=320, after=70) + '<w:ind w:left="360" w:hanging="180"/>'),
    "Heading1": dict(fonts=(SANS, SANS), sz=36, bold=True, color=C_TITLE,
        ppr=spacing(line=300, before=300, after=150)),
    "Heading2": dict(fonts=(SANS, SANS), sz=30, bold=True, color=C_HEAD,
        ppr=spacing(line=300, before=260, after=120)),
    "Heading3": dict(fonts=(SANS, SANS), sz=24, bold=True, color=C_HEAD,
        ppr=spacing(line=300, before=200, after=90)),
    "Heading4": dict(fonts=(SANS, SANS), sz=22, bold=True, color=C_GREY,
        ppr=spacing(line=300, before=160, after=80)),
    "Heading5": dict(fonts=(SANS, SANS), sz=21, bold=True, color=C_GREY,
        ppr=spacing(line=300, before=140, after=70)),
    "BlockText": dict(fonts=("Times New Roman", SERIF), sz=20, bg=BG_SOFT,
        ppr=spacing(line=310, before=80, after=120) + '<w:ind w:left="240" w:right="120"/>',
        border=dict(left=(18, C_HEAD))),
    "SourceCode": dict(fonts=(MONO, SERIF), sz=19, bg=BG_CODE,
        ppr=spacing(line=280, before=80, after=120) + '<w:ind w:left="180"/>'),
    "VerbatimChar": dict(fonts=(MONO, SERIF), sz=19),
    "TableNormal": dict(fonts=("Times New Roman", SERIF), sz=20,
        ppr=spacing(line=300, after=0)),
}

def new_style(sid):
    return (f'<w:style w:type="paragraph" w:styleId="{sid}">'
            f'<w:name w:val="{sid}"/><w:basedOn w:val="Normal"/>'
            f'<w:next w:val="{sid}"/><w:uiPriority w:val="99"/><w:qFormat/>'
            f'<w:pPr></w:pPr><w:rPr></w:rPr></w:style>')

def build(out_path):
    # 1) 从 pandoc 自带默认参考文档起步
    with tempfile.NamedTemporaryFile(suffix=".docx", delete=False) as tf:
        base = tf.name
    try:
        r = subprocess.run(["pandoc", "--print-default-data-file", "reference.docx"],
                           capture_output=True)
        if r.returncode != 0 or not r.stdout:
            raise SystemExit("✗ 取不到 pandoc 默认参考文档，请确认 pandoc 已安装")
        open(base, "wb").write(r.stdout)

        with zipfile.ZipFile(base) as z:
            names = z.namelist()
            doc    = z.read("word/document.xml").decode("utf-8")
            styles = z.read("word/styles.xml").decode("utf-8")
            others = {n: z.read(n) for n in names
                      if n not in ("word/document.xml", "word/styles.xml")}

        # 2) 页面尺寸改成 A5
        new_sect = (f'<w:pgSz w:w="{A5_W}" w:h="{A5_H}"/>'
                    f'<w:pgMar w:top="{MARGIN}" w:right="{MARGIN}" w:bottom="{MARGIN}"'
                    f' w:left="{MARGIN}" w:header="{HEADER_F}" w:footer="{HEADER_F}" w:gutter="0"/>')
        doc = re.sub(r'<w:pgSz[^>]*/>', '', doc)
        doc = re.sub(r'<w:pgMar[^>]*/>', '', doc)
        if "<w:sectPr" in doc:
            # 两种形态都要处理：
            #   自闭合 <w:sectPr />  → 展开成 <w:sectPr>设置</w:sectPr>
            #   成对   <w:sectPr>...</w:sectPr> → 在开标签后插入设置
            # （pandoc 默认参考文档是前一种，直接插入会把设置甩到 sectPr 外面而失效）
            m_self = re.search(r'<w:sectPr\s*/>', doc)
            if m_self:
                doc = doc[:m_self.start()] + f'<w:sectPr>{new_sect}</w:sectPr>' + doc[m_self.end():]
            else:
                doc = re.sub(r'(<w:sectPr[^>]*>)', r'\1' + new_sect, doc, count=1)
        else:
            doc = doc.replace('</w:body>', f'<w:sectPr>{new_sect}</w:sectPr></w:body>')

        # 3) 套样式
        for sid, cfg in CFG.items():
            m = re.search(r'<w:style [^>]*w:styleId="%s"[^>]*>(.*?)</w:style>' % sid, styles, re.S)
            if m:
                body = m.group(1)
            else:
                st = new_style(sid)
                body = re.search(r'<w:style [^>]*w:styleId="%s"[^>]*>(.*?)</w:style>' % sid, st, re.S).group(1)
                last = styles.rfind('</w:style>')
                styles = styles[:last+len('</w:style>')] + \
                         f'<w:style w:type="paragraph" w:styleId="{sid}">{body}</w:style>' + \
                         styles[last+len('</w:style>'):]
                m = re.search(r'<w:style [^>]*w:styleId="%s"[^>]*>(.*?)</w:style>' % sid, styles, re.S)
                body = m.group(1)
            if "ppr" in cfg: body = set_pPr(body, cfg["ppr"])
            body = set_rPr(body, fonts=cfg.get("fonts"), sz=cfg.get("sz"),
                           color=cfg.get("color"), bold=cfg.get("bold"),
                           italic=cfg.get("italic"), bg=cfg.get("bg"))
            if "border" in cfg: body = set_borders(body, **cfg["border"])
            styles = styles[:m.start()] + f'<w:style w:type="paragraph" w:styleId="{sid}">{body}</w:style>' + styles[m.end():]

        with zipfile.ZipFile(out_path, "w", zipfile.ZIP_DEFLATED) as zout:
            zout.writestr("word/document.xml", doc)
            zout.writestr("word/styles.xml", styles)
            for n, d in others.items():
                zout.writestr(n, d)
    finally:
        if os.path.exists(base): os.remove(base)
    print(f"✓ 已重建 A5 参考模板: {out_path}  ({A5_W}×{A5_H} twips = 148×210mm)")

if __name__ == "__main__":
    build(sys.argv[1] if len(sys.argv) > 1 else "a5-ref.docx")
```

## fix_tables.py

```python
# ⚠️ 存在性检查一律用前缀匹配 / 属性容错，不要写成精确字符串相等。
# 原因（实测）：pandoc 不同版本输出的 <w:tblHeader> 形态不同——
#   有的写 <w:tblHeader/>       （无属性）
#   有的写 <w:tblHeader w:val="true"/>（带属性）
# 若有人想"简化"成精确匹配，遇到另一种形态就静默失效，别改回去。

#!/usr/bin/env python3
# 修 pandoc 转出的 docx 表格：
#   1) 补 tblW(固定宽度) + tblLayout(fixed) + tblGrid(均分列宽) + tblBorders(细边框)
#      —— pandoc 不写这些，A5 窄页面下三列表格会撑爆版心，右侧内容被推出页面外 → 看着像"空白表格"
#   2) 单元格补 tcW；表头行加浅蓝底 + 加粗
#      —— pandoc 部分版本不输出 <w:tcPr>，需自行包裹属性元素，并保证 schema 顺序
# 只改 XML 属性，不碰任何文字内容（避免重打字掉字符）
import sys, zipfile, shutil, re

NS_BORDER = ("top", "left", "bottom", "right", "insideH", "insideV")
HDR_FILL = "D9E2F3"
BORDER_SZ = "4"

# 单元格级属性元素（CT_TcPr 的子元素）
CELL_ATTR = (r'(?:<w:tcBorders>.*?</w:tcBorders>'
             r'|<w:tcMar>.*?</w:tcMar>'
             r'|<w:gridSpan[^>]*/>'
             r'|<w:vMerge[^>]*/>'
             r'|<w:vAlign[^>]*/>'
             r'|<w:tcW[^>]*/>'
             r'|<w:shd[^>]*/>'
             r'|<w:textDirection[^>]*/>'
             r'|<w:noWrap[^>]*/>'
             r'|<w:cnfStyle[^>]*/>'
             r'|<w:hideMark[^>]*/>'
             r'|<w:tcFitText[^>]*/>)')

def tbl_borders_xml():
    b = "".join(f'<w:{e} w:val="single" w:sz="{BORDER_SZ}" w:space="0" w:color="000000"/>'
                for e in NS_BORDER)
    return f"<w:tblBorders>{b}</w:tblBorders>"

def build_tcpr(inner, col_w, fill=None):
    """按 CT_TcPr 的 schema 顺序组装：
    tcW → tcBorders → shd → tcMar/textDirection/tcFitText → vAlign → hideMark
    （顺序错了 Word 会判定文件损坏，所以 vAlign 必须排在 shd 之后）"""
    inner = re.sub(r'<w:tcW[^>]*/>', '', inner)
    inner = re.sub(r'<w:shd[^>]*/>', '', inner)
    va = "".join(re.findall(r'<w:vAlign[^>]*/>', inner))
    inner = re.sub(r'<w:vAlign[^>]*/>', '', inner)
    # 使用者要求"表格文字居中"：垂直方向一律居中（无显式 vAlign 时补 center）
    if not va:
        va = '<w:vAlign w:val="center"/>'
    shd = f'<w:shd w:val="clear" w:color="auto" w:fill="{fill}"/>' if fill else ""
    return (f'<w:tcPr><w:tcW w:w="{col_w}" w:type="dxa"/>'
            f'{inner}{shd}{va}</w:tcPr>')

# ⚠️ tcPr 的形态有两种，正则必须都认：
#   <w:tcPr/>  /  <w:tcPr />   （自闭合，pandoc 实际输出的是带空格的这种）
#   <w:tcPr>...</w:tcPr>       （成对）
# 跟上面 tblHeader 是同一类：**精确匹配标签字面量 vs pandoc 实际输出形态**。
TCPR_SELF = r'<w:tcPr\s*/>'
TCPR_ANY = r'<w:tcPr\s*/?>'


def fix_cells(seg, col_w, fill=None):
    """给片段内每个 <w:tc> 补/改 tcPr"""
    def with_pr(m):
        # 情形 A：已有 tcPr（自闭合或成对）—— 只替换 tcPr 本身，
        # 不要补 </w:tc>（成对形态的原闭合标签在后面，会自然保留）
        inner = m.group(1) or ''
        return '<w:tc>' + build_tcpr(inner, col_w, fill)
    # 一个正则吃掉两种形态：自闭合时 group(1) 为 None
    seg = re.sub(r'<w:tc><w:tcPr\s*(?:/>|>(.*?)</w:tcPr>)', with_pr, seg, flags=re.S)

    def no_pr(m):
        # 情形 B：没有 tcPr，属性元素裸放在 <w:tc> 后 → 收进新建的 tcPr
        return '<w:tc>' + build_tcpr(m.group(1), col_w, fill)
    # 用 + （至少 1 个）且排除已有 tcPr，否则会重复插入
    seg = re.sub(r'<w:tc>(?!%s)((?:%s)+)' % (TCPR_ANY, CELL_ATTR), no_pr, seg, flags=re.S)

    # 情形 C：连属性都没有，直接是内容 → 插一个只带 tcW 的 tcPr
    seg = re.sub(r'<w:tc>(?!%s)(?=<w:p[ >])' % TCPR_ANY,
                 lambda m: '<w:tc>' + build_tcpr('', col_w, fill),
                 seg)
    return seg

# ── 单元格文字水平居中 ──────────────────────────────
# 使用者要求"表格文字居中"。水平居中靠段落的 <w:jc w:val="center"/>。
# ⚠️ schema 顺序：CT_PPr 里 jc 必须排在 spacing / ind **之后**，且必须排在
#    rPr / sectPr / textDirection / textAlignment / outlineLvl / divId / cnfStyle **之前**。
#    插错位置 → Word 判定文件损坏（与 tblPr 那次是同一类坑）。
TAIL_ELEMS = (r'(?:<w:rPr>.*?</w:rPr>'
              r'|<w:sectPr>.*?</w:sectPr>'
              r'|<w:textDirection[^>]*/>'
              r'|<w:textAlignment[^>]*/>'
              r'|<w:outlineLvl[^>]*/>'
              r'|<w:divId[^>]*/>'
              r'|<w:cnfStyle[^>]*/>'
              r'|<w:pPrChange>.*?</w:pPrChange>)')


def center_paras(seg):
    """把片段内所有段落设为水平居中（只用于表格内部片段）"""
    def add_jc(m):
        inner = m.group(1)
        if re.search(r'<w:jc\b', inner):
            return '<w:pPr>' + re.sub(r'<w:jc[^>]*/>', '<w:jc w:val="center"/>', inner) + '</w:pPr>'
        tails = "".join(re.findall(TAIL_ELEMS, inner, flags=re.S))
        inner = re.sub(TAIL_ELEMS, '', inner, flags=re.S)
        return '<w:pPr>' + inner + '<w:jc w:val="center"/>' + tails + '</w:pPr>'

    seg = re.sub(r'<w:pPr>(.*?)</w:pPr>', add_jc, seg, flags=re.S)
    # 没有 pPr 的段落：建一个（排除自闭合 <w:p/>，空段落不加）
    seg = re.sub(r'(<w:p(?:\s[^>]*)?>)(?=<w:r[ >])',
                 r'\1<w:pPr><w:jc w:val="center"/></w:pPr>', seg)
    return seg


def bold_rpr(rm):
    inner = rm.group(1)
    if '<w:b/>' in inner:
        return rm.group(0)
    mrf = re.search(r'<w:rFonts[^>]*/>', inner)
    if mrf:
        p = mrf.end()
        return '<w:rPr>' + inner[:p] + '<w:b/>' + inner[p:] + '</w:rPr>'
    return '<w:rPr><w:b/>' + inner + '</w:rPr>'

def fix(path):
    tmp = path + ".tmp"
    with zipfile.ZipFile(path) as zin:
        names = zin.namelist()
        assert "word/document.xml" in names, "不是有效的 docx"
        doc = zin.read("word/document.xml").decode("utf-8")
        others = {n: zin.read(n) for n in names if n != "word/document.xml"}

    # 版心可用宽度（twips）= 页宽 − 左边距 − 右边距
    avail = None
    m_sz = re.search(r'<w:pgSz[^>]*w:w="(\d+)"', doc)
    if m_sz:
        w = int(m_sz.group(1))
        pm = doc[doc.rfind('<w:pgMar'):]
        pm = pm[:pm.find('/>') + 2]
        g = lambda k, d: int(re.search(r'w:%s="(\d+)"' % k, pm).group(1)) if 'w:%s=' % k in pm else d
        avail = w - g('left', 1134) - g('right', 1134)
    if not avail or avail <= 0:
        avail = 6237  # A5 148mm − 20mm×2 ≈ 108mm

    n = 0
    def fix_one(m):
        nonlocal n
        n += 1
        body = m.group(1)
        first_row = re.search(r'<w:tr[ >].*?</w:tr>', body, re.S)
        ncol = len(re.findall(r'<w:tc[ >]', first_row.group(0))) if first_row else 3
        ncol = max(ncol, 1)
        col_w = avail // ncol
        grid = "".join(f'<w:gridCol w:w="{col_w}"/>' for _ in range(ncol))

        # --- tblPr：宽度 + fixed 布局 + 边框 ---
        tpm = re.search(r'<w:tblPr>(.*?)</w:tblPr>', body, re.S)
        pr = tpm.group(1) if tpm else ""
        pr = re.sub(r'<w:tblW[^>]*/>', '', pr)
        pr = re.sub(r'<w:tblLayout[^>]*/>', '', pr)
        pr = re.sub(r'<w:tblBorders>.*?</w:tblBorders>', '', pr, flags=re.S)
        # ⚠️ CT_TblPr 的 schema 顺序：tblStyle → tblW → jc → tblInd → tblBorders
        #    → shd → tblLayout → tblCellMar → tblLook
        # 后果：LibreOffice 宽容照常渲染，Word / 在线预览器直接**整表不显示**——
        # 渲染验证看不出来，只有真打开才发现。
        style_m = re.search(r'<w:tblStyle[^>]*/>', pr)
        style = style_m.group(0) if style_m else ""
        if style_m:
            pr = pr.replace(style_m.group(0), '', 1)
        new_pr = (f'<w:tblPr>{style}<w:tblW w:w="{avail}" w:type="dxa"/>'
                  f'{tbl_borders_xml()}<w:tblLayout w:type="fixed"/>{pr}</w:tblPr>')
        if tpm:
            body = body[:tpm.start()] + new_pr + body[tpm.end():]
        else:
            body = new_pr + body

        # --- tblGrid ---
        gm = re.search(r'<w:tblGrid>.*?</w:tblGrid>', body, re.S)
        new_grid = f'<w:tblGrid>{grid}</w:tblGrid>'
        if gm:
            body = body[:gm.start()] + new_grid + body[gm.end():]
        else:
            p = body.find('</w:tblPr>') + len('</w:tblPr>')
            body = body[:p] + new_grid + body[p:]

        # --- 单元格 ---
        rows = list(re.finditer(r'<w:tr[ >].*?</w:tr>', body, re.S))
        if rows:
            a, b = rows[0].start(), rows[0].end()
            head = body[a:b]
            head = fix_cells(head, col_w, fill=HDR_FILL)
            head = center_paras(head)
            head = re.sub(r'<w:rPr>(.*?)</w:rPr>', bold_rpr, head, flags=re.S)
            # 前缀匹配，容错 <w:tblHeader/> 与 <w:tblHeader w:val="true"/>
            # （改回精确匹配会在 pandoc 输出带属性时静默失效 → 表头底色加不上）
            if not re.search(r'<w:tblHeader\b', head):
                head = (head.replace('<w:trPr>', '<w:trPr><w:tblHeader/>', 1)
                        if '<w:trPr>' in head
                        else head.replace('<w:tr>', '<w:tr><w:trPr><w:tblHeader/></w:trPr>', 1))
            body = body[:a] + head + body[b:]
            tail_start = a + len(head)
            body = body[:tail_start] + center_paras(fix_cells(body[tail_start:], col_w))
        else:
            body = center_paras(fix_cells(body, col_w))
        return "<w:tbl>" + body + "</w:tbl>"

    doc2 = re.sub(r'<w:tbl>(.*?)</w:tbl>', fix_one, doc, flags=re.S)

    with zipfile.ZipFile(tmp, "w", zipfile.ZIP_DEFLATED) as zout:
        zout.writestr("word/document.xml", doc2)
        for k, v in others.items():
            zout.writestr(k, v)
    shutil.move(tmp, path)
    print(f"  ↳ 表格已修复: {n} 个（宽 {avail/1440*25.4:.0f}mm / 均分列 / 细边框 / 表头浅蓝底加粗 / 文字居中）")

if __name__ == "__main__":
    fix(sys.argv[1])
```

## colorize.py

```python
#!/usr/bin/env python3
# 语义上色：颜色 = 功能，不是装饰。
#
# 设计原则：
#   - 只给"标记行 / 标记标题"所在的 run 上色（🟢🔵🟣⚠️🔮❓✅ 这些语义符号）
#   - 正文保持近黑，保证长读不累、整页不花
#   - 每个颜色固定对应一种功能，扫描时颜色即含义
#
# 只改 run 级属性，不碰任何文字内容。
import sys, re, zipfile, shutil

MARKERS = [
    ("🟢", "1E6B3A", "必记 · 深绿（要背的，稳）"),
    ("🔵", "2E5496", "讲解 · 中蓝（理解主力）"),
    ("🟣", "7030A0", "联系 · 紫（拓展）"),
    ("🟠", "C55A11", "前后·联系 · 橙（位置）"),
    ("⚠", "C00000", "易错/坑 · 红（危险）"),
    ("🔮", "00838F", "存疑 · 青（钩子）"),
    ("❓", "B26B00", "三问 · 橙棕（提问）"),
    ("✅", "1E6B3A", "答案 · 深绿（已确定）"),
    ("⏰", "1E6B3A", "限时 · 深绿（同必记）"),
]

# ⚠️ 匹配前必须先剥掉变体选择符 U+FE0F：⚠ 这类字符常写成带 VS16 的 ⚠️，
# 而文档里两种形态都可能出现，只认一种就会静默漏色。
# 统一做法：标记表里只写裸字符，匹配前把文本里的 U+FE0F 全部去掉。
VS16 = '\ufe0f'

# CT_RPr 里 w:color 的合法位置：rFonts/b/i 之后，sz/spacing/u 之前
_ANCHORS = (r'<w:sz[ >]', r'<w:spacing[ >]', r'<w:highlight[ >]',
            r'<w:u[ >]', r'<w:vertAlign[ >]', r'<w:em[ >]')


def add_color(rpr_inner, color):
    rpr_inner = re.sub(r'<w:color[^>]*/>', '', rpr_inner)
    tag = f'<w:color w:val="{color}"/>'
    for a in _ANCHORS:
        m = re.search(a, rpr_inner)
        if m:
            return rpr_inner[:m.start()] + tag + rpr_inner[m.start():]
    return rpr_inner + tag


def colorize_run(run_xml, color):
    m = re.match(r'(<w:r[^>]*>)(.*)(</w:r>)$', run_xml, re.S)
    if not m:
        return run_xml
    open_tag, body, close_tag = m.group(1), m.group(2), m.group(3)
    rpm = re.search(r'<w:rPr>(.*?)</w:rPr>', body, re.S)
    if rpm:
        new_pr = f'<w:rPr>{add_color(rpm.group(1), color)}</w:rPr>'
        body = body[:rpm.start()] + new_pr + body[rpm.end():]
    else:
        body = f'<w:rPr>{add_color("", color)}</w:rPr>' + body
    return open_tag + body + close_tag


# 段落级底纹：整块给底色（字色仍只给标记行，正文近黑）
SHADING = [
    ("🟢", "E8F3EC"),   # 必记块 浅绿
    ("🟠", "FBE9D6"),   # 前后·联系块 浅橙（SKILL.md v3.7 声明已落地，此前脚本里缺失）
    ("⚠", "FCE8E6"),    # 易错行 浅红
]


def add_shading(ppr_inner, fill):
    ppr_inner = re.sub(r'<w:shd[^>]*/>', '', ppr_inner)
    tag = f'<w:shd w:val="clear" w:color="auto" w:fill="{fill}"/>'
    # 【schema 顺序】CT_PPr 的固定顺序是 pBdr → shd → tabs → spacing → ind → jc → rPr。
    # 直接追加到末尾，会把 shd 排到 spacing 之后 —— Word 可能判文件损坏，
    # 而 LibreOffice 宽容、照常打开，所以这个错会静默带到交付件里。
    m = re.search(r'<w:pBdr>.*?</w:pBdr>', ppr_inner, re.S)
    if m:
        return ppr_inner[:m.end()] + tag + ppr_inner[m.end():]
    for pat in (r'<w:tabs[ >]', r'<w:spacing[ >]', r'<w:ind[ >]',
                r'<w:jc[ >]', r'<w:rPr[ >]', r'<w:snapToGrid[ >]'):
        m = re.search(pat, ppr_inner)
        if m:
            return ppr_inner[:m.start()] + tag + ppr_inner[m.start():]
    return ppr_inner + tag


def shade_paragraph(p_xml, fill):
    m = re.match(r'(<w:p(?:\s[^>]*)?>)(.*)(</w:p>)$', p_xml, re.S)
    if not m:
        return p_xml
    open_tag, body, close_tag = m.group(1), m.group(2), m.group(3)
    ppm = re.search(r'<w:pPr>(.*?)</w:pPr>', body, re.S)
    if ppm:
        new_pr = f'<w:pPr>{add_shading(ppm.group(1), fill)}</w:pPr>'
        body = body[:ppm.start()] + new_pr + body[ppm.end():]
    else:
        body = f'<w:pPr>{add_shading("", fill)}</w:pPr>' + body
    return open_tag + body + close_tag


# 段落级左边框：给"钩子"类块一个醒目的竖向标尺（底纹之外另一种整块标识）
BORDERS = [
    ("🔮", "00838F"),   # 存疑块 青色左边框（SKILL.md 标了"已落地"，此前脚本里没有）
]


def add_left_border(ppr_inner, color, sz="18", space="8"):
    ppr_inner = re.sub(r'<w:pBdr>.*?</w:pBdr>', '', ppr_inner, flags=re.S)
    tag = (f'<w:pBdr><w:left w:val="single" w:sz="{sz}" '
           f'w:space="{space}" w:color="{color}"/></w:pBdr>')
    # pBdr 必须在 shd 之前
    for pat in (r'<w:shd[ >]', r'<w:tabs[ >]', r'<w:spacing[ >]',
                r'<w:ind[ >]', r'<w:jc[ >]', r'<w:rPr[ >]'):
        m = re.search(pat, ppr_inner)
        if m:
            return ppr_inner[:m.start()] + tag + ppr_inner[m.start():]
    return ppr_inner + tag


def border_paragraph(p_xml, color):
    m = re.match(r'(<w:p(?:\s[^>]*)?>)(.*)(</w:p>)$', p_xml, re.S)
    if not m:
        return p_xml
    open_tag, body, close_tag = m.group(1), m.group(2), m.group(3)
    ppm = re.search(r'<w:pPr>(.*?)</w:pPr>', body, re.S)
    if ppm:
        new_pr = f'<w:pPr>{add_left_border(ppm.group(1), color)}</w:pPr>'
        body = body[:ppm.start()] + new_pr + body[ppm.end():]
    else:
        body = f'<w:pPr>{add_left_border("", color)}</w:pPr>' + body
    return open_tag + body + close_tag


# 块间留白：语义块（🟢🔵🟣🟠⚠🔮）之间给一点呼吸，别糊成一片
# 只加"段前"，不动"段后"——段后模板里已有（120 twips），段前原本是 0。
BLOCK_GAP_BEFORE = "100"        # twips ≈ 1.8mm（叠加原有段后 2.1mm，块间约 3.9mm）
BLOCK_GAP_MARKS = ("🟢", "🔵", "🟣", "🟠", "⚠", "🔮")


def add_gap(ppr_inner, before):
    m = re.search(r'<w:spacing([^>]*)/>', ppr_inner)
    if m:
        attrs = m.group(1)
        if re.search(r'\bw:before=', attrs):
            attrs = re.sub(r'w:before="[^"]*"', f'w:before="{before}"', attrs)
        else:
            attrs = attrs + f' w:before="{before}"'
        return ppr_inner[:m.start()] + f'<w:spacing{attrs}/>' + ppr_inner[m.end():]
    tag = f'<w:spacing w:before="{before}"/>'
    # schema：shd → spacing → ind → jc → rPr（spacing 必须排在 shd 之后）
    for pat in (r'<w:ind[ >]', r'<w:jc[ >]', r'<w:rPr[ >]'):
        mm = re.search(pat, ppr_inner)
        if mm:
            return ppr_inner[:mm.start()] + tag + ppr_inner[mm.start():]
    return ppr_inner + tag


def gap_paragraph(p_xml, before):
    m = re.match(r'(<w:p(?:\s[^>]*)?>)(.*)(</w:p>)$', p_xml, re.S)
    if not m:
        return p_xml
    open_tag, body, close_tag = m.group(1), m.group(2), m.group(3)
    ppm = re.search(r'<w:pPr>(.*?)</w:pPr>', body, re.S)
    if ppm:
        # 标题本来就有段前间距，别动
        if re.search(r'<w:pStyle[^>]*w:val="Heading', ppm.group(1)):
            return p_xml
        new_pr = f'<w:pPr>{add_gap(ppm.group(1), before)}</w:pPr>'
        body = body[:ppm.start()] + new_pr + body[ppm.end():]
    else:
        body = f'<w:pPr>{add_gap("", before)}</w:pPr>' + body
    return open_tag + body + close_tag


def process(doc):
    stats = {}

    def run_sub(m):
        run = m.group(0)
        text = "".join(re.findall(r'<w:t[^>]*>(.*?)</w:t>', run, re.S)).replace(VS16, '')
        for sym, color, desc in MARKERS:
            if sym in text:
                stats[desc] = stats.get(desc, 0) + 1
                return colorize_run(run, color)
        return run

    doc2 = re.sub(r'<w:r[^>]*>.*?</w:r>', run_sub, doc, flags=re.S)

    # 段落底纹（在上色之后，避免相互干扰）
    def para_sub(m):
        p_xml = m.group(0)
        text = "".join(re.findall(r'<w:t[^>]*>(.*?)</w:t>', p_xml, re.S)).replace(VS16, '')
        for sym, fill in SHADING:
            if sym in text:
                stats[f"底纹 · {sym}"] = stats.get(f"底纹 · {sym}", 0) + 1
                return shade_paragraph(p_xml, fill)
        return p_xml

    doc2 = re.sub(r'<w:p(?:\s[^>]*)?>.*?</w:p>', para_sub, doc2, flags=re.S)

    # 段落左边框（在底纹之后跑：pBdr 要插在 shd 之前，顺序反了会破坏 schema）
    def border_sub(m):
        p_xml = m.group(0)
        text = "".join(re.findall(r'<w:t[^>]*>(.*?)</w:t>', p_xml, re.S)).replace(VS16, '')
        for sym, color in BORDERS:
            if sym in text:
                stats[f"左边框 · {sym}"] = stats.get(f"左边框 · {sym}", 0) + 1
                return border_paragraph(p_xml, color)
        return p_xml

    doc2 = re.sub(r'<w:p(?:\s[^>]*)?>.*?</w:p>', border_sub, doc2, flags=re.S)

    # 块间留白（最后跑：spacing 要排在 shd / pBdr 之后，反过来会破坏 schema）
    def gap_sub(m):
        p_xml = m.group(0)
        text = "".join(re.findall(r'<w:t[^>]*>(.*?)</w:t>', p_xml, re.S)).replace(VS16, '')
        for sym in BLOCK_GAP_MARKS:
            if sym in text:
                stats["块间留白"] = stats.get("块间留白", 0) + 1
                return gap_paragraph(p_xml, BLOCK_GAP_BEFORE)
        return p_xml

    doc2 = re.sub(r'<w:p(?:\s[^>]*)?>.*?</w:p>', gap_sub, doc2, flags=re.S)
    return doc2, stats


def main(path):
    tmp = path + ".tmp"
    with zipfile.ZipFile(path) as zin:
        names = zin.namelist()
        doc = zin.read("word/document.xml").decode("utf-8")
        others = {n: zin.read(n) for n in names if n != "word/document.xml"}

    doc2, stats = process(doc)

    with zipfile.ZipFile(tmp, "w", zipfile.ZIP_DEFLATED) as zout:
        zout.writestr("word/document.xml", doc2)
        for k, v in others.items():
            zout.writestr(k, v)
    shutil.move(tmp, path)

    total = sum(stats.values())
    if total:
        print(f"  ↳ 语义上色: 共 {total} 处")
        for _, _, desc in MARKERS:
            if desc in stats:
                print(f"      {desc}: {stats[desc]}")
        for k in [k for k in stats if k.startswith(("底纹", "左边框", "块间留白"))]:
            print(f"      {k}: {stats[k]}")
    else:
        print("  ↳ 语义上色: 未发现标记符号（跳过）")


if __name__ == "__main__":
    main(sys.argv[1])
```

## add_pagenum.py

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
add_pagenum.py — 给 docx 注入页脚页码（居中，小字，含 PAGE/NUMPAGES 域）
用法: python3 add_pagenum.py <file.docx>
幂等：已有页脚则跳过
"""
import sys, re, shutil, zipfile, os

FOOTER_XML = '''<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<w:ftr xmlns:w="http://schemas.openxmlformats.org/wordprocessingml/2006/main">
  <w:p>
    <w:pPr>
      <w:jc w:val="center"/>
      <w:rPr><w:sz w:val="16"/><w:szCs w:val="16"/><w:color w:val="888888"/></w:rPr>
    </w:pPr>
    <w:r><w:rPr><w:sz w:val="16"/><w:color w:val="888888"/></w:rPr><w:fldChar w:fldCharType="begin"/></w:r>
    <w:r><w:rPr><w:sz w:val="16"/><w:color w:val="888888"/></w:rPr><w:instrText xml:space="preserve"> PAGE </w:instrText></w:r>
    <w:r><w:rPr><w:sz w:val="16"/><w:color w:val="888888"/></w:rPr><w:fldChar w:fldCharType="separate"/></w:r>
    <w:r><w:rPr><w:sz w:val="16"/><w:color w:val="888888"/></w:rPr><w:t>1</w:t></w:r>
    <w:r><w:rPr><w:sz w:val="16"/><w:color w:val="888888"/></w:rPr><w:fldChar w:fldCharType="end"/></w:r>
    <w:r><w:rPr><w:sz w:val="16"/><w:color w:val="888888"/></w:rPr><w:t xml:space="preserve"> / </w:t></w:r>
    <w:r><w:rPr><w:sz w:val="16"/><w:color w:val="888888"/></w:rPr><w:fldChar w:fldCharType="begin"/></w:r>
    <w:r><w:rPr><w:sz w:val="16"/><w:color w:val="888888"/></w:rPr><w:instrText xml:space="preserve"> NUMPAGES </w:instrText></w:r>
    <w:r><w:rPr><w:sz w:val="16"/><w:color w:val="888888"/></w:rPr><w:fldChar w:fldCharType="separate"/></w:r>
    <w:r><w:rPr><w:sz w:val="16"/><w:color w:val="888888"/></w:rPr><w:t>1</w:t></w:r>
    <w:r><w:rPr><w:sz w:val="16"/><w:color w:val="888888"/></w:rPr><w:fldChar w:fldCharType="end"/></w:r>
  </w:p>
</w:ftr>'''

def main():
    if len(sys.argv) < 2:
        print("用法: python3 add_pagenum.py <file.docx>"); sys.exit(1)
    path = sys.argv[1]
    tmp = path + ".tmp"
    shutil.copy2(path, tmp)
    zin = zipfile.ZipFile(tmp, 'r')
    names = zin.namelist()
    items = {n: zin.read(n) for n in names}
    zin.close()

    if any(n.startswith('word/footer') and n.endswith('.xml') for n in names):
        print("↳ 已有页脚，跳过"); os.remove(tmp); return

    footers = sorted([n for n in names if re.match(r'word/footer\d+\.xml$', n)])
    idx = len(footers) + 1
    fname = f'word/footer{idx}.xml'

    docrels = 'word/_rels/document.xml.rels'
    rels = items[docrels].decode('utf-8')
    rid = 1
    while f'Id="rId{rid}"' in rels:
        rid += 1
    newrel = f'<Relationship Id="rId{rid}" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/footer" Target="footer{idx}.xml"/>'
    rels = rels.replace('</Relationships>', newrel + '</Relationships>')
    items[docrels] = rels.encode('utf-8')
    items[fname] = FOOTER_XML.encode('utf-8')

    ct = items['[Content_Types].xml'].decode('utf-8')
    if 'footer' not in ct:
        ct = ct.replace('</Types>', '<Override PartName="/word/footer%d.xml" ContentType="application/vnd.openxmlformats-officedocument.wordprocessingml.footer+xml"/></Types>' % idx)
    items['[Content_Types].xml'] = ct.encode('utf-8')

    doc = items['word/document.xml'].decode('utf-8')
    if 'footerReference' not in doc:
        doc = doc.replace('</w:sectPr>', f'<w:footerReference w:type="default" r:id="rId{rid}"/></w:sectPr>')
        items['word/document.xml'] = doc.encode('utf-8')

    zout = zipfile.ZipFile(path, 'w', zipfile.ZIP_DEFLATED)
    for n in names:
        zout.writestr(n, items[n])
    if fname not in names:
        zout.writestr(fname, items[fname])
    zout.close()
    os.remove(tmp)
    print(f"✓ 页码已注入 ({fname})")

if __name__ == '__main__':
    main()
```

## normalize_order.py

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
normalize_order.py —— 把 docx 里 pPr / tblPr / tcPr 的子元素重排成 OOXML schema 规定的顺序。

【为什么要这个脚本】
pandoc / fix_tables / colorize 拼接 XML 时，个别子元素会落到 schema 顺序之外：
  · pandoc 把有序列表段落写成 <w:numPr> 在 <w:pStyle> 之前
    （CT_PPr 规定 pStyle 必须在 numPr 之前）—— 文科书没列表所以没暴露，
    英语书大量三问/答案/习题步骤列表一触发就是几十处。
  · fix_tables 拼 tblPr 时把 <w:jc> 留在了最末
    （CT_TblPr 规定 jc 紧跟 tblW 之后、tblBorders 之前）。
严格校验器（verify_docx.py 的闸三）据此判 FAIL，Word 打开虽通常能容忍，
但 LibreOffice/在线预览器可能整表不显示，属于"没报错≠做对了"的那类坑。

【做法】只重排这三个容器的顶层子元素，不动任何文字内容，不动 run 级属性。
CT_*Pr 的 schema 顺序是固定的，重排到规范顺序对任意合法 docx 都安全
（Word 自身保存时也会归一化）。放在流水线最后一步，一劳永逸兜住所有顺序漂移。
"""
import sys, re, zipfile, shutil
import xml.etree.ElementTree as ET

W = "http://schemas.openxmlformats.org/wordprocessingml/2006/main"
ET.register_namespace("w", W)

ORDER = {
    "pPr":   ["pStyle", "numPr", "pBdr", "shd", "tabs", "spacing", "ind",
              "jc", "rPr", "sectPr"],
    "tblPr": ["tblStyle", "tblpPr", "tblW", "jc", "tblInd", "tblBorders",
              "shd", "tblLayout", "tblCellMar", "tblLook"],
    "tcPr":  ["cnfStyle", "tcW", "gridSpan", "vMerge", "tcBorders", "shd",
              "tcMar", "textDirection", "vAlign", "hideMark"],
}


def reorder(name, inner):
    if not inner.strip():
        return inner
    wrapper = '<w:%s xmlns:w="%s">%s</w:%s>' % (name, W, inner, name)
    try:
        root = ET.fromstring(wrapper)
    except ET.ParseError:
        return inner  # 解析不了就原样返回，绝不丢内容
    kids = list(root)
    rank = {t: i for i, t in enumerate(ORDER[name])}

    def key(e):
        tag = e.tag.split("}")[-1]
        return rank.get(tag, len(ORDER[name]))  # 未知元素排最后

    kids_sorted = sorted(kids, key=key)
    for k in kids:
        root.remove(k)
    for k in kids_sorted:
        root.append(k)
    out = ET.tostring(root, encoding="unicode")
    m = re.match(r"^<w:%s[^>]*>(.*)</w:%s>$" % (name, name), out, re.S)
    return m.group(1) if m else inner


CONTAINER_RE = re.compile(r"<w:(pPr|tblPr|tcPr)>(.*?)</w:\1>", re.S)


def fix_part(xml):
    return CONTAINER_RE.sub(lambda m: "<w:%s>%s</w:%s>" % (m.group(1), reorder(m.group(1), m.group(2)), m.group(1)), xml)


def main(path):
    tmp = path + ".tmp"
    with zipfile.ZipFile(path) as zin:
        names = zin.namelist()
        parts = {n: zin.read(n) for n in names}
    changed = 0
    for n in names:
        if not n.endswith(".xml"):
            continue
        try:
            text = parts[n].decode("utf-8")
        except UnicodeDecodeError:
            continue
        if "<w:pPr>" not in text and "<w:tblPr>" not in text and "<w:tcPr>" not in text:
            continue
        new = fix_part(text)
        if new != text:
            parts[n] = new.encode("utf-8")
            changed += 1
    if changed:
        with zipfile.ZipFile(tmp, "w", zipfile.ZIP_DEFLATED) as zout:
            for n in names:
                zout.writestr(n, parts[n])
        shutil.move(tmp, path)
        print("  ↳ schema 顺序已归一化: %d 个 xml 部件" % changed)
    else:
        print("  ↳ schema 顺序本就合规，无需归一化")


if __name__ == "__main__":
    main(sys.argv[1])
```

## verify_docx.py

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
verify_docx.py —— 交付前一键验收（自检三道闸的「闸三」）

用法:
    python3 verify_docx.py 输出.docx [源.md]      # 正常验收
    python3 verify_docx.py --selftest             # 跑内置已知错误用例

------------------------------------------------------------------
【为什么要这个脚本】三次事故，共同点是「没报错 ≠ 做对了」：

  · 表格整块消失 —— XML well-formed 通过、LibreOffice 渲染通过，
    但 `tcPr` 数量是 **0**（单元格宽度一个都没生成）。没人去数。
  · 数值审计静默失效 —— 脚本打印"未发现不自洽"，实际什么都没审。
  · 底纹缺失 —— 脚本打印"语义上色 79 处"，实际只上了 🟢 一类。

这三次的共同教训：**「脚本没报错」不能当验收标准。**
能当标准的只有一条：**能抓到故意埋进去的错误。**

------------------------------------------------------------------
【测试先行】下面是**先于实现**定下的已知错误用例。
写完实现再想用例，会不自觉迁就代码，测出来的"通过"是假的。
所以本文件里 CASES 写在检测函数之前，`--selftest` 逐个喂进去确认抓得到。
"""

# ==================================================================
# 第一部分：已知错误用例（先于实现定义）
# ==================================================================
# 每条 = (用例名, 注入方式, 期望被哪一项抓到, 期望判定)
CASES = [
    ("XML 未闭合",            "inject_broken_xml",   "xml_wellformed", "FAIL"),
    ("shd 排到 spacing 之后",  "inject_shd_last",     "schema_order",   "FAIL"),
    ("tblLayout 排到 tblBorders 之前", "inject_tblpr_order", "schema_order", "FAIL"),
    ("删光 tcPr（静默失效）",  "inject_no_tcpr",      "tables",         "FAIL"),
    ("删光底纹（旧脚本症状）", "inject_no_shading",   "shading",        "WARN"),
    ("删掉 vertAlign（上下标漏转）", "inject_no_vertalign", "vertalign", "WARN"),
    ("docx 少一个字符",        "inject_char_lost",    "chars",          "FAIL"),
    ("结束标记重复（文末事故）", "inject_endblock_dup", "endblock_dup",   "FAIL"),
    ("结束标记重复·本份写法",   "inject_endblock_dup2", "endblock_dup",  "FAIL"),
    ("交付说明泄漏（文末事故）", "inject_delivery_leak", "delivery_leak",  "FAIL"),
    ("公式语法未转义（~0~）",   "inject_literal_mark",  "literal_marks",  "FAIL"),
]
# 说明：shading / vertalign 判 WARN 不判 FAIL —— 某些科目确实不该有这几类，
# 但「有 0 类」必须报出来让人看见，这正是三次事故里最阴的那种"静默"。

import re
import sys
import zipfile
import xml.etree.ElementTree as ET

# ---------- schema 顺序表（CT_PPr / CT_TblPr / CT_TcPr 的子元素顺序）----------
ORDER_PPR = ['pStyle', 'numPr', 'pBdr', 'shd', 'tabs', 'spacing', 'ind',
             'jc', 'rPr', 'sectPr']
ORDER_TBLPR = ['tblStyle', 'tblpPr', 'tblW', 'jc', 'tblInd', 'tblBorders',
               'shd', 'tblLayout', 'tblCellMar', 'tblLook']
ORDER_TCPR = ['cnfStyle', 'tcW', 'gridSpan', 'vMerge', 'tcBorders', 'shd',
              'tcMar', 'textDirection', 'vAlign', 'hideMark']

SHADING_EXPECT = {
    "E8F3EC": "必记块 🟢 浅绿",
    "FBE9D6": "前后·联系 🟠 浅橙",
    "FCE8E6": "易错 ⚠ 浅红",
}
BORDER_EXPECT = "00838F"   # 存疑块 🔮 青色左边框

PASS, FAIL, WARN, SKIP = "PASS", "FAIL", "WARN", "SKIP"


class Check:
    def __init__(self, name, status, detail=""):
        self.name, self.status, self.detail = name, status, detail

    def line(self):
        mark = {PASS: "✓", FAIL: "✗", WARN: "!", SKIP: "-"}[self.status]
        return f"  [{mark}] {self.name:<14} {self.status:<4} {self.detail}"


def _seq(inner, order):
    return [m.group(1) for m in re.finditer(r'<w:([a-zA-Z]+)[ >/]', inner)
            if m.group(1) in order]


def _ordered(inner, order):
    seq = _seq(inner, order)
    idx = [order.index(t) for t in seq]
    return idx == sorted(idx), seq


# ==================================================================
# 第二部分：检测项
# ==================================================================
def check_xml_wellformed(doc_xml):
    try:
        ET.fromstring(doc_xml)
        return Check("xml_wellformed", PASS, "well-formed")
    except ET.ParseError as e:
        return Check("xml_wellformed", FAIL, f"XML 解析失败: {e}")


def check_schema_order(doc_xml):
    bad = []
    for tag, order, label in (("pPr", ORDER_PPR, "w:pPr"),
                              ("tblPr", ORDER_TBLPR, "w:tblPr"),
                              ("tcPr", ORDER_TCPR, "w:tcPr")):
        for m in re.finditer(r'<w:%s>(.*?)</w:%s>' % (tag, tag), doc_xml, re.S):
            ok, seq = _ordered(m.group(1), order)
            if not ok:
                bad.append(f"{label}:{'→'.join(seq)}")
                break
    if bad:
        return Check("schema_order", FAIL,
                     f"{len(bad)} 类顺序违规（Word 可能判损坏或不显示）: {bad[:2]}")
    return Check("schema_order", PASS, "pPr / tblPr / tcPr 顺序合规")


def check_tables(doc_xml):
    n_tc = len(re.findall(r'<w:tc[ >]', doc_xml))
    n_tcpr = doc_xml.count('<w:tcPr>')
    n_tcw = doc_xml.count('<w:tcW')
    n_hdr = len(re.findall(r'<w:shd[^>]*w:fill="D9E2F3"', doc_xml))
    if n_tc == 0:
        return Check("tables", SKIP, "文档无表格")
    if n_tcpr == 0:
        return Check("tables", FAIL,
                     f"{n_tc} 个单元格但 tcPr = 0 —— fix_tables.py 静默失效，宽度/表头底色全没写进去")
    if n_tcpr != n_tc:
        return Check("tables", WARN, f"单元格 {n_tc} 个但 tcPr 只有 {n_tcpr} 个")
    return Check("tables", PASS, f"单元格 {n_tc} / tcPr {n_tcpr} / tcW {n_tcw} / 表头底色 {n_hdr}")


def check_shading(doc_xml):
    found = {k: doc_xml.count(k) for k in SHADING_EXPECT}
    border = len(re.findall(r'<w:pBdr>.*?w:color="%s"' % BORDER_EXPECT, doc_xml, re.S))
    missing = [v for k, v in SHADING_EXPECT.items() if found[k] == 0]
    detail = " / ".join(f"{v}×{found[k]}" for k, v in SHADING_EXPECT.items()) + f" / 左边框 {border}"
    # 三类全 0 = colorize.py 根本没生效（比"只有 🟢"更严重）
    if len(missing) == len(SHADING_EXPECT):
        return Check("shading", WARN,
                     f"三类底纹全为 0（{detail}）—— colorize.py 疑似未生效")
    # 旧版脚本的症状：只有 🟢 一类，其余全 0
    if found["E8F3EC"] > 0 and len(missing) == len(SHADING_EXPECT) - 1:
        return Check("shading", WARN,
                     f"只有 🟢 一类底纹（{detail}）—— colorize.py 疑似旧版，🟠/⚠ 没落地")
    return Check("shading", PASS, detail)


def check_vertalign(doc_xml):
    """上下标有两种形态，都要数：
    - 纯文本路线 `~x~` / `^x^` → pandoc 转成 <w:vertAlign> run
    - 原生公式路线 `$x_0$` / `$x^2$` → Word 的 <m:sSub> / <m:sSup> 对象

    只数一种 = 改用公式后误报"上下标没了"。混用两种写法时总数要相加。"""
    n = (len(re.findall(r'<w:vertAlign[^>]*/>', doc_xml))
         + len(re.findall(r'<m:sSub[ >]', doc_xml))
         + len(re.findall(r'<m:sSup[ >]', doc_xml)))
    if n == 0:
        return Check("vertalign", WARN,
                     "0 个上下标 —— 理科一章应 100+（可能整章漏转）；"
                     "语文/英语/地理等文科为 0 属正常")
    if n < 10:
        return Check("vertalign", WARN, f"只有 {n} 个上下标，个位数 = 漏了")
    return Check("vertalign", PASS, f"{n} 处")


# Unicode 上下标 → ASCII（prep_md.py 把 ₀² 转成 ~0~/^2^，pandoc 再变 Word vertAlign，
# 字符在 docx 里是 ASCII 数字/符号；md 源里若直接写 Unicode ₀²，逐字比对会误报"丢失"。
# 这里做逆归一化，让 chars 对账对上下标免疫——无论作者写 ₀ 还是 ~0~ 都能过。）
_SUPSUB_NORM = {
    '⁰': '0', '¹': '1', '²': '2', '³': '3', '⁴': '4', '⁵': '5', '⁶': '6',
    '⁷': '7', '⁸': '8', '⁹': '9', '⁺': '+', '⁻': '-', 'ⁿ': 'n', 'ⁱ': 'i',
    '₀': '0', '₁': '1', '₂': '2', '₃': '3', '₄': '4', '₅': '5', '₆': '6',
    '₇': '7', '₈': '8', '₉': '9', '₊': '+', '₋': '-', 'ₐ': 'a', 'ₑ': 'e',
    'ₒ': 'o', 'ₓ': 'x',
}


def _norm_supsub(t):
    return ''.join(_SUPSUB_NORM.get(c, c) for c in t)


def check_chars(doc_xml, md_text):
    if md_text is None:
        return Check("chars", SKIP, "未提供 md，跳过对账")
    # ⚠️ 公式内容在 <m:t>（OMML），不在 <w:t>。启用原生公式后，
    # 只取 w:t 会漏掉公式里的全部字符 —— 公式越漂亮，判"丢失"越多。
    dst = "".join(re.findall(r'<w:t(?:\s[^>]*)?>(.*?)</w:t>', doc_xml, re.S))
    dst += "".join(re.findall(r'<m:t(?:\s[^>]*)?>(.*?)</m:t>', doc_xml, re.S))
    src_chars = set(c for c in _norm_supsub(md_text) if ord(c) > 0x2000)
    lost = sorted(c for c in src_chars if c not in dst)
    if lost:
        return Check("chars", FAIL, f"丢失 {len(lost)} 种字符: {''.join(lost[:20])}")
    return Check("chars", PASS, f"{len(src_chars)} 种字符零丢失")


# ------------------------------------------------------------------
# 文末查重 + 交付说明泄漏 + 公式语法残留（第四次文末事故后加的"别靠人眼盯"闸）
# 背景：交付件尾部曾四次混入"重复的本册完成块 + 写给 AI 的验收说明
#       （请验收 / 页数由内容决定 / 有问题直接说）"，以及代码块里上下标没转义
#       导致 ~0~ 原样印在纸上。这些都是'静默污染'，肉眼容易漏，必须脚本拦。
# ------------------------------------------------------------------
_DELIVERY_LEAK = [
    "请验收", "页数：由内容决定", "有问题直接说", "改完重跑转换脚本",
]


def _doc_text(doc_xml):
    return "".join(re.findall(r'<w:t(?:\s[^>]*)?>(.*?)</w:t>', doc_xml, re.S))


def check_endblock_dup(doc_xml):
    """结束标记重复出现 = 文末块被复制了（第四次事故的根）。

    ⚠️ **两种写法都要认**（2026-09-27 修）：分册产物写「── 本册完成 ──」，
    SKILL 第十节模板写「── 本份完成 ──」。只认其中一种，另一种形态下这个闸**恒 PASS**——
    自测照样绿（自测注入的是被认的那一款），实际产物却完全没被检查。
    这是"自测通过 ≠ 有效"的典型：用例必须覆盖真实存在的全部形态。"""
    txt = _doc_text(doc_xml)
    n = txt.count("本册完成") + txt.count("本份完成")
    if n >= 2:
        return Check("endblock_dup", FAIL,
                     f"结束标记出现 {n} 次 —— 文末块被复制了，删到 1 份"
                     f"（'本册完成' / '本份完成' 两种写法都计入）")
    return Check("endblock_dup", PASS, "结束标记唯一")


def check_delivery_leak(doc_xml):
    """交付说明泄漏：写给 AI 的验收话混进了给学生看的交付件。"""
    txt = _doc_text(doc_xml)
    hit = [p for p in _DELIVERY_LEAK if p in txt]
    if hit:
        return Check("delivery_leak", FAIL,
                     f"交付说明泄漏 {len(hit)} 处（{hit}）—— 这是写给 AI 的，不是给学生看的")
    return Check("delivery_leak", PASS, "无交付说明泄漏")


def check_literal_marks(doc_xml):
    """公式上下标语法 ~x~/^x^ 没转成 Word vertAlign，会原样印在纸上。
    正常情况 pandoc 把全部 ~x~ 转掉；docx 里若还有成对的 ~…~ / ^…^，
    几乎一定是代码块里没转义（如 v~0~）。范围号 3~5V 是单个 ~，不构成对，不误报。"""
    bad = re.findall(r'[~^][^\s~^]+[~^]', _doc_text(doc_xml))
    if bad:
        return Check("literal_marks", FAIL,
                     f"公式语法没转义 {len(bad)} 处（如 {bad[:3]}）—— 会原样印 ~ 上标")
    return Check("literal_marks", PASS, "无未转义的公式上下标语法")


# 学科名白名单：跨学科栏若只剩这些光秃秃的词 = 硬凑
_SUBJECTS = {"数学", "物理", "化学", "生物", "地理", "历史", "政治", "语文", "英语",
             "计算机", "信息技术", "工程", "天文", "美术", "音乐", "体育"}

# 章节必备件（拆章后各章要自足，缺件会静默过去）
# ⚠️ "三问"用宽匹配：措辞随装订层级变（本章/本编/本册/本板块三问都有），
#    写成"本章三问"会把合订本全判成缺件（物理必修一实测误报过一次）。
_REQUIRED = ["三问", "例题", "习题", "易错", "存疑点", "站台", "复习总结"]


def check_empty_field(doc_xml, md_text=None):
    """🟠 的「同构 / 跨学科」不许填「无」，也不许留模板占位符 __。

    起因（同一个口径返工两次）：物理卷 8 处「跨学科：数学」被判硬凑 → 全改成"无"；
    地理卷又被质问"竟然是无" → 再把 10 处「同构：无」填满。
    "无"和"硬凑"是同一个病的两种表现，正确解是写出真内容，不是在两者之间来回摆。"""
    if not md_text:
        return Check("empty_field", SKIP, "未提供 md 源，跳过")
    hits = []
    for tag in ("同构", "跨学科"):
        for m in re.finditer(r'%s[:：]\s*(无|__)\s*$' % tag, md_text, re.M):
            hits.append(f"{tag}@第{md_text[:m.start()].count(chr(10)) + 1}行")
    if hits:
        return Check("empty_field", FAIL,
                     f"🟠 栏填'无'或未填 {len(hits)} 处（{hits[:5]}）—— 必须写出真内容："
                     f"同构写思维方法同构，跨学科写带公式/概念的真调用")
    return Check("empty_field", PASS, "🟠 栏均写出内容，无'无'")


def check_vague_cross(doc_xml, md_text=None):
    """跨学科栏只写光秃秃学科名（"数学"两个字）= 硬凑（与同构重复，等于没写）。"""
    if not md_text:
        return Check("vague_cross", SKIP, "未提供 md 源，跳过")
    hits = []
    for m in re.finditer(r'跨学科[:：](.+)', md_text):
        v = m.group(1).strip().rstrip("。；;").strip()
        if v in _SUBJECTS:
            hits.append(f"第{md_text[:m.start()].count(chr(10)) + 1}行「{v}」")
    if hits:
        return Check("vague_cross", FAIL,
                     f"跨学科栏只有学科名 {len(hits)} 处（{hits[:5]}）—— "
                     f"要写清被调用的公式/概念，如'物理（科里奥利力 F=2mvω·sinφ）'")
    return Check("vague_cross", PASS, "跨学科栏均带具体内容")


def check_struct_complete(doc_xml, md_text=None):
    """章节必备件清点。拆章之后各章要自足，缺件现在会静默过去。"""
    src = md_text or _doc_text(doc_xml)
    if not src:
        return Check("struct_complete", SKIP, "无源文本，跳过")
    miss = [k for k in _REQUIRED if k not in src]
    if miss:
        return Check("struct_complete", WARN,
                     f"缺必备件 {len(miss)} 项（{miss}）—— 拆章后各章要自足，确认是真不需要还是漏了")
    return Check("struct_complete", PASS, f"必备件齐全（{len(_REQUIRED)} 项）")


def check_number_sync(doc_xml, md_text=None):
    """编号三方一致：正文 `### 📍 [N.M]` / 本章知识全景 / 本章站台。

    起因（数学标准 v3）：拆分知识点后编号最容易在某一处漏改，而漏改是静默的——
    正文看着没问题，读者按编号回跳就跳空了。"""
    if not md_text:
        return Check("number_sync", SKIP, "未提供 md 源，跳过")
    lines = md_text.split("\n")

    def sect(prefix):
        try:
            a = next(i for i, l in enumerate(lines) if l.startswith(prefix))
        except StopIteration:
            return []
        b = next((i for i, l in enumerate(lines) if i > a and l.startswith("## ")),
                 len(lines))
        return lines[a:b]

    body = sorted(set(re.findall(
        r"\[(\d+\.\d+)\]",
        "\n".join(l for l in lines if l.lstrip().startswith("### 📍")))))
    if not body:
        return Check("number_sync", SKIP, "正文无编号，跳过")
    pano = sorted(set(re.findall(r"\[(\d+\.\d+)\]", "\n".join(sect("## 本章知识全景")))))
    plat = sorted(set(re.findall(r"\[(\d+\.\d+)\]", "\n".join(sect("## 🗺️")))))

    miss_pano = [n for n in body if n not in pano]
    miss_plat = [n for n in body if n not in plat]
    extra = sorted(set(pano + plat) - set(body))
    if miss_pano or miss_plat or extra:
        bits = []
        if miss_pano:
            bits.append("全景缺 %s" % miss_pano[:5])
        if miss_plat:
            bits.append("站台缺 %s" % miss_plat[:5])
        if extra:
            bits.append("越界 %s" % extra[:5])
        return Check("number_sync", FAIL,
                     "编号三方不一致（正文 %d 个）：%s —— 按编号回跳会跳空" %
                     (len(body), "；".join(bits)))
    return Check("number_sync", PASS, "编号三方一致（%d 个）" % len(body))


def check_render(path):
    import os
    import shutil
    import subprocess
    import tempfile
    soffice = shutil.which("soffice")
    if not soffice and os.path.exists(r"C:\Program Files\LibreOffice\program\soffice.exe"):
        soffice = r"C:\Program Files\LibreOffice\program\soffice.exe"
    if not soffice:
        return Check("render", SKIP, "未找到 LibreOffice，跳过")
    try:
        outdir = tempfile.mkdtemp()
        subprocess.run([soffice, "--headless", "--convert-to", "pdf",
                        "--outdir", outdir, path],
                       capture_output=True, timeout=180)
        pdfs = [f for f in os.listdir(outdir) if f.endswith(".pdf")]
        if not pdfs:
            return Check("render", FAIL, "渲染未产出 PDF")
        raw = open(os.path.join(outdir, pdfs[0]), "rb").read()
        n = len(re.findall(rb'/Type\s*/Page[^s]', raw))
        shutil.rmtree(outdir, ignore_errors=True)
        return Check("render", PASS, f"{n} 页（报告项：超预期请人工判断是真需要还是注水）")
    except Exception as e:
        return Check("render", WARN, f"渲染异常: {e}")


def verify_doc_xml(doc_xml, md_text=None, pedagogy=False):
    # 通用 docx 完整性检查（始终运行）
    checks = [
        check_xml_wellformed(doc_xml),
        check_schema_order(doc_xml),
        check_tables(doc_xml),
        check_shading(doc_xml),
        check_vertalign(doc_xml),
        check_chars(doc_xml, md_text),
        check_endblock_dup(doc_xml),
        check_delivery_leak(doc_xml),
        check_literal_marks(doc_xml),
    ]
    # 讲义约定检查（编号三方一致 / 结构完整 / 空字段 / 含糊交叉引用）
    # 属本 skill 的教学约定，依赖源 md，默认关闭；加 --pedagogy 启用。
    if pedagogy:
        checks += [
            check_empty_field(doc_xml, md_text),
            check_vague_cross(doc_xml, md_text),
            check_struct_complete(doc_xml, md_text),
            check_number_sync(doc_xml, md_text),
        ]
    return checks


# ==================================================================
# 第三部分：内置自测（把 CASES 逐个喂进去，确认抓得到）
# ==================================================================
HEAD = ('<?xml version="1.0" encoding="UTF-8" standalone="yes"?>'
        '<w:document xmlns:w="http://schemas.openxmlformats.org/wordprocessingml/2006/main">'
        '<w:body>')
TAIL = '</w:body></w:document>'

BASE = HEAD + '''
<w:p><w:pPr><w:pStyle w:val="Compact"/><w:shd w:val="clear" w:color="auto" w:fill="E8F3EC"/>
<w:spacing w:after="120"/></w:pPr><w:r><w:rPr><w:color w:val="1E6B3A"/></w:rPr>
<w:t>必记 H</w:t></w:r><w:r><w:rPr><w:vertAlign w:val="subscript"/></w:rPr><w:t>2</w:t></w:r>
<w:r><w:t>O 与 2×10</w:t></w:r><w:r><w:rPr><w:vertAlign w:val="superscript"/></w:rPr>
<w:t>-6</w:t></w:r>SUP_EXTRA</w:p>
<w:p><w:pPr><w:shd w:val="clear" w:color="auto" w:fill="FBE9D6"/><w:spacing w:after="120"/></w:pPr>
<w:r><w:t>前后联系</w:t></w:r></w:p>
<w:p><w:pPr><w:pBdr><w:left w:val="single" w:sz="18" w:space="8" w:color="00838F"/></w:pBdr>
<w:spacing w:after="120"/></w:pPr><w:r><w:t>存疑点</w:t></w:r></w:p>
<w:tbl><w:tblPr><w:tblStyle w:val="Table"/><w:tblW w:w="7031" w:type="dxa"/>
<w:tblBorders><w:top w:val="single" w:sz="4" w:space="0" w:color="000000"/></w:tblBorders>
<w:tblLayout w:type="fixed"/></w:tblPr>
<w:tblGrid><w:gridCol w:w="3515"/><w:gridCol w:w="3515"/></w:tblGrid>
<w:tr><w:tc><w:tcPr><w:tcW w:w="3515" w:type="dxa"/>
<w:shd w:val="clear" w:color="auto" w:fill="D9E2F3"/></w:tcPr>
<w:p><w:r><w:t>列一</w:t></w:r></w:p></w:tc>
<w:tc><w:tcPr><w:tcW w:w="3515" w:type="dxa"/>
<w:shd w:val="clear" w:color="auto" w:fill="D9E2F3"/></w:tcPr>
<w:p><w:r><w:t>列二</w:t></w:r></w:p></w:tc></w:tr>
<w:tr><w:tc><w:tcPr><w:tcW w:w="3515" w:type="dxa"/></w:tcPr>
<w:p><w:r><w:t>甲</w:t></w:r></w:p></w:tc>
<w:tc><w:tcPr><w:tcW w:w="3515" w:type="dxa"/></w:tcPr>
<w:p><w:r><w:t>乙</w:t></w:r></w:p></w:tc></w:tr></w:tbl>
''' + TAIL

# 基准样本要凑够 10 个以上上下标，否则 vertalign 项在"未埋错"时就先报 WARN，
# 自测会分不清哪些 WARN 是样本太小、哪些是注入出来的。
SUP_EXTRA = "".join(
    '<w:r><w:rPr><w:vertAlign w:val="superscript"/></w:rPr><w:t>%d</w:t></w:r>' % i
    for i in range(10))
BASE = BASE.replace("SUP_EXTRA", SUP_EXTRA)

BASE_MD = "必记 H~2~O 与 2×10^-6^ 前后联系 存疑点 列一 列二 甲 乙"

# md 类闸（empty_field / vague_cross / struct_complete）看的是源文本，
# 所以要单独备一份"合格 md"作基准，再往里注入错误——不能污染 BASE_MD，
# 否则 check_chars 会拿它对 docx 比字符，直接误报丢字。
GOOD_MD = "\n".join([
    "## ❓ 本章三问", "### ❓ 本板块三问", "## 📌 例题", "## ✏️ 习题",
    "## ⚠️ 易错点深讲", "## 🔮 存疑点", "## 🗺️ 本章站台", "## 📋 复习总结",
    "同构：与简谐运动同构",
    "跨学科：物理（科里奥利力 F = 2mvω·sin φ）",
])

# (标签, 注入后的 md, 期望闸名, 期望状态)
MD_SYNC_OK = """## 本章知识全景
- [1.1] 甲
- [1.2] 乙

### 📍 [1.1] 甲
正文内容

### 📍 [1.2] 乙
正文内容

## 🗺️ 本章站台
- [1.1] 甲
- [1.2] 乙
"""

MD_SYNC_BAD = """## 本章知识全景
- [1.1] 甲
- [1.2] 乙

### 📍 [1.1] 甲
正文内容

### 📍 [1.2] 乙
正文内容

## 🗺️ 本章站台
- [1.1] 甲
"""

MD_CASES = [
    ("同构栏填无", "同构：无", "empty_field", FAIL),
    ("跨学科栏填无", "跨学科：无", "empty_field", FAIL),
    ("模板占位未填", "同构：__", "empty_field", FAIL),
    ("跨学科只写学科名", "跨学科：数学", "vague_cross", FAIL),
    ("跨学科写清内容不算硬凑", GOOD_MD, "vague_cross", PASS),
    ("缺必备件（删站台）", GOOD_MD.replace("本章站台", ""), "struct_complete", WARN),
    ("必备件齐全", GOOD_MD, "struct_complete", PASS),
    ("编号三方一致", MD_SYNC_OK, "number_sync", PASS),
    ("站台漏一个编号", MD_SYNC_BAD, "number_sync", FAIL),
]


def _inject(name, xml):
    if name == "inject_broken_xml":
        return xml.replace("</w:body>", "</w:oops></w:body>")
    if name == "inject_shd_last":
        # 把 shd 挪到 spacing 之后（违反 CT_PPr 顺序）
        return re.sub(r'(<w:shd[^>]*/>)(<w:spacing[^>]*/>)', r'\2\1', xml, count=1)
    if name == "inject_tblpr_order":
        # tblLayout 挪到 tblBorders 之前
        # （必须容中间的换行/缩进：pandoc 输出的 XML 是带换行的，
        #   不容空白的话注入根本不生效，自测会假通过）
        return re.sub(r'(<w:tblBorders>.*?</w:tblBorders>)(\s*)(<w:tblLayout[^>]*/?>)',
                      r'\3\2\1', xml, count=1, flags=re.S)
    if name == "inject_no_tcpr":
        return re.sub(r'<w:tcPr>.*?</w:tcPr>', '', xml, flags=re.S)
    if name == "inject_no_shading":
        return re.sub(r'<w:shd[^>]*/>', '', xml)
    if name == "inject_no_vertalign":
        return re.sub(r'<w:vertAlign[^>]*/>', '', xml)
    if name == "inject_char_lost":
        return xml.replace("<w:t>甲</w:t>", "<w:t></w:t>", 1)
    if name == "inject_endblock_dup":
        return xml.replace("<w:p><w:r><w:t>列一</w:t></w:r></w:p>",
                           "<w:p><w:r><w:t>列一</w:t></w:r></w:p>"
                           "<w:p><w:r><w:t>── 本册完成 ──</w:t></w:r></w:p>"
                           "<w:p><w:r><w:t>── 本册完成 ──</w:t></w:r></w:p>", 1)
    if name == "inject_endblock_dup2":
        return xml.replace("<w:p><w:r><w:t>列一</w:t></w:r></w:p>",
                           "<w:p><w:r><w:t>列一</w:t></w:r></w:p>"
                           "<w:p><w:r><w:t>── 本份完成 ──</w:t></w:r></w:p>"
                           "<w:p><w:r><w:t>── 本份完成 ──</w:t></w:r></w:p>", 1)
    if name == "inject_delivery_leak":
        return xml.replace("<w:p><w:r><w:t>列一</w:t></w:r></w:p>",
                           "<w:p><w:r><w:t>列一</w:t></w:r></w:p>"
                           "<w:p><w:r><w:t>请验收。</w:t></w:r></w:p>", 1)
    if name == "inject_omml":
        return xml.replace("<w:p><w:r><w:t>列一</w:t></w:r></w:p>",
                           "<w:p><w:r><w:t>列一</w:t></w:r></w:p>"
                           "<m:oMath><m:r><m:t>初</m:t></m:r></m:oMath>", 1)
    if name == "inject_literal_mark":
        return xml.replace("<w:t>列一</w:t>", "<w:t>列一 v~0~</w:t>", 1)
    raise ValueError(name)


def selftest():
    print("=== 内置自测：把已知错误逐个喂进去，确认抓得到 ===")
    print("（判据：不是'脚本没崩'，是'错误被报出来'）\n")
    base = verify_doc_xml(BASE, BASE_MD, pedagogy=True)
    base_map = {c.name: c.status for c in base}
    print("基准样本（未埋错）应全部 PASS / SKIP：")
    for c in base:
        print("   " + c.line())

    print("\n注入已知错误后：")
    all_ok = True
    for label, injector, expect_name, expect_status in CASES:
        bad = _inject(injector, BASE)
        res = {c.name: c for c in verify_doc_xml(bad, BASE_MD, pedagogy=True)}
        got = res[expect_name].status
        # 期望 FAIL 的项，实际是 FAIL 即通过；期望 WARN 同理
        hit = (got == expect_status) or (expect_status == "WARN" and got == "FAIL")
        all_ok &= hit
        print(f"  [{'✓' if hit else '✗'}] {label:<28} → {expect_name} = {got}（期望 {expect_status}）")
        if not hit:
            print(f"        ✗ 没抓到！这是元层面的失效：脚本看起来在跑，实际抓不住。")

    # ---- 公式形态（OMML）----
    # 不走 CASES 框架：chars 是 md ↔ docx 两边对账，必须同时给两边，
    # 只改 docx 那一侧测不出"公式里的中文被判丢失"这个 bug。
    print("\n公式形态（OMML）：文本层纳入 <m:t>、上下标计入 <m:sSub>/<m:sSup>：")
    # md 侧只追加"公式里确实存在的那几个中文"（初 / 落体）。
    # 多写一个汉字而 docx 里没有 → chars 报丢失，那是假失败，测不出真问题。
    omml_md = BASE_MD + "初落体"
    omml_xml = BASE.replace(
        "<w:p><w:r><w:t>列一</w:t></w:r></w:p>",
        "<w:p><w:r><w:t>列一</w:t></w:r></w:p>"
        "<m:oMath>"
        "<m:sSup><m:e><m:r><m:t>v</m:t></m:r></m:e>"
        "<m:sup><m:r><m:t>2</m:t></m:r></m:sup></m:sSup>"
        "<m:r><m:t>初</m:t></m:r>"
        "<m:r><m:t>落体</m:t></m:r>"
        "</m:oMath>", 1)
    res = {c.name: c for c in verify_doc_xml(omml_xml, omml_md, pedagogy=True)}
    for nm, want in (("chars", "PASS"), ("vertalign", "PASS")):
        got = res[nm].status
        # vertalign 只要求"不是 0/个位数"，WARN 也算数到了；chars 必须 PASS
        hit = got == want or (nm == "vertalign" and got == "WARN")
        all_ok &= hit
        print(f"  [{'✓' if hit else '✗'}] {nm:<11} = {got}（期望 {want}）—— {res[nm].detail}")
        if not hit:
            print("        ✗ 公式形态没被正确识别 —— 启用公式后这里会误判")

    print("\nmd 类闸（看源文本，单独注入，含 3 条'不该误报'的反向用例）：")
    for label, md, expect_name, expect_status in MD_CASES:
        res = {c.name: c for c in verify_doc_xml(BASE, md, pedagogy=True)}
        got = res[expect_name].status
        hit = (got == expect_status) or (expect_status == "WARN" and got == "FAIL")
        all_ok &= hit
        print(f"  [{'✓' if hit else '✗'}] {label:<26} → {expect_name} = {got}（期望 {expect_status}）")
        if not hit:
            print("        ✗ 没抓到 / 误报！这是元层面的失效：脚本看起来在跑，实际判定不准。")

    print("\n" + ("✓ 自测通过：所有已知错误都能被抓到" if all_ok
                  else "✗ 自测失败：有错误没被抓到，脚本不可信"))
    return 0 if all_ok else 1


# ==================================================================
# 第四部分：入口
# ==================================================================
def main():
    if "--selftest" in sys.argv:
        sys.exit(selftest())

    pedagogy = "--pedagogy" in sys.argv
    args = [a for a in sys.argv[1:] if not a.startswith("--")]

    if len(args) < 1:
        print(__doc__)
        print("用法: python3 verify_docx.py 输出.docx [源.md] [--pedagogy]")
        print("      python3 verify_docx.py --selftest")
        sys.exit(2)

    path = args[0]
    md_text = None
    if len(args) > 1:
        md_text = open(args[1], encoding="utf-8").read()

    with zipfile.ZipFile(path) as z:
        doc_xml = z.read("word/document.xml").decode("utf-8")

    print(f"=== 交付前验收（闸三）: {path} ===")
    if not pedagogy:
        print("  （讲义约定检查已跳过；加 --pedagogy 启用 编号/结构/空字段/交叉引用 检查）")
    checks = verify_doc_xml(doc_xml, md_text, pedagogy=pedagogy)
    checks.append(check_render(path))
    for c in checks:
        print(c.line())

    n_fail = sum(1 for c in checks if c.status == FAIL)
    n_warn = sum(1 for c in checks if c.status == WARN)
    print(f"\n{'✗ 有 FAIL，不要交付' if n_fail else '✓ 无 FAIL，可交付'}"
          f"（FAIL {n_fail} / WARN {n_warn}）")
    if n_warn:
        print("  WARN 需人工确认：是真的不需要，还是静默失效。")
    sys.exit(1 if n_fail else 0)


if __name__ == "__main__":
    main()
```
