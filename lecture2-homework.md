# 第2课 课后作业
### 转换为bit位 Num2Bits 

- 参数：`nBits`
- 输入信号：`in`
- 输出信号：`b[nBits]`

输出信号应该是长度为`nBits`的位数组，相当于`in`的二进制表示。 `b[0]` 是最低有效位。
```Solidity
template Num2Bits(n) {
    signal input in;
    signal output out[n];

    var lc1=0;
    var e2=1;
    for (var i = 0; i<n; i++) {
        out[i] <-- (in \ 2**i) % 2; //先计算出out再做电路约束检查
        out[i] * (out[i] -1 ) === 0; //检查不是0就是1，确保二进制
        lc1 += out[i] * e2; //还原in再做检查
        e2 = e2+e2;
    }
    lc1 === in; 
}

component main {public [in]}= Num2Bits(3);

---------------------------------------------
input.json
{
  "in": 3
}
```

报错记录：

```Solidity
Error: Scalar size does not match
    at _multiExp (/Users/oker/project/zk/circom-starter-master-test/node_modules/ffjavascript/build/main.cjs:4975:19)
    at WasmCurve.multiExpAffine (/Users/oker/project/zk/circom-starter-master-test/node_modules/ffjavascript/build/main.cjs:5012:22)
    at Object.groth16Prove [as prove] (/Users/oker/project/zk/circom-starter-master-test/node_modules/snarkjs/build/main.cjs:825:33)
    at groth16 (/Users/oker/project/zk/circom-starter-master-test/node_modules/hardhat-circom/dist/index.js:384:38)
    at SimpleTaskDefinition.circomCompile [as action] (/Users/oker/project/zk/circom-starter-master-test/node_modules/hardhat-circom/dist/index.js:483:22)
    at Environment._runTaskDefinition (/Users/oker/project/zk/circom-starter-master-test/node_modules/hardhat/src/internal/core/runtime-environment.ts:308:14)
    at Environment.run (/Users/oker/project/zk/circom-starter-master-test/node_modules/hardhat/src/internal/core/runtime-environment.ts:156:14)
    at main (/Users/oker/project/zk/circom-starter-master-test/node_modules/hardhat/src/internal/cli/cli.ts:272:7)
error Command failed with exit code 1.
info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.
```

https://github.com/iden3/snarkjs/issues/116

zkRepl 在线工具可以运行，本地中同样判零电路也只有一个输入，没有报错。

### 判零 IsZero 

- 参数：无
- 输入信号：`in`
- 输出信号：`out`

要求：如果`in`为零，`out`应为`1`。 如果`in`不为零，`out`应为`0`。 这个有点棘手！

```Solidity
template IsZero() {
  signal input in;
  signal output out;

  signal inv;
  inv <-- in !=0 ? 1/in : 0; //利用提示值
  out <== -in * inv + 1; //in为0，out为1
  in * out === 0; //验证输入输出符合约束
}

component main { public [ in ] } = IsZero();

---------------------------------------------
input.json
{
  "in": 0
}
```

### 相等 IsEqual 

- 参数：无
- 输入信号：`in[2]`
- 输出信号：`out`

要求：如果 `in[0]` 等于 `in[1]`，则 `out` 应为 `1`。 否则，`out` 应该是 `0`。

```Solidity
include "IsZero.circom"; 
// IsZero.circom加上main直接引入会编译报错，多个main组件。Multiple main components in the project structure

template IsEqual() {
  signal input in[2];
  signal output out;

  component isz = IsZero();
  in[1] - in[0] ==> isz.in; // 两个数的差作为IsZero的输入判断是否为0
  isz.out ==> out;
}

component main = IsEqual();

---------------------------------------------
input.json
{
  "in": ["0", "1"]
}
```

### 少于 LessThan 

- 参数：无
- 输入信号：`in[2]`。 假设提前知道这些最多 2252−1。
- 输出信号：`out`

要求：如果 `in[0]` 严格小于 `in[1]`，则 `out` 应为 `1`。 否则，`out` 应该是 `0`。

- 扩展 1：如果您知道输入信号最多为 2k−1(k≤252)，您如何减少该电路所需的约束总数？ 编写一个在`k`中参数化的电路版本。
- 扩展 2：编写 LessEqThan（测试 in[0] 是否 ≤ in[1]）、GreaterThan 和 GreaterEqThan

