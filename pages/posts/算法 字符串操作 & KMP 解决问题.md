- Summary
  1. JS 字符串不可变原则
  2. 区间反转通用模板
  3. 刷题 （abcde → 后面 k 个到前面去 → dcabc）三次反转
  4. 刷题 （是否是重复子串） → 暴力 1→ n // 2 → KMP

## 一、JS 里字符串相关的基本点

1. **字符串不可变**
   - `s[0] = 'x'` 根本改不动字符串。

   - 想"原地修改"：必须

     ```ts
     const arr = s.split('')
     // ...在 arr 上用下标改
     return arr.join('')
     ```

2. **区间反转的通用模板**

   ```jsx
   function reverse(arr, l, r) {
     while (l < r) {
       [arr[l], arr[r]] = [arr[r], arr[l]]
       l++
       r--
     }
   }

   ```

之后所有题（541、右旋、KMP 版构造）都在反复用这个思路：**把字符串当数组 + 区间反转 / 下标对比**。

---

## 二、541. 反转字符串 II 的思路

题意：每 **2k** 个字符一组，只反转其中前 **k** 个。

核心思路：

1. 先 `arr = s.split('')`

2. 写 `reverse(l, r)` 在 `arr` 上交换。

3. 主循环按 **步长 2k** 跑：

   ```ts
   for (let i = 0; i < n; i += 2 * k) {
     const left = i
     const right = Math.min(i + k - 1, n - 1)
     reverse(arr, left, right)
   }
   return arr.join('')
   ```

要点：

- 步长是 `2 * k`，不是 `k`。
- 右边界用 `Math.min` 防越界，自动 cover “少于 k”/“介于 k 和 2k” 的情况。

---

## 三、右旋字符串的“三次翻转”套路

右旋 k 位：`"abcdefg"`，k=2 → `"fgabcde"`

套路：

1. 整体反转：`"gfedcba"`
2. 反转前 k 段：`"fgedcba"`
3. 反转后面那段：`"fgabcde"`

代码骨架：

```jsx
function rotateRight(s, k) {
  const arr = s.split('')
  const n = arr.length
  k = k % n

  const reverse = (l, r) => {
    while (l < r) {
      [arr[l], arr[r]] = [arr[r], arr[l]]
      l++
      r--
    }
  }

  reverse(0, n - 1)
  reverse(0, k - 1)
  reverse(k, n - 1)

  return arr.join('')
}

```

这跟“541 的区间反转模板”是同一套路，只是换了用法。

---

## 四、459. 重复的子字符串 —— 两种做法

### 1. 暴力枚举子串长度 L

逻辑：

1. `n = s.length`
2. 枚举 `L` 从 1 到 `n-1`
   - 要求 `n % L === 0`（能整除才有可能完整重复）
   - 取 `pattern = s.slice(0, L)`
   - 重复 `n / L` 次，看拼出来的是否等于 `s`

```jsx
function repeatedSubstringPattern(s) {
  const n = s.length
  for (let L = 1; L < n; L++) {
    if (n % L !== 0)
      continue
    const pattern = s.slice(0, L)
    const repeatCount = n / L
    let str = ''
    for (let i = 0; i < repeatCount; i++) {
      str += pattern
    }
    if (str === s)
      return true
  }
  return false
}
```

本质：**枚举所有候选 L，穷举验证**。

---

### 2. KMP / LPS 版本的判定

### ① 先搞清楚前缀 / 后缀 / LPS

- 前缀（真前缀）：从头开始的一段，不包含整个串本身

  `"abab"` 的真前缀：`"a"`, `"ab"`, `"aba"`

- 后缀（真后缀）：从尾开始的一段，不包含整个串本身

  `"abab"` 的真后缀：`"b"`, `"ab"`, `"bab"`

- “既是前缀又是后缀”的叫 **前后缀**，取最长那个，它的长度就是 LPS 某一位的值。

对串 `s`：

- `lps[i]` = 在前缀 `s[0..i]` 里，**最长的前后缀长度**。

例：

- `"abab"`：整串的最长前后缀是 `"ab"`，所以最后一位 `lps[3] = 2`
- `"aabaa"`：整串的最长前后缀是 `"aa"`，所以 `lps[4] = 2`

---

### ② 暴力构造 LPS

暴力建表：

```jsx
function buildLpsBrutal(s) {
  const n = s.length
  const lps = Array.from({ length: n }).fill(0)

  for (let i = 0; i < n; i++) {
    for (let L = i; L >= 1; L--) {
      const prefix = s.slice(0, L)
      const suffix = s.slice(i - L + 1, i + 1)
      if (prefix === suffix) {
        lps[i] = L
        break
      }
    }
  }
  return lps
}

```

意思就是：

