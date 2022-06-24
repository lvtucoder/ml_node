# FM、FFM和AFM三种算法的对比

## 三种算法的对比

|                      | FM                                                           | FFM                                                          | AFM                                                          |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **交叉项**           | <img src="/Users/webuy/Library/Application Support/typora-user-images/image-20220614150023840.png" alt="image-20220614150023840" style="zoom:25%;" /> | <img src="/Users/webuy/Library/Application Support/typora-user-images/image-20220614150056568.png" alt="image-20220614150056568" style="zoom:25%;" /> | <img src="/Users/webuy/Library/Application Support/typora-user-images/image-20220614150109707.png" alt="image-20220614150109707" style="zoom:25%;" /> |
| **待学习的参数个数** | 1. LR部分：<br />     1 + n<br />2. Embedding部分：<br />     n * k | 1. LR部分：<br />     1 + n<<br />2. Embedding部分: <br />     n * f * k | 1. LR部分：<br />     1 + n<<br />2. Embedding部分: <br />     n * f * k<br />3. Attention Network部分参数:<br />     k * t + t * 2<br />4. MLP 部分参数：<br />     K * 1 |
| **优点**             | 使用隐向量相乘模拟特征交叉，适用于稀疏场景                   | 提出field的概念，细化隐向量的表示                            | 通过attention network学习不同特征交互的重要性，性能更好，可解释性强 |
| **缺点**             | 一个特征只对应一个向量                                       | 慢                                                           | 未考虑高阶组合特征                                           |