```Solidity
template LessThan(k) {
    assert(k <= 252); //限制输入的有效范围，减少约束的总数
    signal input in[2];
    signal output out;

    component n2b = Num2Bits(k+1);

    n2b.in <== in[0]+ (1<<k) - in[1];//两个数字相减，并加上最大值2**k

    out <== 1-n2b.out[k];
}

component main = LessThan(250);

---------------------------------------------
input.json
{
  "in": ["1", "2"]
}
```

### 选择器 Selector 

- 参数：`nChoices`
- 输入信号：`in[nChoices]`, `index`
- 输出：`out`

要求：输出`out`应该等于`in[index]`。 如果 `index` 越界（不在 [0, nChoices) 中），`out` 应该是 `0`。

```Solidity
include "IsEqual.circom"; 
include "LessThan.circom"; 

//求和
template CalculateTotal(n) {
    signal input in[n];
    signal output out;

    signal sums[n];

    sums[0] <== in[0];

    for (var i = 1; i < n; i++) {
        sums[i] <== sums[i-1] + in[i];
    }

    out <== sums[n-1];
}

template QuinSelector(choices) {
    signal input in[choices];
    signal input index;
    signal output out;
    
    // 确保 index < choices
    component lessThan = LessThan(4);
    lessThan.in[0] <== index;
    lessThan.in[1] <== choices;
    lessThan.out === 1;
    //index 越界时，这里的约束不是就无法生成了吗？实践后确实报错：Error: Error: Assert Failed. Error in template QuinSelector_5 line: 31

    component calcTotal = CalculateTotal(choices);
    component eqs[choices];

    // 遍历in[choices]，通过遍历时的i值和输入的index值判断是否相等
    for (var i = 0; i < choices; i ++) {
        eqs[i] = IsEqual();
        eqs[i].in[0] <== i;
        eqs[i].in[1] <== index;

        //不相等时，eqs out为0，否则为1。1*in[i]则为index下的值
        calcTotal.in[i] <== eqs[i].out * in[i];
    }

    // 正常返回数组中index下的值，越界时，求和结果为0
    out <== calcTotal.out;
}

component main { public [ in, index ] } = QuinSelector(2);
```

### 判负 IsNegative （没看懂）

注意：信号是模 p（Babyjubjub 素数）的残基，并且没有`负`数模 p 的自然概念。 但是，很明显，当我们将`p-1`视为`-1`时，模运算类似于整数运算。 所以我们定义一个约定：`取负` 按照惯例认为是 (p/2, p-1] 中的余数，非负是 [0, p/2) 中的任意数

- 参数：无
- 输入信号：`in`
- 输出信号：`out`

要求：如果根据我们的约定，`in` 为负数，则 `out` 应为 `1`。 否则，`out` 应该是 `0`。 您可以自由使用[CompConstant circuit](https://github.com/iden3/circomlib/blob/master/circuits/compconstant.circom)，它有一个常量参数`ct`，如果`in`（二进制数组）在解释为整数时严格大于 `ct` 则输出 `1` ，否则为 `0`。

[解决方案](https://github.com/iden3/circomlib/blob/master/circuits/sign.circom#L23)

- 理解检查：为什么我们不能只使用 LessThan 或上一个练习中的比较器电路之一？

```Solidity
template Sign() {
    signal input in[254];
    signal output sign;

    component comp = CompConstant(10944121435919637611123202872628637544274182200208017171849102093287904247808);

    var i;

    for (i=0; i<254; i++) {
        comp.in[i] <== in[i];
    }

    sign <== comp.out;
}
```

### 整数除法 IntegerDivide 

注意：这个电路非常难！

- 参数：`nbits`。 使用 `assert` 断言这最多为 126！
- 输入信号：`dividend`, `divisor` （被除数，除数）
- 输出信号：`remainder`, `quotient` （余数，商）

要求：首先，检查`dividend`和`divisor`是否最多为`nbits`位长。 接下来，计算并约束`余数`和`商`。

- 扩展：您将如何修改电路以处理负的被除数？

[解决方案](https://github.com/darkforest-eth/circuits/blob/master/perlin/perlin.circom#L44)（忽略第二个参数SQRT_P，这是无关紧要的）

### 排序 Sort 【可选】 

- 参数：`N`
- 输入信号：`in[N]`
- 输出信号：`out[N]`

要求：将输入`in[N]`的`N`个数字按照从小到大进行排列，并输出到`out[N]`信号中。

```Solidity

```