**对每个前缀 `s[0..i]` 暴力试所有长度 L，找一个“头 L == 尾 L”的最大 L。**

KMP 优化的只是在“怎么找到这个 L”，不是换问题本身。

---

### ③ KMP 构造 LPS 的高效写法

核心循环：

```jsx
const lps = Array.from({ length: n }).fill(0)
let len = 0 // 当前"已知前后缀长度"
let i = 1

while (i < n) {
  if (s[i] === s[len]) {
    len++
    lps[i] = len
    i++
  }
  else {
    if (len > 0) {
      len = lps[len - 1] // 关键回退
    }
    else {
      lps[i] = 0
      i++
    }
  }
}

```

**本质：**

- **给每个 i 算一个 lps[i]**
- 暴力写法：`L--` 一格一格试
- KMP 写法：用 `len = lps[len - 1]` 一步跳到“更短但有希望的 L”，避免从 L-1, L-2... 再试一轮

总结：

> 因为新前缀只多了一个字符，它的 LPS 最多是旧 LPS + 1，
>
> 如果这个 +1 方案挂了，就退回“旧前缀自己的 LPS”，
>
> 不用再从所有 L 一路往下遍历。

---

### ④ 用 LPS 判定“是否是重复子串”

记：

- `n = s.length`
- `len = lps[n - 1]`（**整串的最长前后缀长度**）
- 定义 `period = n - len`

结论：

```text
if (len > 0 && n % period === 0) ⇒ s 是重复子串
否则 ⇒ 不是
```

**为什么这两个条件够了？**

1. `len > 0`

   ⇒ 前缀 `s[0..len-1]` = 后缀 `s[n-len..n-1]`

   ⇒ 写成下标：对所有 `i ∈ [n-len, n-1]`，`s[i] = s[i - (n-len)]`

   ⇒ 记 `p = n - len`，得到：**对所有 `i >= p`，`s[i] = s[i - p]`**

   ⇒ 这就是“**有周期 p**”：尾部那块每隔 p 个字符重复一次。

2. `n % p === 0`

   ⇒ 串长正好是 p 的整数倍，可以切成若干段，每段 p 长

   ⇒ 再用上面“周期 p”的关系，能证明**每一段都等于第一段**

   ⇒ 所以串 = 第一段重复 k 次 ⇒ 就是重复子串。

反例 `ab12ab` 很典型：

- 最长前后缀 `"ab"`，len = 2
- `n = 6` ⇒ `p = 6 - 2 = 4`
- 周期关系确实存在（尾巴那段和前面对齐），但 `6 % 4 ≠ 0` ⇒ 切块会剩余半段 ⇒ **不是“重复子串”**。

完整代码

```jsx
function repeatedSubstringPattern(s) {
  const n = s.length
  if (n <= 1)
    return false

  // 1. 构造 lps 数组：lps[i] = s[0..i] 的最长前后缀长度
  const lps = Array.from({ length: n }).fill(0)
  let len = 0 // 当前"已匹配的前后缀长度"
  let i = 1 // 从第二个字符开始

  while (i < n) {
    if (s[i] === s[len]) {
      // 匹配：前后缀长度 +1
      len++
      lps[i] = len
      i++
    }
    else {
      // 失配：尝试用更短的前后缀
      if (len > 0) {
        len = lps[len - 1]
      }
      else {
        lps[i] = 0
        i++
      }
    }
  }

  // 2. 用最后一个 lps 值判断是否为重复子串
  const longestPrefixSuffix = lps[n - 1] // 整串的最长前后缀
  if (longestPrefixSuffix === 0)
    return false // 根本没有前后缀

  const period = n - longestPrefixSuffix // 推出的"最小循环节长度"
  return n % period === 0 // 能整除 → 可以由该子串重复构成
}
```

---

## 五、几个“套路”

1. **遇到“按规则翻转字符串/数组”**
   - 先 `split` / 数组化
   - 写一个通用 `reverse(l, r)`
   - 再思考：下标怎么跑（`i += ?`），每轮操作哪一段
2. **遇到“旋转”类操作**
   - 优先想“**三次翻转**”这个模板（左旋、右旋都能套）
3. **遇到“是否由某个子串重复构成”这类题**
   - 暴力：枚举 L，`n % L == 0`，`pattern.repeat(n/L) === s`
   - 进阶：用 LPS / KMP：
     - 构造 lps 数组
     - `len = lps[n-1]`
     - `period = n - len`
     - 判 `len > 0 && n % period === 0`
4. **对 LPS / KMP 的心智模型**
   - `lps[i]`：前缀 `s[0..i]` 的最长前后缀长度。
   - 构造 lps 就是在**对每个 i，找到最大的 L 使得“头 L == 尾 L”**，只不过 KMP 用 lps 递推，把暴力的 `L--` 优化掉。
